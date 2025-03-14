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

### Implement Grafana | Prometheus (wip.)
* Create a new directory to store all files and navigate there:
```bash
mkdir -p ~/k3s/grafana-prometheus && \
cd ~/k3s/grafana-prometheus
```
* Create a new namespace:
```bash
kubectl create namespace monitoring
```
* Create a secret for the login credentials:
```bash
kubectl create secret generic grafana-secret \
--namespace=monitoring \
--from-literal=admin-user=dontUseAdmin \
--from-literal=admin-password='SuperSecurePassword123!'
```
* Create a self-signed certificate if needed:
```bash
openssl req -x509 -nodes -newkey rsa:4096 -keyout domain.key -out domain.crt -days 365
```
* Create a TLS secret for HTTPS:
```bash
kubectl create secret tls grafana-tls \
--namespace monitoring \
--cert=domain.crt \
--key=domain.key
```
* Install and apply the Flannel CNI:
```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```
* Restart k3s:
```bash
sudo systemctl restart k3s
```
* Verify that flannel is running:
```bash
kubectl get pods -n kube-flannel
```
* Create a ConfigMap `prometheus-config.yaml` for Prometheus Configuration:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 10s  

    scrape_configs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace]
            regex: ".*"  
            action: keep
```
* Create `prometheus-deployment.yaml` for Secure Prometheus Deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  serviceAccountName: prometheus
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      securityContext:
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
      containers:
        - name: prometheus
          image: prom/prometheus:v2.37.0
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus/"
            - "--web.enable-lifecycle"
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: config-volume
              mountPath: /etc/prometheus
            - name: storage-volume
              mountPath: /prometheus
          securityContext:
            readOnlyRootFilesystem: true
            runAsNonRoot: true
      volumes:
        - name: config-volume
          configMap:
            name: prometheus-config
        - name: storage-volume
          emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  selector:
    app: grafana
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
```
* Deploy Prometheus:
```bash
kubectl apply -f prometheus-config.yaml -f prometheus-deployment.yaml
```
* Create `grafana-deployment.yaml` for Secure Grafana Deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:latest
          env:
            - name: GF_SECURITY_ADMIN_USER
              valueFrom:
                secretKeyRef:
                  name: grafana-secret
                  key: admin-user
            - name: GF_SECURITY_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: grafana-secret
                  key: admin-password
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  selector:
    app: grafana
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
```
* Deploy Grafana:
```bash
kubectl apply -f grafana-deployment.yaml
```
* Create an Ingress `grafana-ingress.yaml` to forward Grafana with HTTPS:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - grafana.web.ctf.htl-villach.at
      secretName: grafana-tls
  rules:
    - host: grafana.web.ctf.htl-villach.at
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grafana
                port:
                  number: 80
```
* Deploy the ingress:
```bash
kubectl apply -f grafana-ingress.yaml
```
* Create a ConfigMap `alertmanager-config.yaml` for the Alert Manager:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    route:
      group_by: ['alertname']
      receiver: 'slack-notifications'
    receivers:
      - name: 'slack-notifications'
        slack_configs:
          - channel: '#alerts'
            send_resolved: true
            api_url: 'https://hooks.slack.com/services/XXXXX/XXXXX/XXXXX'
```
* Create `alertmanager-deployment.yaml` to deploy the Alert Manager:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alertmanager
  template:
    metadata:
      labels:
        app: alertmanager
    spec:
      containers:
        - name: alertmanager
          image: prom/alertmanager:v0.24.0
          args:
            - "--config.file=/etc/alertmanager/alertmanager.yml"
          ports:
            - containerPort: 9093
          volumeMounts:
            - name: config-volume
              mountPath: /etc/alertmanager
      volumes:
        - name: config-volume
          configMap:
            name: alertmanager-config
```
* Deploy the Alert Manager:
```bash
kubectl apply -f alertmanager-config.yaml -f alertmanager-deployment.yaml
```
* Verify that the pods are active and running:
```bash
kubectl get pods -n monitoring
```

---

### Implement K3s Dashboard
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
chmod 644 /etc/rancher/k3s/certs/ca.crt && \
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