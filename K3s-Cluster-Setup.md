# How to 'infrastructure' | Setting up a K3s Cluster

---

> [!IMPORTANT]
> Before getting started, consider working through the guides `README.md` and `Docker-Registry-Setup.md` in order to set up everything needed in advance.

---

### Set up the load balancer
* Add the load balancer's dns name (lb):
> [!NOTE]
> Change `ip` to the load balancer's ip address.
```bash
sudo -- sh -c "echo 'ip ct.ctf.htl-villach.at' >> /etc/hosts"
```
* Create a directory to store the configuration as well as the portainer data and navigate into it:
```bash
mkdir -p ~/loadbalancer/{db,portainer,certs} && \
cd ~/loadbalancer
```
> [!NOTE]
> Replace `hostname` with the hostname of the machine the container stack is running on (`ct` is the global DNS - delete or change it if there is another DNS then the hostname)
```bash
openssl req -x509 -nodes -newkey rsa:2048 -keyout certs/domain.key -out certs/domain.crt -days 365 -addext "subjectAltName = DNS.1:hostname,DNS.2:ct.ctf.htl-villach.at"
```
* Create and open the `docker-compose.yml` file and define the nginx as well as the portainer:
```yaml
version: '3' # Specify Docker Compose version


networks:
  my-loadbalancer-net:
    driver: bridge # Custom network to facilitate inter-service communication


volumes:
  # Bind host directory to store Portainer data
  my-database-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: ./db

  # Bind host directory to store Portainer data
  my-portainer-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: ./portainer


services:


  nginx:
    image: nginx:latest
    restart: always # Always restart if the container stops
    ports:
      - 6443:6443 # Expose Nginx on port 8443 (HTTPS) for K3s

    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro # Mount Nginx configuration file as read-only

    networks:
      - my-loadbalancer-net # Attach Nginx service to the custom network


  db:
    image: mysql:latest

    volumes:
      - my-database-data:/var/lib/mysql

    env_file: .env
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE_PORT: ${MYSQL_DATABASE_PORT}

    networks:
      - my-loadbalancer-net

    healthcheck:
       test: 'echo "select now(); " | mysql -h ${MYSQL_DATABASE_HOST} -u${MYSQL_USER} -p${MYSQL_PASSWORD}'
       interval: 1s
       retries: 10
       start_period: 10s
       timeout: 1s

    ports:
    - "3306:3306"


  portainer:
    image: portainer/portainer-ce
    ports:
      - 9443:9443 # Expose Portainer on port 9443 (HTTPS)

    restart: on-failure # Restart only on failure

    volumes:
      - my-portainer-data:/data # Mount Portainer data volume
      - $XDG_RUNTIME_DIR/docker.sock:/var/run/docker.sock # Rootless Docker socket mount for Portainer

    networks:
      - my-loadbalancer-net # Attach Portainer to the custom network
```
* Create and open the `nginx.conf` file and define the load balancer rules including the forwarding:
> [!NOTE]
> Change the ip addresses under `k3s_servers` if needed - the port should be the same though.
```nginxconf
events {}

stream {
  upstream k3s_servers {
    server 172.23.0.56:6443;
    server 172.23.0.57:6443;
  }

  server {
    listen 6443;
    proxy_pass k3s_servers;
  }
}
```
* Create and open the `.env` file and define the variables for the database:
```sh
MYSQL_DATABASE=k3s
MYSQL_USER=k3s
MYSQL_PASSWORD=MySuperSecurePassw0rd
MYSQL_ROOT_PASSWORD=MySuperSecureR00tPassw0rd
MYSQL_DATABASE_PORT=3306
MYSQL_DATABASE_HOST=ct.ctf.htl-villach.at
K3S_TOKEN=MySuperSecureT0ken
```
* Deploy the container stack with its services:
```bash
docker compose up --build -d
```

---

### Set up the K3s servers
* Add the load balancer's dns name to the server:
> [!NOTE]
> Change `ip` to the load balancer's ip address.
```bash
sudo -- sh -c "echo 'ip ct.ctf.htl-villach.at' >> /etc/hosts"
```
* Create a directory for all important k3s data:
```bash
mkdir ~/k3s && \
cd ~/k3s
```
* Copy the `.env` file from the load balancer to the k3s server and activate it:
```bash
source .env
```
* Export the variable for the installation process and verify its contents
```bash
export K3S_DATASTORE_ENDPOINT='mysql://'$MYSQL_USER':'$MYSQL_PASSWORD'@tcp('$MYSQL_DATABASE_HOST':'$MYSQL_DATABASE_PORT')/'$MYSQL_DATABASE && \
echo $K3S_DATASTORE_ENDPOINT
```
* Install k3s:
> [!WARNING]
> When changes to the databes or the token are made, consider redeploying the whole database instance, including resetting the persistent bind volume on the load balancer. Further, uninstall the k3s instance on the servers, so that every bit of data is reset - if not it might lead to data inconsistency.
```bash
curl -sfL https://get.k3s.io | sh -s - server \
--token=$K3S_TOKEN \
--node-taint CriticalAddonsOnly=true:NoExecute \
--tls-san ct.ctf.htl-villach.at
```
* Verify the installation, display all pods:
```bash
sudo k3s kubectl get nodes
```
* Repeat the process for the other servers
> [!NOTE]
> If you want to print the token use `sudo cat /var/lib/rancher/k3s/server/node-token` on any off the servers.
* Use this command to make further work with `k3s` rootless:
```bash
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```

