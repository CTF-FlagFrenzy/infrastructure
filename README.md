# How to 'infrastructure' | From zero to hero (Docker)

---

### Installing Docker
* Uninstall all conflicting packages:
```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```
* Update repository:
```bash
sudo apt-get update
```
* Add Docker's official gpg-key:
```bash
sudo apt-get install ca-certificates curl && \
sudo install -m 0755 -d /etc/apt/keyrings && \
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc && \
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
* Add the repository to apt-sources:
```bash
echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
* Update again:
```bash
sudo apt-get update
```
* Install Docker packages:
```bash
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
* Verify that Docker is running:
```bash
sudo docker run hello-world
```

---

### Set up rootless Docker
* Install uidmap:
```bash
sudo apt-get install -y uidmap
``` 
* Shut down Docker deamon:
```bash
sudo systemctl disable --now docker.service docker.socket
```
* Install Dockerd-Rootless-Setup:
```bash
dockerd-rootless-setuptool.sh install
```
* Define variable DOCKER_HOST:
```bash
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
```
* Verify that Docker is running rootless:
```bash
docker run hello-world
```