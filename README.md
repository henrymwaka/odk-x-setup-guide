# ODK-X Sync Endpoint: Definitive Installation Guide

This guide provides a complete, verified process for deploying a fully functional ODK-X Sync Endpoint on an Ubuntu server. It incorporates lessons learned from common installation issues to ensure a smooth, reliable setup.

---

## Part 1: Initial Server Preparation

### 1.1. Verify Prerequisites
Before starting, confirm that **Java**, **Maven**, and **Docker** are installed and that **Docker Swarm Mode** is active.

```bash
# Check for Java version 11 or higher
java -version

# Check for Maven version 3.3.3 or higher
mvn -v

# Check the Docker version and Swarm status
docker --version
docker info | grep Swarm
```

> **Note:** If any software is missing, install it using `sudo apt install`.  
> If Swarm is inactive, enable it with:
```bash
docker swarm init
```

---

### 1.2. Clone the ODK-X Repositories
Create a dedicated project directory and download the required code from GitHub.

```bash
mkdir ~/odk-x-setup && cd ~/odk-x-setup

git clone https://github.com/odk-x/sync-endpoint-default-setup
cd sync-endpoint-default-setup

git clone https://github.com/odk-x/sync-endpoint
```

---

## Part 2: Build the Application Components

### 2.1. Build the Main Application (via Maven)
```bash
cd sync-endpoint

# This step takes 20+ minutes
mvn clean install

cd ..
```

---

### 2.2. Build Helper Components
```bash
docker build --pull -t odk/sync-web-ui https://github.com/odk-x/sync-endpoint-web-ui.git
docker build --pull -t odk/db-bootstrap db-bootstrap
docker build --pull -t odk/openldap openldap
docker build --pull -t odk/phpldapadmin phpldapadmin
```

---

## Part 3: Server Configuration

### 3.1. Configure Nginx
```bash
mkdir -p ./config/nginx

# Main server config
cat > ./config/nginx/sync-endpoint-http.conf <<'EOF'
server {
  listen 80;
  resolver 127.0.0.11 valid=5s;
  include /etc/nginx/conf.d/proxy_buffer.conf;
  include /etc/nginx/conf/sync-endpoint-locations.conf;
}
EOF

# Routing rules
cat > ./config/nginx/sync-endpoint-locations.conf <<'EOF'
client_max_body_size 100M;
gzip on;
gzip_disable "msie6";
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_buffers 16 8k;
gzip_http_version 1.1;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

location / {
  proxy_pass http://sync:8080;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header Host $http_host;
  proxy_set_header X-Forwarded-Host $host;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_redirect off;
}

location /web-ui {
  proxy_pass http://web-ui:8080;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header Host $http_host;
  proxy_set_header X-Forwarded-Host $host;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_redirect off;
}
EOF

# Proxy buffer settings
cat > ./config/nginx/proxy_buffer.conf <<'EOF'
proxy_buffering on;
proxy_buffer_size 16k;
proxy_buffers 8 16k;
proxy_busy_buffers_size 32k;
EOF
```

---

### 3.2. Set Server Hostname
```bash
echo "security.server.hostname=YOUR_SERVER_IP" >> config/sync-endpoint/security.properties
```

---

