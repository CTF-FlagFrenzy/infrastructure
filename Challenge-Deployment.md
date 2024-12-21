# How to 'infrastructure' | Deploying Challenges

---

> [!IMPORTANT]
> Before getting started, consider working through the guides `README.md`, `Docker-Registry.md` and `K3-Cluster.md` in order to set up everything needed in advance.

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
  name: challenge-1
  labels:
    app: challenge
    team: '1'
spec:
  replicas: 1
  selector:
    matchLabels:
      app: challenge
      team: '1'
  template:
    metadata:
      labels:
        app: challenge
        team: '1'
    spec:
      containers:
      - name: challenge-app
        image: registry:5000/ceasar_cypher
        ports:
          - containerPort: 8005
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
  name: challenge-1
spec:
  selector:
    app: challenge
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

### Set up NGINX-Ingress to Forward Service