---

### Set up the K3s agents
* Add the load balancer's dns name to the server:
> [!NOTE]
> Change `ip` to the load balancer's ip address.
```bash
sudo -- sh -c "echo 'ip ct.ctf.htl-villach.at' >> /etc/hosts"
```
* Create a directory for all important k3s data:
```bash
mkdir ~/k3s && \
cd ~/k3s
```
* Copy the `.env` file from the load balancer to the k3s agent and activate it:
```bash
source .env
```
* Export the variable for the installation process and verify its contents
```bash
export K3S_DATASTORE_ENDPOINT='mysql://'$MYSQL_USER':'$MYSQL_PASSWORD'@tcp('$MYSQL_DATABASE_HOST':'$MYSQL_DATABASE_PORT')/'$MYSQL_DATABASE && \
echo $K3S_DATASTORE_ENDPOINT
```
* Install k3s:
```bash
curl -sfL https://get.k3s.io | \
K3S_URL=https://ct.ctf.htl-villach.at:6443 \
K3S_TOKEN=$K3S_TOKEN \
sh -
```
* Verify the installation, display all pods on one of the servers:
```bash
k3s kubectl get nodes
```

---

### Install kubectl on Windows for testing
* Install the package and verify:
```ps
winget install -e --id Kubernetes.kubectl; `
kubectl version --client
```
* Create the folder `.kube` and navigate to it:
```ps
mkdir .kube; `
cd .kube
```
* Create a new config file:
```ps
New-Item config -type file
```
* Open it with a prefered editor and paste in the contents of the server's `/etc/rancher/k3s/k3s.yaml` file.
> [!NOTE]
> Consider changing the ip under `server:` to the load balancer's hostname.
* To verify, check for the available nodes in the cluster:
```ps
kubectl get nodes
```

---

### Implement a dashboard
* Pull the latest dashboard release:
```bash
k3s kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.4/aio/deploy/recommended.yaml
```
* Add an admin user for the dashboard (`dashboard.admin-user.yml`):
```yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```
* Assign the admin role (`dashboard.admin-user-role.yml`):
```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```
* Deploy the admin-user configuration:
> [!NOTE]
> If you are doing this from your dev machine (windows set up earlier), remove k3s and just use kubectl.
```bash
k3s kubectl create -f dashboard.admin-user.yml -f dashboard.admin-user-role.yml
```
* Get the bear token for the login:
```bash
k3s kubectl -n kubernetes-dashboard create token admin-user
```
* On the dev machine, enable the proxy:
```ps
kubectl proxy
```
* On the dashboard (`http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`) log in using the token gathered earlier.
* Verify the functionality by viewing the nodes.

---

### Implement private registry v1 (recommended)
* Add the registry's dns name to the server:
> [!NOTE]
> Change `ip` to the registry's ip address.
```bash
sudo -- sh -c "echo 'ip registry' >> /etc/hosts"
```
* Create a secret for the registry's credentials and general information:
> [!NOTE]
> Change `myuser` and `mypassword` to the correct values.
```bash
kubectl create secret docker-registry registry-credentials \
  --docker-server=https://registry:5000 \
  --docker-username=myuser \
  --docker-password=mypassword \
```
* Verify the secrets:
```bash
kubectl describe secret registry-credentials
```

---

### Implement private registry v2 (not recommended)
> [!WARNING]
> This might cause some errors.
* Add the registry's dns name to the server:
> [!NOTE]
> Change `ip` to the registry's ip address.
```bash
sudo -- sh -c "echo 'ip registry' >> /etc/hosts"
```
* Copy the needed files from the registry to the server:
> [!NOTE]
> Change `myuser` to the right user on the registry server.
```bash
sudo scp plonerf@registry:~/docker-registry/certs/domain* /etc/rancher/k3s/
```
* Create `/etc/rancher/k3s/.env` for the registry's login credentials:
> [!NOTE]
> Change the values if needed.
```sh
REGISTRY_USER=RegistryUser
REGISRTY_PASSWORD=MySuperSecurePassw0rd
```
* Update K3s configuration in `/etc/rancher/k3s/registries.yaml` to integrate the private registry:
```yml
mirrors:
  "registry:5000":

    endpoint:
      - "https://registry:5000"


configs:
  "registry:5000":

    env_file: .env
    environment:
      REGISTRY_USER: ${REGISTRY_USER}
      REGISRTY_PASSWORD: ${REGISRTY_PASSWORD}

    auth:
      username: ${REGISTRY_USER}
      password: ${REGISRTY_PASSWORD}

    tls:
      cert_file: "domain.crt"
      key_file: "domain.key"
```
* Repeat the process on all servers.