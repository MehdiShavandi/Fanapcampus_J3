
## Step1: preparing and hardening OS with ansible

## Step2: Install and config docker service with ansible

**Best docker daemon configuration**

```bash

sudo vim /etc/docker/daemon.json

{
  "registry-mirrors": ["https://hub.hamdocker.ir/" , "https://repo.shavandy.ir" ],
  "insecure-registries": ["https://repo.shavandy.ir"],
  "bip": "172.100.0.1/24",
  "data-root": "/mnt/data",
  "log-driver": "json-file",
  "log-level": "info",
  "log-opts": {
    "cache-disabled": "false",
    "cache-max-file": "5",
    "cache-max-size": "20m",
    "cache-compress": "true",
    "labels": "MeCanHost",
    "max-file": "5",
    "max-size": "10m"
  }
}

```

## registry with nginx

### Create the main nginx configuration. Paste this code block into a new file called auth/nginx.conf with Ansible role nginx :

```bash

vim nginx.conf

events {
    worker_connections  1024;
}

http {

  upstream docker-registry {
    server registry:5000;
  }

  ## Set a variable to help us decide if we need to add the
  ## 'Docker-Distribution-Api-Version' header.
  ## The registry always sets this header.
  ## In the case of nginx performing auth, the header is unset
  ## since nginx is auth-ing before proxying.
  map $upstream_http_docker_distribution_api_version $docker_distribution_api_version {
    '' 'registry/2.0';
  }

  server {
    listen 443 ssl;
    server_name repo.shavandy.ir;

    # SSL
    ssl_certificate /etc/nginx/conf.d/cert.pem;
    ssl_certificate_key /etc/nginx/conf.d/key.pem;

    # Recommendations from https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html
    ssl_protocols TLSv1.1 TLSv1.2;
    ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;

    # disable any limits to avoid HTTP 413 for large image uploads
    client_max_body_size 0;

    # required to avoid HTTP 411: see Issue #1486 (https://github.com/moby/moby/issues/1486)
    chunked_transfer_encoding on;

    location /v2/ {
      # Do not allow connections from docker 1.5 and earlier
      # docker pre-1.6.0 did not properly set the user agent on ping, catch "Go *" user agents
      if ($http_user_agent ~ "^(docker\/1\.(3|4|5(?!\.[0-9]-dev))|Go ).*$" ) {
        return 404;
      }

      # To add basic authentication to v2 use auth_basic setting.
      #auth_basic "Registry realm";
      #auth_basic_user_file /etc/nginx/conf.d/nginx.htpasswd;

      ## If $docker_distribution_api_version is empty, the header is not added.
      ## See the map directive above where this variable is defined.
      add_header 'Docker-Distribution-Api-Version' $docker_distribution_api_version always;

      proxy_pass                          http://registry:5000;
      proxy_set_header  Host              $http_host;   # required for docker client's sake
      proxy_set_header  X-Real-IP         $remote_addr; # pass on real client's IP
      proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
      proxy_set_header  X-Forwarded-Proto $scheme;
      proxy_read_timeout                  900;
    }
  }
  server {
      listen 80;
      server_name repo.shavandy.ir;
      return 301 https://$host$request_uri;
  }

```

### Create a password file auth/nginx.htpasswd for mehdi with server.

```bash

# Change directory
cd /opt/services

# create htpasswd file
htpasswd -Bc auth/nginx.htpasswd mehdi

# check
cat auth/nginx.htpasswd

```

### create certificate files to the auth/ directory with Ansible.

```bash 

!/bin/bash
# Create certificate files in the auth/ directory
CERT_LOCATION=/opt/services/auth

# Generate key and cert
openssl req -x509 -nodes -newkey rsa:4096 -days 365 \
-keyout $CERT_LOCATION/key.pem \
-subj "/C=IR/ST=Iran/L=Tehran/O=DockerMe/OU=IT/CN=Shavandy.ir/emailAddress=mhdi.shavandi@gmail.com" \
-out $CERT_LOCATION/cert.pem 

# Check certificate
openssl x509 -text -noout -in $CERT_LOCATION/cert.pem

```

