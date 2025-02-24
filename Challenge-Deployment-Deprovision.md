# How to 'infrastructure' | Deploying / Deprovisioning Challenges

---

> [!IMPORTANT]
> Before getting started, consider working through the guides `README.md`, `Docker-Registry-Setup.md` and `K3s-Cluster-Setup.md` in order to set up everything needed in advance.

---

### Set up API Endpoints
* On the master nodes, create a directory for the deployment service and navigate there:
```bash
mkdir -p ~/fastapi-deployment-api/app
cd ~/fastapi-deployment-api/app
```
* Create an `main.py` file:
```py
from fastapi import FastAPI, HTTPException, Header
from pydantic import BaseModel
import subprocess
import os

app = FastAPI()

# Environment variable for API key
API_KEY = os.getenv("API_KEY", "default_secure_key")

# Define the request body model
class ChallengeRequest(BaseModel):
    teamid: str
    challenge: str

class SecretRequest(BaseModel):
    teamid: str
    teamkey: str

@app.post("/deploy")
async def deploy_service(request: ChallengeRequest, authorization: str = Header(None)):
    # Check authorization header
    if authorization != f"Bearer {API_KEY}":
        raise HTTPException(status_code=401, detail="Unauthorized")

    try:
        # Extract values from the request body
        teamid = request.teamid
        challenge = request.challenge

        # Construct and run deployment script
        command = f"/bin/bash ./deploy.sh {teamid} {challenge}"
        result = subprocess.run(command, shell=True, capture_output=True, text=True)

        if result.returncode == 0:
            output = result.stdout.strip()
            # Extract service URL from script output
            service_url = [line for line in output.splitlines() if line.startswith("https")][0]
            return {"message": "Deployment successful", "url": service_url}
        else:
            raise HTTPException(status_code=500, detail=result.stderr)

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


@app.post("/deprovision")
async def deprovision_service(request: ChallengeRequest, authorization: str = Header(None)):
    # Check authorization header
    if authorization != f"Bearer {API_KEY}":
        raise HTTPException(status_code=401, detail="Unauthorized")

    try:
        # Extract values from the request body
        teamid = request.teamid
        challenge = request.challenge

        # Construct and run deprovision script
        command = f"/bin/bash ./deprovision.sh {teamid} {challenge}"
        result = subprocess.run(command, shell=True, capture_output=True, text=True)

        if result.returncode == 0:
            output = result.stdout.strip()
            return {"message": "Deprovision successful", "details": {"team": teamid, "challenge": challenge}}
        else:
            raise HTTPException(status_code=500, detail=result.stderr)

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/teamkey")
async def create_teamkey_service(request: SecretRequest, authorization: str = Header(None)):
    # Check authorization header
    if authorization != f"Bearer {API_KEY}":
        raise HTTPException(status_code=401, detail="Unauthorized")

    try:
        # Extract values from the request body
        teamid = request.teamid
        teamkey = request.teamkey

        # Construct and run teamkey script
        command = f"/bin/bash ./teamkey.sh {teamid} {teamkey}"
        result = subprocess.run(command, shell=True, capture_output=True, text=True)

        if result.returncode == 0:
            output = result.stdout.strip()
            details = [line for line in output.splitlines() if line.startswith("TEAMKEY")][0]
            return {"message": "Secret Creation successful", "details": details}
        else:
            raise HTTPException(status_code=500, detail=result.stderr)

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```
* Create a deployment script `deployment.yml` for the kubernetes deployment as well as service:
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-${TEAMID}-${CHALLENGE}
  namespace: namespace-team-${TEAMID}
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
        resources:
          requests:
            memory: "256Mi" # sample value
            cpu: "250m" # sample value
          limits:
            memory: "512Mi" # sample value
            cpu: "500m" # sample value
      imagePullSecrets:
      - name: registry-credentials
