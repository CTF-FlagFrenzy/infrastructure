# How to 'infrastructure' | From zero to hero

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

### Setup rootless Docker
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

### Setup Private Local Docker Registry with Authentication and HTTPS
* Create a directory to store the configuration as well as images and navigate into it:
```bash
mkdir -p ~/docker-registry/{data,certs,auth,portainer} && \
cd ~/docker-registry
```
* Obtain the htpasswd utility by installing the apache2-utils package:
```bash
sudo apt install apache2-utils -y
```
* Create the first user, replacing `username` with the username you want to use:
> [!NOTE]
> --> The -B flag orders the use of the bcrypt algorithm, which Docker requires.  
> --> To add more users, re-run the previous command without -c, which creates a new file
```bash
htpasswd -Bc auth/registry.password username
```
* Create certificates for https communication:
> [!NOTE]
> Replace `hostname` with the hostname of the machine the container stack is running on
```bash
openssl req -x509 -nodes -newkey rsa:2048 -keyout certs/domain.key -out certs/domain.crt -days 365 -addext "subjectAltName = DNS.1:hostname,DNS.2:registry"
```
* Create and open the `docker-compose.yaml` file and define the registry as well as the ui with its proxy:
```yaml
version: '3' # Specify Docker Compose version


networks:
  my-registry-net:
    driver: bridge # Custom network to facilitate inter-service communication


volumes:
  # Bind host directory to store Docker registry data
  my-registry-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: ./data

  # Bind host directory to store Portainer data
  my-portainer-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: ./portainer


services:


  registry:
    image: registry:2
    restart: always # Always restart on failure or during container stop
    ports:
      - 5000:5000 # Expose Docker registry on port 5000

    environment:
      # Set the address and port for the registry
      REGISTRY_HTTP_ADDR: 0.0.0.0:5000

      # Set up TLS for secure communication
      REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt # Path to TLS certificate
      REGISTRY_HTTP_TLS_KEY: /certs/domain.key # Path to TLS key

      # Secret used to sign tokens for HTTP basic authentication
      REGISTRY_HTTP_SECRET: supersecuresecret

      # Authentication configuration using htpasswd file
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry-Realm # Authentication realm
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/registry.password # Path to htpasswd file

      # Storage configuration
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /data # Directory where images will be stored

      # Enable image deletion from the registry
      REGISTRY_STORAGE_DELETE_ENABLED: 'true'

    volumes:
      - my-registry-data:/data # Mount registry data volume
      - ./certs:/certs # Mount local certificate directory
      - ./auth:/auth # Mount local authentication directory

    networks:
      - my-registry-net # Attach registry service to the custom network


  registry-ui:
    image: joxit/docker-registry-ui:latest
    restart: always # Always restart if the container stops

    environment:
      REGISTRY_TITLE: FlagFrenzy | Registry # Custom title for UI
      REGISTRY_SECURED: true # Enable secured access to the registry
      SINGLE_REGISTRY: true # Only a single registry is used
      DELETE_IMAGES: true # Allow image deletion from the UI
      SHOW_CONTENT_DIGEST: true # Show content digest in UI
      SHOW_CATALOG_NB_TAGS: true # Display number of tags in catalog
      TAGLIST_PAGE_SIZE: 100 # Set page size for the tag list
      CATALOG_ELEMENTS_LIMIT: 10 # Limit number of elements in the catalo
      THEME: dark # Set UI theme to dark - no one uses light mode

      # Same authentication settings as the registry service
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry-Realm
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/registry.password

    depends_on:
      - registry # Ensure the registry service starts before the UI

    volumes:
      - ./certs:/etc/nginx/certs:ro # Mount certificates directory as read-only
      - ./auth:/auth # Mount authentication directory

    networks:
      - my-registry-net # Attach registry-ui service to the custom network


  nginx:
    image: nginx:latest
    restart: always # Always restart if the container stops
    ports:
      - 8443:443 # Expose Nginx on port 8443 (HTTPS)

    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro # Mount Nginx configuration file as read-only
      - ./certs:/etc/nginx/certs:ro # Mount certificates directory as read-only

    depends_on:
      - registry # Ensure the registry starts before Nginx
      - registry-ui # Ensure the registry UI starts before Nginx

    networks:
      - my-registry-net # Attach Nginx service to the custom network


  portainer:
    image: portainer/portainer-ce
    ports:
      - 9443:9443 # Expose Portainer on port 9443 (HTTPS)

    restart: on-failure # Restart only on failure

    volumes:
      - my-portainer-data:/data # Mount Portainer data volume
      - $XDG_RUNTIME_DIR/docker.sock:/var/run/docker.sock # Rootless Docker socket mount for Portainer

    networks:
      - my-registry-net # Attach Portainer to the custom network
```
* Create and open the `nginx.conf` file and define the proxy rules including the forwarding:
```nginxconf
# Set the number of worker processes to automatically adjust to the number of CPU cores
worker_processes auto;

events {
    # Maximum number of simultaneous connections that can be handled by each worker process
    worker_connections 1024;
}

http {
    # Upstream block for Docker Registry
    upstream registry {
        server registry:5000; # Docker registry running on port 5000 within the Docker network
    }

    # Upstream block for Docker Registry UI
    upstream registry-ui {
        server registry-ui:80; # Docker registry UI running on port 80 within the Docker network
    }

    server {
        # Listen on port 443 with SSL enabled
        listen 443 ssl;
        server_name manager1; # The hostname used to access the Docker registry and UI

        # Path to the SSL certificate and private key
        ssl_certificate /etc/nginx/certs/domain.crt;
        ssl_certificate_key /etc/nginx/certs/domain.key;


        # Route all API requests (e.g., /v2/) to the Docker Registry
        location /v2/ {
            proxy_pass https://registry; # Forward API requests to the registry upstream
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # These headers are required for Docker to trust the registry
            # certificate and enable client-side certificate validation
            proxy_ssl_verify on; # Enable SSL verification for registry communication
            proxy_ssl_trusted_certificate /etc/nginx/certs/domain.crt; # Trusted certificate for validation
            proxy_ssl_session_reuse off; # Disable SSL session reuse for better security

            proxy_set_header Authorization $http_authorization; # Forward authorization headers
        }


        # Route all other requests (like UI access) to the Docker Registry UI
        location / {
            proxy_pass http://registry-ui; # Forward UI requests to the registry-ui upstream
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```
* Deploy the container stack with its services:
```bash
docker compose up --build
```
* Verify that the stack is working properly by accessing the ui (`https://ip_or_dns_to_machine:8443`):

