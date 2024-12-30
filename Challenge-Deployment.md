# How to 'infrastructure' | Deploying Challenges

---

> [!IMPORTANT]
> Before getting started, consider working through the guides `README.md`, `Docker-Registry-Setup.md` and `K3s-Cluster-Setup.md` in order to set up everything needed in advance.

> [!CAUTION]
> The values provided are mostly test values - This is **NOT** the final, dynamic version. Further, no TLS is implemented yet. This will be integrated soon though.

---

### Load / Create secrets for team-keys
> [!TIP]
> If the secrets are loaded when creating a team during an automated process, this step can be skipped.
* Create a secret with the teamkey for dynamic challenge deployment:
```bash
kubectl create secret generic teamkey-1 --from-literal=TEAMKEY="mysupersecureTEAMKEYforteam1"
```
* Verify that the secret is created successfully:
```bash
kubectl describe secret teamkey-1
```

---

### Deploy the service 
> [!CAUTION]
> ***Not dynamic yet!!!*** The challanges run with static names so far in order to make development easier.
* Make sure the needed image is on the registry by accessing the web registry's interface `https://registry:8443`.
* Create a deployment script `deployment.yml` for the kubernetes deployment as well as service:
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: challenge-1-deployment
  labels:
    app: challenge1
    team: team1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: challenge1
      team: '1'
  template:
    metadata:
      labels:
        app: challenge1
        team: '1'
    spec:
      containers:
      - name: challenge1
        image: registry:5000/ceasar_cypher
        ports:
          - containerPort: 80
        env:
        - name: TEAMKEY
          valueFrom:
            secretKeyRef:
              name: key-team1
              key: TEAMKEY
      imagePullSecrets:
      - name: registry-credentials
---
apiVersion: v1
kind: Service
metadata:
  name: challenge-1-service
spec:
  selector:
    app: challenge1
    team: '1'
  ports:
  - protocol: TCP
    port: 80
```
* Deploy the challenge:
```bash
kubectl apply -f deployment.yml
```
* Check if the pod(s) is/are deployed:
```bash
kubectl get pods
```
* Check if the service is running:
```bash
kubectl get svc
```
* ***Optional:*** To verify the service is running, a self-destroying test container can be created for testing (afterwards, the tests needed can be performed):
```bash
kubectl run test-pod --rm -it --image=busybox:1.28 --restart=Never -- sh
```

---

### Set up a DNS-Server for forwarding subdomains
* Install dnsmasq:
```bash
sudo apt-get install dnsmasq
```
* Edit `/etc/dnsmasq.conf` in order to set the dns entries:
> [!NOTE]
> Change the ip address as well as the name if needed
```bash
address=/web/192.168.80.33
address=/.web/192.168.80.33
```
* Restart the service:
> [!WARNING]
> When encountering the issue `dnsmasq[585141]: failed to create listening socket for port 53: Address already in use` on restart, consider these three steps in order to resolve the problem:
> * Disable the system's resolved service:
> ```bash
> sudo systemctl disable --now systemd-resolved
> ```
> * Delete the `resolv.conf` file:
> ```bash
> sudo rm -f /etc/resolv.conf
> ```
> * Add localhost as nameserver to replace the configuration with dnsmasq
> ```bash
> echo "nameserver 127.0.0.1" | sudo tee /etc/resolv.conf
> ```
```bash
sudo systemctl restart dnsmasq
```
* Verify the installation by checking the DNS resolution:
```bash
dig web @127.0.0.1
```
```bash
dig mychallenge.web @127.0.0.1
```

---

### Set up Ingress to Forward Services
> [!CAUTION]
> ***Not dynamic yet!!!*** The ingress scripts run with static names so far in order to make development easier.
* On the load balancer, edit the `nginx.conf` file by adding this:
> [!NOTE]
> Change the port to the right one for the internal traefik load balancer. Find the port when executing `kubectl get svc -n kube-system`.
```conf
# Load balancer for K3s
http {
    upstream traefik_nodes {
        least_conn; # Distribute traffic based on least connections
        server 192.168.80.31:32216; # K3s master node 1
        server 192.168.80.32:32216; # K3s master node 2
    }

    server {
        listen 80;
        server_name ~^(?<subdomain>.+)\.web$;

        location / {
            proxy_pass http://traefik_nodes;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```
* Redeploy the nginx service:
> [!WARNING]
> Before redeploying, consider exposing the nginx' port 80 in order to forward requests comming in on the host's port 80 to the container. Add the following line to the `docker-compose.yml` file in the ports section on within the nginx service:
> ```yaml
> --- other configurations ---
>
> ports:
>   - 6443:6443
>   - 80:80 # add this line
>
> --- other configurations ---
> ```
```bash
docker compose up --build -d nginx
```
* Create a ingress script `ingress.yml` in order to forward / expose the service:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: challenge-1-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: "challenge-1.web"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: challenge-1-service
            port:
              number: 80
```

---

### Troubleshooting
**Verify the ingress resource:**
* The output should confirm:
  * The host matches the expected subdomain (e.g., challenge-1.web).
  * The backend service (challenge-1-service) and port (80) are correct.
```bash
kubectl describe ingress challenge-1-ingress
```
**Check the service:**
* Ensure the service:
  * Has a valid ClusterIP.
  * Exposes port 80 (or the correct target port).
```bash
kubectl describe svc challenge-1-service
```
**Verify the Pods and Endpoints:**
* The ENDPOINTS column should list the pod IPs and ports.
* If there are no endpoints, ensure the pods are running and match the service's selector.
```bash
kubectl get endpoints challenge-1-service
```
**Confirm Traefik Service NodePort:**
* Confirm that:
  * The service type is LoadBalancer.
  * The right port is mapped to Traefik's HTTP service.
```bash
kubectl get svc -n kube-system
```
**Inspect Traefik Deployment:**
* The service should be running.
```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik
```
* If not, restart the deployment for traefik:
```bash
kubectl rollout restart deployment traefik -n kube-system
```