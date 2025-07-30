# ODK-X Sync Endpoint: A Complete Guide for Ubuntu 22.04 on Proxmox

This guide provides a comprehensive, step-by-step walkthrough for setting up the ODK-X Sync Endpoint on an Ubuntu 22.04 LTS server, virtualized using Proxmox VE. It includes not only the standard installation steps but also critical troubleshooting solutions discovered during a real-world deployment.

**Author:** Shaykins
**Date:** July 30, 2025
**System:** Ubuntu 22.04.5 LTS on Proxmox VE

---

## A Note on the Official Documentation

This guide is intended to supplement the [Official ODK-X Sync Endpoint Manual Setup Guide](https://docs.odk-x.org/sync-endpoint-manual-setup/). The official guide provides the foundational "happy path" for installation.

However, during a real-world setup on a clean Ubuntu 22.04 server, several critical issues were encountered that are not covered in the official documentation. This guide provides the solutions to those specific problems, which include:

1.  **Missing Nginx Configuration:** The `sync-endpoint-default-setup` repository did not contain the necessary Nginx configuration files, causing a total failure of the reverse proxy. This guide provides the exact content for these missing files.
2.  **Docker Swarm Startup Race Condition:** The Nginx container would start faster than the backend `sync` service, causing it to crash with a `host not found` error. This guide implements a `resolver` directive in the Nginx configuration to solve this race condition.
3.  **Real-World Troubleshooting:** This document includes the diagnostic steps (like checking Docker Swarm configs and service logs) that were required to identify and solve these issues.

Our goal is to help others who may encounter these same challenges and provide a clear path to a successful deployment.

---

## Part 1: Proxmox VM & Initial Ubuntu Setup

### 1.1. Proxmox VM Configuration

1.  **Create the VM:** Start with a clean Ubuntu 22.04 LTS template.
2.  **System Resources:**
    * **CPU:** 2+ Cores
    * **Memory:** 4GB+ RAM
    * **Disk:** 50GB+ (depending on expected data volume)
3.  **Network:**
    * Use a bridged network interface.
    * Configure a static IP address for the VM to ensure stable access. In this guide, we use `your.server.ip` as a placeholder. **Remember to replace this with your server's actual static IP address.**

### 1.2. Ubuntu Server Preparation

Once the VM is running, SSH into it and perform the following initial setup.

1.  **Update System Packages:**
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```

2.  **Install Essential Dependencies:**
    ```bash
    sudo apt install -y git docker.io docker-compose maven openjdk-11-jdk
    ```

3.  **Initialize Docker Swarm Mode:**
    The ODK-X stack is deployed as a Docker Swarm. Initialize it with this command:
    ```bash
    sudo docker swarm init
    ```

---

## Part 2: ODK-X Sync Endpoint Installation & Build

With the server prepared, proceed with building the ODK-X components.

1.  **Clone the Setup Repository:**
    ```bash
    # Navigate to your home directory
    cd ~
    
    # Clone the main setup repository
    git clone [https://github.com/odk-x/sync-endpoint-default-setup](https://github.com/odk-x/sync-endpoint-default-setup)
    cd sync-endpoint-default-setup
    ```

2.  **Clone the Sync Endpoint Source Code:**
    ```bash
    # A full clone is performed to get the entire repository history.
    git clone [https://github.com/odk-x/sync-endpoint](https://github.com/odk-x/sync-endpoint)
    ```

3.  **Build the Sync Endpoint `.war` File:**
    This requires Maven and will compile the Java application.
    ```bash
    cd sync-endpoint
    mvn clean install
    cd ..
    ```

4.  **Build Required Docker Images:**
    These commands build the necessary service images locally.
    ```bash
    docker build --pull -t odk/sync-web-ui [https://github.com/odk-x/sync-endpoint-web-ui.git](https://github.com/odk-x/sync-endpoint-web-ui.git)
    docker build --pull -t odk/db-bootstrap db-bootstrap
    docker build --pull -t odk/openldap openldap
    docker build --pull -t odk/phpldapadmin phpldapadmin
    ```

---

## Part 3: Configuration - The Critical Steps

This section details the most critical and often overlooked configuration steps. Failure to follow these steps precisely is the primary cause of login issues.

### 3.1. The Nginx Configuration Anomaly

A common pitfall is to assume the Nginx service is configured via a single `nginx.conf` file. **This is incorrect.**

The `docker-compose.yml` file for this project uses the **Docker Swarm `configs` feature**. This means it does not look for a single `nginx.conf` file. Instead, it expects several smaller, specifically named configuration files to be present in the `./config/nginx/` directory. Editing or creating `nginx.conf` will have no effect.

The key discovery during troubleshooting was that these files were missing entirely, causing Nginx to fail silently or use a default configuration that could not route traffic to the backend services.

### 3.2. Creating the Correct Nginx Configuration Files

The following commands will create the three required Nginx configuration files with the correct content.

1.  **Create the Main Server File (`sync-endpoint-http.conf`):**
    This file sets up the server and includes the other configs. The `resolver` directive is crucial to prevent a startup race condition where Nginx starts before the `sync` service is discoverable on the Docker network.
    ```bash
    cat <<'EOF' > ./config/nginx/sync-endpoint-http.conf
    server {
      listen 80;
      # Add the resolver to fix service discovery.
      # 127.0.0.11 is Docker's internal DNS, do not change this.
      resolver 127.0.0.11 valid=5s;
    
      # Include the other configuration files
      include /etc/nginx/conf.d/proxy_buffer.conf;
      include /etc/nginx/conf/sync-endpoint-locations.conf;
    }
    EOF
    ```

2.  **Create the Locations & Routing File (`sync-endpoint-locations.conf`):**
    This file contains the core proxy logic, telling Nginx how to route requests for `/` to the `sync` service and requests for `/web-ui` to the `web-ui` service.
    ```bash
    cat <<'EOF' > ./config/nginx/sync-endpoint-locations.conf
    # Increase max upload size. Important for large attachments.
    client_max_body_size 100M;
    
    # gzip compression
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
      proxy_redirect off;
    }
    
    location /web-ui {
      proxy_pass http://web-ui:8080;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header Host $http_host;
      proxy_redirect off;
    }
    EOF
    ```

3.  **Create the Proxy Buffer File (`proxy_buffer.conf`):**
    This file provides performance tuning for Nginx's proxy buffering.
    ```bash
    cat <<'EOF' > ./config/nginx/proxy_buffer.conf
    proxy_buffering on;
    proxy_buffer_size 16k;
    proxy_buffers 8 16k;
    proxy_busy_buffers_size 32k;
    EOF
    ```

### 3.3. Configure the Sync Endpoint Hostname

The Sync Endpoint service needs to know its own public hostname.
```bash
# Navigate to the config directory
cd config/sync-endpoint

# Add your hostname to the security properties file
# Replace your.server.ip with your server's actual static IP
echo "security.server.hostname=your.server.ip" >> security.properties
cd ../..
```

---

## Part 4: Deployment & Final Setup

With all configurations in place, you can now deploy the stack.

1.  **Deploy the Docker Stack:**
    ```bash
    docker stack deploy -c docker-compose.yml syncldap
    ```
    Wait about one minute for all services to initialize. You can check the status with `docker service ls`.

2.  **Configure LDAP User and Permissions:**
    * Access phpLDAPadmin in your browser: `https://your.server.ip:40000`
    * Log in as the administrator:
        * **Login DN:** `cn=admin,dc=example,dc=org`
        * **Password:** The password you set in your `ldap.env` file (default is `password`).
    * Create a new user (e.g., `odkuser`) under `ou=people`.
    * **CRITICAL LOGIN STEP:** For the user to be able to log in, they must be assigned the `SYNCHRONIZE_TABLES` role.
        * Navigate to the group with **`gidNumber=503`** (`default_prefix_synchronize_tables`).
        * Add your new user's username (e.g., `odkuser`) to the **`memberUid`** attribute of this group.

3.  **Test the Login:**
    Navigate to the Web UI at `http://your.server.ip/web-ui/` and log in with the user you just created. It should now be successful.

---

## Part 5: Troubleshooting Common Errors

If you encounter issues, here are the solutions to the problems discovered during this build process.

* **Error:** `nginx: [emerg] host not found in upstream "sync"` in `docker service logs syncldap_nginx`.
    * **Cause:** This is a classic **startup race condition** in Docker Swarm. When you deploy the stack, all containers start at once. The Nginx container is lightweight and starts very quickly. It immediately tries to find the IP address for the service named "sync". However, the `sync` service takes longer to initialize. If Nginx looks for "sync" before the Docker network has registered it, Nginx fails to find the host and crashes.
    * **Solution:** Adding the `resolver 127.0.0.11 valid=5s;` directive to the `sync-endpoint-http.conf` file.
        * `resolver 127.0.0.11`: This explicitly tells Nginx to use Docker's own internal DNS server to find other services.
        * `valid=5s`: This is the key part of the fix. It tells Nginx not to give up if it can't find the "sync" host immediately. Instead, it will cache the result for 5 seconds and try again, giving the `sync` service enough time to start and register itself on the network.

* **Error:** `failed to update config ... Error response from daemon: rpc error: code = InvalidArgument desc = only updates to Labels are allowed`.
    * **Cause:** You cannot change a Docker Swarm config while it is in use by a running service.
    * **Solution:** You must first remove the old, empty configs. This requires tearing down the stack completely, which releases the lock on the configuration files.
        ```bash
        docker stack rm syncldap
        # Wait 10 seconds for services to stop
        docker config rm syncldap_com.nginx.sync-endpoint.conf
        # ... (and the other two nginx configs)
        docker stack deploy -c docker-compose.yml syncldap
        
