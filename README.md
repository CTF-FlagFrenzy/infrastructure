# How to 'infrastructure' | From zero to hero (Docker)

---

### Table of Contents
- [Install Docker](#install-docker)
- [Set up rootless Docker](#set-up-rootless-docker)
- [Set up a portainer for easier management](#set-up-a-portainer-for-easier-management)
- [Set up podman-compose (when rootless docker does not work)](#set-up-podman-compose-when-rootless-docker-does-not-work)

> [!IMPORTANT]
> Before getting started, consider setting up blank ubuntu servers (1 Load Balancer / 2 Matser Nodes / 4 Agent Nodes) in advance. In this use case, everything is testet and working using ubuntu 22.04 as well as 24.x.

---

### Install Docker
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

---

### Set up a portainer for easier management
* Create a repository for portainer and navigate there:
```bash
mkdir ~/portainer && \
cd ~/portainer
```
* Create the configuration file `docker-compose.yml`:
```yaml
networks:
  my-network:

volumes:
  portainer-data:

services:
  # Service for management container
  manage-docker:
    image: portainer/portainer-ce
    ports:
      - "9500:9443"
    volumes:
      - portainer-data:/data
      - $XDG_RUNTIME_DIR/docker.sock:/var/run/docker.sock # Portainer - rootless mount
    restart: always

    networks:
      - my-network
```
* Deploy the Service (it can be accessed using `<your_server_ip>:9500` after a few seconds):
```bash
docker compose up --build -d
```

---

### Set up podman-compose (when rootless docker does not work)
* Disabel the docker socket:
```bash
sudo systemctl disable docker && \
sudo systemctl disable docker.socket
```
* Remove all docker packets:
```bash
sudo dnf remove -y docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine && \
sudo dnf remove -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
* Delete the directories:
```bash
sudo rm -rf /var/lib/docker /var/lib/containerd && \
sudo rm -f /usr/local/bin/docker-compose && \
sudo rm -f /etc/yum.repos.d/docker-ce.repo
```
* Clear the Cache:
```bash
sudo dnf clean all && \
sudo dnf makecache
```
* Install podman-compose:
```bash
sudo dnf install -y podman podman-docker && \
sudo dnf install -y python3-pip && \
sudo pip3 install podman-compose
```
* Start podman:
```bash
sudo systemctl --user enable podman.socket
```
* Export the runtime directory:
> [!NOTE]
> If an error occurs, confirm the runtime directory. If not, relogin
> ```bash
> id -u && \
> ls -ld /run/user/$(id -u)
> ```
```bash
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock
```
* Verify installation:
```bash
podman run hello-world
```