### run with docker compose with Ansible role registry .

```bash

version: '3'

services:
  registry:
    image: registry:2
    volumes:
      - ./data:/var/lib/registry
      - /opt/services/auth:/auth
    networks:
      - web_net
    environment:  
       REGISTRY_AUTH: htpasswd
       REGISTRY_AUTH_HTPASSWD_PATH: /auth/nginx.htpasswd
       REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm

networks:
  web_net:
    external: true

```

### login to registry with server.

```bash

docker login https://repo.shavandy.ir -u mehdi -p mehdi

```

### Image Tag and Push to local Registry with server.

```bash

# tag image
docker tag wordpress:latest repo.shavandy.ir/wordpress:latest
# push image
docker push repo.shavandy.ir/wordpress:latest

# tag image
docker tag mysql:5.7 repo.shavandy.ir/mysql:5.7
# push image
docker push repo.shavandy.ir/mysql:5.7

# check repository
curl -u "mehdi:mehdi" -k https://repo.MeCan.ir/v2/_catalog

```

## Step3:  Running wordpress with docker


### create nginx config file for wordpress proxy pass with Ansible role ngnix.

```bash

server {
  listen 443 ssl;
  server_name wp.shavandy.ir;

  # SSL
  ssl_certificate /etc/nginx/conf.d/cert.pem;
  ssl_certificate_key /etc/nginx/conf.d/key.pem;

  location / {
    proxy_pass            http://wordpress:80;
    proxy_set_header  Host              $http_host;   # required for docker client's sake
    proxy_set_header  X-Real-IP         $remote_addr; # pass on real client's IP
    proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header  X-Forwarded-Proto $scheme;
    add_header X-Powered-By "Ahmad Rafiee | shavandy.ir";
    }
 }

 server {
    listen 80;
    server_name wp.shavandy.ir;
    return 301 https://$host$request_uri;
}

}

```

### create self sign certificate with openssl command and check it with Ansible. 

```bash

!/bin/bash
# Create certificate files in the auth/ directory
CERT_LOCATION=/opt/services/auth

# Generate key and cert
openssl req -x509 -nodes -newkey rsa:4096 -days 365 \
-keyout $CERT_LOCATION/key.pem \
-subj "/C=IR/ST=Iran/L=Tehran/O=DockerMe/OU=IT/CN=Shavandy.ir/emailAddress=mhdi.shavandi@gmail.com" \
-out $CERT_LOCATION/cert.pem 

# Check certificate
openssl x509 -text -noout -in $CERT_LOCATION/cert.pem

```

### compose file with Ansible role wordpress.

```bash

---
version: '3.8'

networks:
  app_net:
    external: false
    name: app_net
  web_net:
    external: true
    name: web_net

    
volumes:
  wp_db:
    name: wp_db
  wp_wp:
    name: wp_wp

services:
  db:
    image: mysql:5.7
    container_name: mysql
    volumes:
      - wp_db:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: sdfascsdvsfdvweliuoiquowecefcwaefef
      MYSQL_DATABASE: DockerMe
      MYSQL_USER: DockerMe
      MYSQL_PASSWORD: sdfascsdvsfdvweliuoiquowecefcwaefef
    networks:
      - app_net

  wordpress:
    image: wordpress:latest
    container_name: wordpress
    volumes:
      - wp_wp:/var/www/html/
    depends_on:
      - db
    restart: always
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: DockerMe
      WORDPRESS_DB_NAME: DockerMe
      WORDPRESS_DB_PASSWORD: sdfascsdvsfdvweliuoiquowecefcwaefef
    ports:
      - 8000:80
    networks:
      - web_net
      - app_net

```