---
apiVersion: v1
kind: Service
metadata:
  name: service-${TEAMID}-${CHALLENGE}
  namespace: namespace-team-${TEAMID}
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
  namespace: namespace-team-${TEAMID}
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
* Create a bash script `deploy.sh` that automates the process:
```bash
#!/bin/bash

IN_TEAMID=$1
IN_CHALLENGE=$2

export TEAMID=$IN_TEAMID
export CHALLENGE=$IN_CHALLENGE
export SUBDOMAIN=`echo 'Attention: Team '$TEAMID' wants to steal the recipe for '$CHALLENGE'-cookies' | md5sum | cut -f1 -d" "`

printf 'Deploying challenge "'$CHALLENGE'" for team '$TEAMID'...\n\n'
envsubst < deployment.yml | kubectl apply -f -
envsubst < ingress.yml | kubectl apply -f -

printf '\nDone!\n'
printf '\nExposing challenge at...\n'
printf 'https://'$SUBDOMAIN'.web.ctf.htl-villach.at'
```
* Create a bash script `deprovision.sh` that deletes the challenge:
```bash
#!/bin/bash

IN_TEAMID=$1
IN_CHALLENGE=$2

export TEAMID=$IN_TEAMID
export CHALLENGE=$IN_CHALLENGE

printf 'Deprovisioning challenge "'$CHALLENGE'" for team '$TEAMID'...\n\n'

kubectl delete deployment deployment-${TEAMID}-${CHALLENGE}
kubectl delete service service-${TEAMID}-${CHALLENGE}
kubectl delete ingress ingress-${TEAMID}-${CHALLENGE}

printf '\nDone!\n'
printf '\nAll resources removed for challenge "'$CHALLENGE'" by team '$TEAMID'.\n'
```
* Create a bash script `teamkey.sh` that creates the scecrets:
```bash
#!/bin/bash

IN_TEAMID=$1
IN_TEAMKEY=$2

export TEAMID=$IN_TEAMID
export TEAMKEY=$IN_TEAMKEY

printf 'Setting up environment for team '$TEAMID'...\n\n'

kubectl create namespace namespace-team-$TEAMID
kubectl create secret generic teamkey-$TEAMID --from-literal=TEAMKEY=$TEAMKEY

printf '\nDone!\n'
```
* Navigate into the root folder:
```bash
cd ~/fastapi-deployment-api
```
* Create a `Dockerfile`:
```Dockerfile
# Use an official Python image
FROM python:3.10-slim

# Install gettext for envsubst
RUN apt-get update && apt-get install -y gettext && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Copy application files
COPY app .

# Install FastAPI and Uvicorn
RUN pip install fastapi uvicorn

# Expose port 8080
EXPOSE 8080

# Run the FastAPI application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```
* Create a `.env` file:
```env
API_KEY=mySuperSecureKey
```
* Create a `docker-compose.yml` file:
```yml
version: '3.8'

services:
  deployment-api:
    build: .
    container_name: fastapi-deployment
    ports:
      - "8080:8080"

    env_file: .env
    environment:
      - API_KEY=${API_KEY}

    volumes:
      - ./app:/app
      - /usr/local/bin/kubectl:/usr/local/bin/kubectl # Bind kubectl binary
      - /etc/rancher/k3s/k3s.yaml:/root/.kube/config:ro # Bind kube config for kubectl

    restart: always
```
* Deploy the service:
```bash
docker compose up --build -d
```

---