---

### Use Private Local Docker Registry
* Add registry host dns name to your client:
> [!NOTE]
> Change `ip` to the machine's ip address where the registry is running.
```bash
sudo echo 'ip my_registry_host' >> /etc/hosts
```
* Copy the container-host's certificate to the client:
```bash
sudo scp user@my_registry_host:~/docker-registry/certs/domain.crt /usr/share/ca-certificates
```
* Create a folder for the registry certificate authority and copy the certificate:
```bash
sudo mkdir -p /etc/docker/certs.d/my_registry_host\:5000 && \
sudo cp /usr/share/ca-certificates/domain.crt /etc/docker/certs.d/my_registry_host\:5000/ca.crt
```
* Restart the certificate service to apply the changes for the certificate trust:
> [!NOTE]
> In the pop up, click `yes`, select the certificate using `space` and press `enter` to finish.
```bash
sudo dpkg-reconfigure ca-certificates
```
* Create a new image from a container:
```bash
docker commit container_name image_name
```
* Login to the registry:
```bash
docker login https://my_registry_host:5000
```
> [!IMPORTANT]
> If the certificate is not trusted due to an unknown certificate-authority (ca), consider restarting the client. The configuration might not have been applied.
* Tag an image to prepare it for pushing:
```bash
docker tag base_image my_registry_host:5000/image[:tag]
```
* Push the image onto the registry:
```bash
docker push my_registry_host:5000/image[:tag]
```
* Pull an image from registry:
```bash
docker pull my_registry_host:5000/image[:tag]
```
