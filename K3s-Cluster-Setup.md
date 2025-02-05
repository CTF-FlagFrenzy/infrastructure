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
> [!WARNING]
> When using podman-compose instead of docker, consider changing the portainer mout for the socket.
> ```yaml
>     volumes:
>       - my-portainer-data:/data # Mount Portainer data volume
>       - $XDG_RUNTIME_DIR/podman/podman.sock:/var/run/docker.sock # --- change this ---
> ```
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
> [!WARNING]
> When using podman-compose, consider binding the session so that the pods don't go down after the session is closed. Afterwards, deploy the pods using `podman-compose up --build -d`
> ```bash
> loginctl enable-linger
> ```
```bash
docker compose up --build -d
```
> [!TIP]
> If the database is not reachable, consider allowing traffic trhough the firewall.
> ```bash
> sudo firewall-cmd --add-port=3306/tcp --permanent
> ```

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
* If it does not work permanently consider changing the ownership:
```bash
sudo chown manager: /etc/rancher/k3s/k3s.yaml
```
* Further, consider changing the server attribute in the `/etc/rancher/k3s/k3s.yaml` from localhost to the server ip:
> [!NOTE]
> Change the ip address to the k3s master's. Do this on both masters. This is for the fastapi service in order for it to communicate with the cluster out of the container.
```yaml
- cluster:
    certificate-authority-data: 
    server: https://172.23.0.57:6443
  name: default
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
* On the host create a ssh tunnel to access the manager:
```bash
ssh -i .\.ssh\id_rsa_FlagFrenzy -N -L localhost:8001:localhost:8001 manager@172.23.0.56
```
* On the dashboard (`http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`) log in using the token gathered earlier.
* Verify the functionality by viewing the nodes.

---

### Implement private registry on the masters
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

### Implement private registry on the agents
> [!NOTE]
> All commands have been executed as root.
* Create needed folders:
```bash
mkdir -p /etc/rancher/k3s/certs/ && \
mkdir -p /var/lib/rancher/k3s/agent/etc/containerd/certs.d/web.ctf.htl-villach.at:5000/
```
* Get the certificate:
```bash
scp manager@172.23.0.55:~/docker-registry/certs/domain.crt /var/lib/rancher/k3s/agent/etc/containerd/certs.d/web.ctf.htl-villach.at:5000/
```
* Copy the certificate the persist it:
```bash
cp /var/lib/rancher/k3s/agent/etc/containerd/certs.d/web.ctf.htl-villach.at:5000/domain.crt /etc/rancher/k3s/certs/ca.crt
```
* Change some permissions:
```bash
chmod 644 /etc/rancher/k3s/certs/ca.crt
chown root:root /etc/rancher/k3s/certs/ca.crt
```
* Edit the `/etc/rancher/k3s/registries.yaml`:
```yml
mirrors:
  "web.htl-villach.at:5000":

    endpoint:
      - "https://web.htl-villach.at:5000"


configs:
  "web.ctf.htl-villach.at:5000":

    auth:
      username: "cookie_pusher"
      password: "MyR€gistryUser123!"

    tls:
      ca_file: "/etc/rancher/k3s/certs/ca.crt"
```
* Restart the k3s-agent service:
```bash
systemctl restart k3s-agent
```
* Try to pull an image:
```bash
crictl pull web.ctf.htl-villach.at:5000/solana-assets
```