### Reconfigure the load balancer
* On the load balancer, edit the `nginx.conf` file by adding this:
> [!NOTE]
> Change the port to the right one for the internal traefik load balancer. Find the port when executing `kubectl get svc -n kube-system`. Further, consider changing the second level domain `web` to the right one.
```conf
# Load balancer for K3s
http {
    # Upstreams
    upstream traefik_nodes {
        least_conn;
        server 172.23.0.56:32430;
        server 172.23.0.57:32430;
    }

    upstream fastapi_nodes {
        least_conn;
        server 172.23.0.56:8080;
        server 172.23.0.57:8080;
    }

    # HTTP-to-HTTPS redirect
    server {
        listen 80;
        server_name ~^(?<subdomain>.+)\.web.ctf.htl-villach.at$;
        return 301 https://$host$request_uri;
    }

    # HTTPS for challenge
    server {
        listen 443 ssl;
        server_name ~^(?<subdomain>.+)\.web.ctf.htl-villach.at$;

        ssl_certificate /etc/nginx/certs/domain.crt;
        ssl_certificate_key /etc/nginx/certs/domain.key;

        location /teamkey {
            allow 172.23.0.55; # Webapp IP
            deny all;
            proxy_pass http://fastapi_nodes;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /deploy {
            allow 172.23.0.55; # Webapp IP
            deny all;
            proxy_pass http://fastapi_nodes;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /deprovision {
            allow 172.23.0.55; # Webapp IP
            deny all;
            proxy_pass http://fastapi_nodes;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location / {
            proxy_pass https://traefik_nodes;
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
docker compose down -v nginx && \
docker compose up --build -d nginx
```
* Test the teamkey creation endpoint:
> [!NOTE]
> Change the values for Bearer `mySuperSecureKey`, teamid `1` as well as teamkey `mySuperSecureTeamkeyForTeam1` to the correct ones.
```bash
curl -k -X POST "https://challenge.web.ctf.htl-villach.at/teamkey" \
-H "Authorization: Bearer mySuperSecureKey" \
-H "Content-Type: application/json" \
-d '{"teamid":"1", "teamkey":"mySuperSecureTeamkeyForTeam1"}'
```
* The output should look like this:
```bash
{
  "message": "Secret created successful", 
  "details": TEAMKEY: 28 byte
}
```
* Make sure the needed image is on the registry by accessing the web registry's interface `https://registry:8443`.
* Test the deployment endpoint:
> [!NOTE]
> Change the values for Bearer `mySuperSecureKey`, teamid `1` as well as challenge `ceasar-cipher` to the correct ones.
```bash
curl -k -X POST "https://challenge.web.ctf.htl-villach.at/deploy" \
-H "Authorization: Bearer mySuperSecureKey" \
-H "Content-Type: application/json" \
-d '{"teamid":"1", "challenge":"ceasar-cipher"}'
```
* The output should look like this:
```bash
{
  "message":"Deployment successful",
  "url":"https://<hash>.web.ctf.htl-villach.at"
}
```
* Test the deprovisioning endpoint:
> [!NOTE]
> Change the values for Bearer `mySuperSecureKey`, teamid `1` as well as challenge `ceasar-cipher` to the correct ones.
```bash
curl -k -X POST "https://challenge.web.ctf.htl-villach.at/deprovision" \
-H "Authorization: Bearer mySuperSecureKey" \
-H "Content-Type: application/json" \
-d '{"teamid":"1", "challenge":"ceasar-cipher"}'
```
* The output should look like this:
```bash
{
  "message":"Deprovision successful",
  "details": {
    "team":"1",
    "challenge":"ceasar-cipher"
  }
}
```

---

### Implement the Endpoints onto the Webapp
* tbd.

---

### Secure the Endpoints
> [!CAUTION]
> It does not have to be done - already secure
* Create a service account:
```bash
kubectl create serviceaccount deployer
```
* Bind the service account to a role:
```bash
kubectl create rolebinding deployer-binding --clusterrole=edit --serviceaccount=default:deployer
```
* Check the token is created automatically (look for `secrets`):
```bash
kubectl get serviceaccount deployer -o yaml
```
* If ***NOT***, create the secret manually and patch the user (+ verify again):
```bash
kubectl create secret generic deployer-token --from-literal=token=mySuperSecureToken --dry-run=client -o yaml | kubectl apply -f - && \
kubectl patch serviceaccount deployer -p '{"secrets":[{"name": "deployer-token"}]}' && \
kubectl get serviceaccount deployer -o yaml
```
* Retrieve the token (the output should be the valid token set earlier):
```bash
kubectl get secret deployer-token -o jsonpath='{.data.token}' | base64 --decode
```
* Set credentials for the deployer:
> [!IMPORTANT]
> This command has to be executed with sudo
```bash
sudo kubectl config set-credentials deployer --token=mySuperSecureToken
```
* Set the context for the deployer user:
> [!IMPORTANT]
> This command has to be executed with sudo
```bash
sudo kubectl config set-context deployer-context --cluster=default --user=deployer
```
* Verify the configuration:
```bash
kubectl config get-contexts
```
* Export the context:
```bash
kubectl config view --minify --context=deployer-context --raw > /tmp/deployer-kubeconfig.yaml && \
sudo mv /tmp/deployer-kubeconfig.yaml /etc/rancher/k3s/deployer-kubeconfig.yaml
```
* Edit the `docker-compose.yml` file:
```yml
 volumes:
      - ./app:/app
      - /usr/local/bin/kubectl:/usr/local/bin/kubectl
      - /etc/rancher/k3s/k3s.yaml:/root/.kube/config:ro # --- delete this ---
      - /etc/rancher/k3s/deployer-kubeconfig.yaml:/root/.kube/config:ro # --- add this ---
```
* Redeploy the container:
```bash
docker compose down -v && \
docker cmopose up --build -d
```
* Test the security implementation by navigating inside the container and listing all nodes:
```
kubectl get nodes
```

---

### Set up a DNS-Server for forwarding Subdomains (only locally)
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