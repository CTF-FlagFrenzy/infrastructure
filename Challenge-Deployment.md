# How to 'infrastructure' | Deploying Challenges

---

> [!IMPORTANT]
> Before getting started, consider working through the guides `README.md`, `Docker-Registry-Setup.md` and `K3s-Cluster-Setup.md` in order to set up everything needed in advance.

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

### Deploy a challenge 
* Make sure the needed image is on the registry by accessing the web registry's interface `https://registry:8443`.
* Create a deployment script `deployment.yml` for the kubernetes deployment as well as service:
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-${TEAMID}-${CHALLENGE}
  labels:
    app: ${CHALLENGE}
    team: '${TEAMID}'
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ${CHALLENGE}
      team: '${TEAMID}'
  template:
    metadata:
      labels:
        app: ${CHALLENGE}
        team: '${TEAMID}'
    spec:
      containers:
      - name: ${CHALLENGE}
        image: registry:5000/${CHALLENGE}
        ports:
          - containerPort: 80
        env:
        - name: TEAMKEY
          valueFrom:
            secretKeyRef:
              name: teamkey-${TEAMID}
              key: TEAMKEY
      imagePullSecrets:
      - name: registry-credentials
---
apiVersion: v1
kind: Service
metadata:
  name: service-${TEAMID}-${CHALLENGE}
spec:
  selector:
    app: ${CHALLENGE}
    team: '${TEAMID}'
  ports:
  - protocol: TCP
    port: 80
```
* Create a ingress script `ingress.yml` in order to forward / expose the service:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-${TEAMID}-${CHALLENGE}
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: "${SUBDOMAIN}.web.ctf.htl-villach.at"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service-${TEAMID}-${CHALLENGE}
            port:
              number: 80
```
* Create a bash script `script.sh` that automates the process:
```bash
#!/bin/bash

IN_TEAMID=$1
IN_CHALLENGE=$2

export TEAMID=$IN_TEAMID
export CHALLENGE=$IN_CHALLENGE
export SUBDOMAIN=`echo 'Attention: Team '$TEAMID' wants to steal the recipe for '$CHALLENGE'-cookies' | sha256sum | cut -f1 -d" "`

printf 'Deploying challenge "'$CHALLENGE'" for team '$TEAMID'...\n\n'
envsubst < deployment.yml | kubectl apply -f -
envsubst < ingress.yml | kubectl apply -f -

printf '\nDone!\n'
printf '\nExposing challenge at...\n'
echo $SUBDOMAIN'.web.ctf.htl-villach.at'
```
* Set permissions for the script:
```bash
chmod 775 script.sh
```
* Deploy the challenge:
> [!NOTE]
> Change the values to deploy the correct service for the correct team. Further, the challenge name for the script has to match the registry entry. Consider to use lower case letters seperated by '-'.
```bash
./script.sh <TEAMID> <CHALLENGE>
```
* On the load balancer, edit the `nginx.conf` file by adding this:
> [!NOTE]
> Change the port to the right one for the internal traefik load balancer. Find the port when executing `kubectl get svc -n kube-system`. Further, consider changing the second level domain `web` to the right one.
```conf
# Load balancer for K3s
http {
    upstream traefik_nodes {
        least_conn; # Distribute traffic based on least connections
        server 172.23.0.56:32430; # K3s master node 1
        server 172.23.0.57:32430; # K3s master node 2
    }


    server {
        listen 80;
        server_name ~^(?<subdomain>.+)\.web.ctf.htl-villach.at$;
        return 301 https://$host$request_uri; # Redirect HTTP to HTTPS
    }


    server {
        listen 443 ssl;
        server_name ~^(?<subdomain>.+)\.web.ctf.htl-villach.at$;

        ssl_certificate /etc/nginx/certs/domain.crt;  # Path to your TLS certificate
        ssl_certificate_key /etc/nginx/certs/domain.key; # Path to your private key

        location / {
            proxy_pass http://traefik_nodes; # Forward to Traefik using HTTP
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
> Before redeploying, consider exposing the nginx' port 80 as well as 443 in order to forward requests comming in on the host's port 80/443 to the container. Further, add the certificates for HTTPS, to enable TLS.  
> Add the following lines to the `docker-compose.yml` file in the ports section on within the nginx service:
> ```yaml
> --- other configurations ---
>
> ports:
>   - 6443:6443
>   - 80:80 # HTTP ( --- add this line --- )
>   - 443:443 # HTTPS ( --- add this line --- )
> volumes:
>   - ./nginx.conf:/etc/nginx/nginx.conf:ro # Mount Nginx configuration file as read-only
>   - ./certs:/etc/nginx/certs:ro # Mount certificates directory as read-only ( --- add this line --- )
>
> --- other configurations ---
> ```
```bash
docker compose up --build -d nginx
```

---

### Set up a DNS-Server for forwarding subdomains (only locally)
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

### Troubleshooting
> [!TIP]
> When shutting down the cluster VMs, consider starting them in the correct order to prevent conflicts. (load balancer --> masters --> workers). Ensure this step before troubleshooting.
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