### 3.3. Create Docker Compose File
```bash
cat > docker-compose.yml <<'YAML'
version: "3.3"

services:
  ldap-service:
    image: odk/openldap
    deploy:
      replicas: 1
    networks: [ldap-network]
    volumes:
      - ldap-vol:/var/lib/ldap
      - ldap-slapd.d-vol:/etc/ldap/slapd.d
    env_file: [ldap.env]

  phpldapadmin:
    image: odk/phpldapadmin
    deploy:
      replicas: 1
    ports:
      - "${PHP_LDAPADMIN_PORT:-40000}:443"
    networks: [ldap-network]
    env_file: [ldap.env]

  db:
    image: postgres:9.6
    deploy:
      replicas: 1
    networks: [db-network]
    volumes:
      - db-vol:/var/lib/postgresql/data
    env_file: [db.env]

  db-bootstrap:
    image: odk/db-bootstrap
    deploy:
      replicas: 1
      restart_policy: { condition: none }
      placement:
        constraints: [ "node.role == manager" ]
    networks: [db-network, sync-network]
    volumes:
      - type: bind
        source: /var/run/docker.sock
        target: /var/run/docker.sock
    env_file: [db.env, sync.env]

  sync:
    image: odk/sync-endpoint
    deploy:
      replicas: 1
    networks: [ldap-network, db-network, sync-network]
    env_file: [sync.env]
    secrets:
      - org.opendatakit.aggregate.security.properties
      - org.opendatakit.aggregate.jdbc.properties

  web-ui:
    image: odk/sync-web-ui
    deploy:
      replicas: 1
    networks: [sync-network]
    hostname: web-ui
    configs:
      - org.opendatakit.sync-web-ui.application.properties

  nginx:
    image: nginx:1.21.3
    deploy:
      replicas: 1
    ports:
      - "80:80"
    networks: [sync-network]
    configs:
      - source: com.nginx.sync-endpoint.conf
        target: /etc/nginx/conf.d/default.conf
      - source: com.nginx.sync-endpoint-locations.conf
        target: /etc/nginx/conf/sync-endpoint-locations.conf
      - source: com.nginx.proxy_buffer.conf
        target: /etc/nginx/conf.d/proxy_buffer.conf
    depends_on: [sync, web-ui]

networks:
  ldap-network:
    driver: overlay
    driver_opts: { encrypted: "" }
    internal: true
  db-network:
    driver: overlay
    driver_opts: { encrypted: "" }
    internal: true
  sync-network:
    driver: overlay
    driver_opts: { encrypted: "" }
    internal: true

volumes:
  db-vol: {}
  ldap-vol: {}
  ldap-slapd.d-vol: {}

configs:
  org.opendatakit.sync-web-ui.application.properties:
    file: ./config/web-ui/application.properties
  com.nginx.sync-endpoint.conf:
    file: ./config/nginx/sync-endpoint-http.conf
  com.nginx.sync-endpoint-locations.conf:
    file: ./config/nginx/sync-endpoint-locations.conf
  com.nginx.proxy_buffer.conf:
    file: ./config/nginx/proxy_buffer.conf

secrets:
  org.opendatakit.aggregate.security.properties:
    file: ./config/sync-endpoint/security.properties
  org.opendatakit.aggregate.jdbc.properties:
    file: ./config/sync-endpoint/jdbc.properties
YAML
```

---

## Part 4: Deployment and Verification

### 4.1. Clean Deployment
```bash
docker stack rm syncldap
sleep 10
docker volume rm $(docker volume ls -f "label=com.docker.stack.namespace=syncldap" -q)

docker stack deploy -c docker-compose.yml syncldap
```

---

### 4.2. Verify
```bash
docker service ls
```
✅ All services should show `1/1` replicas.

---

### 4.3. Create First User (phpLDAPadmin)
1. Go to: `http://YOUR_SERVER_IP:40000`  
2. **Login DN:** `cn=admin,dc=example,dc=org`  
   **Password:** `admin`  
3. Create a new user under **ou=people**.  
4. Assign the user to group `gidNumber=503`.

You can now log in to: `http://YOUR_SERVER_IP/web-ui/`.

---

## Part 5: Connecting a Mobile Device

### 5.1. Enable Unsafe Authentication (HTTP only)
In **ODK-X Survey**:
- **Menu** → **Admin Settings** → **User & Device Identity** → **Server Settings** → enable **Allow Unsafe/Unsecured Authentication**.

---

### 5.2. Configure Server Settings
- **Server URL:** `http://YOUR_SERVER_IP`
- **Username & Password:** LDAP credentials created above.

---

### 5.3. Test Sync
Press **🔄 Sync**. ✅ Success if no errors appear.

---

## Part 6: Maintenance

### Complete Wipe & Reinstall
```bash
docker stack rm syncldap
sleep 10
docker volume rm $(docker volume ls -f "label=com.docker.stack.namespace=syncldap" -q)
```
Then redeploy with **4.1**.

---

**Author:** Adapted for reliability from ODK-X official documentation with additional tested fixes.
