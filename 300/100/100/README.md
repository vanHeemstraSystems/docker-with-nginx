# 100 - How to forward Nginx incoming request to Docker container port

Based on "How to forward Nginx incoming request to Docker container port" at https://popovserhii.com/how-to-forward-nginx-incoming-request-to-docker-container-port

Supposed, we have next configuration in ```docker-compose.dev.yml```:

```
version: "3.7"

# See https://stackoverflow.com/questions/29261811/use-docker-compose-env-variable-in-dockerbuild-file
services:

  webui:
    build:
      context: ./webui
      dockerfile: Dockerfile.dev
      args: # from env_file
        IMAGE_REPOSITORY: ${IMAGE_REPOSITORY}
        PROXY_USER: ${PROXY_USER}
        PROXY_PASSWORD: ${PROXY_PASSWORD}
        PROXY_FQDN: ${PROXY_FQDN}
        PROXY_PORT: ${PROXY_PORT}
    env_file:
      - .env
    container_name: ${UNIQUE_NAMESPACE}-webui-dev
    privileged: true    
    security_opt:
      - no-new-privileges:true      
    ports:
      - "8081:3001"
    volumes:
      - ./webui:/app
      - /app/node_modules
    environment:
      - CHOKIDAR_USEPULLING=true
    networks:
      - gateway-dev      

  gateway:
    build:
      context: ./gateway
      dockerfile: Dockerfile.dev
      args: # from env_file
        IMAGE_REPOSITORY: ${IMAGE_REPOSITORY}
        PROXY_USER: ${PROXY_USER}
        PROXY_PASSWORD: ${PROXY_PASSWORD}
        PROXY_FQDN: ${PROXY_FQDN}
        PROXY_PORT: ${PROXY_PORT}
    env_file:
      - .env
    container_name: ${UNIQUE_NAMESPACE}-gateway-dev
    depends_on:
      - webui    
    privileged: true    
    security_opt:
      - no-new-privileges:true
    restart: unless-stopped     
    ports:
      - "80:80"
    volumes:
      - ./gateway:/app
      - /app/node_modules
    environment:
      - CHOKIDAR_USEPULLING=true
    networks:
      - default
      - gateway-dev
    external_links:
      - microservices-webui-dev
      - d2iq-management-d2iq-dev

# see https://stackoverflow.com/questions/45255066/create-networks-automatically-in-docker-compose
networks:
  gateway-dev:
    external: true
    name: gateway-dev
```
docker-compose-dev.yml

- In the **webui** service example above (ports: - "8081:3001"), we just make port forwarding from 3001 port of Docker container ("8081:**3001**") to 8081 port of our real server/host ("**8081**:3001").
- In the **gateway** service example above (ports: - "80:80"), we just make port forwarding from 80 port of Docker container ("80:**80**") to 80 port of our real server/host ("**80**:80").

**NOTE**: Take a note to ```restart: unless-stopped```. This automatically start Docker container even after a server reboot.

```microservices-webui-dev``` is the Docker container name of the current container that contains the following services:
- webui (ports: -"8081:3001")
- gateway (ports: -"80:80")

```d2iq-management-d2iq-dev``` is the Docker container name of the neighbouring container that contains the following services:
- d2iq (ports: -"8090:3000")

The solution was straightforward simply, you should use ```proxy_pass``` in your Nginx server block configuration to forward all incoming requests to **Docker port** for those services that share the same Docker container with the gateway (i.e. gateway and webui) and for those services that are part of the neighbouring Docker container(s) (i.e. d2iq). MAKE SURE **NO DOCKER PORTS** ARE THE SAME BETWEEN SERVICES **INSIDE** THE SAME DOCKER CONTAINER AND **NO DOCKER PORTS** ARE THE SAME **BETWEEN** CONTAINERS, THEY SHOULD ALL BE DIFFERENT (e.g. **not** 3000 for d2iq and 3000 for webui, but 3000 for d2iq and 300**1** for webui).

```
# See https://popovserhii.com/how-to-forward-nginx-incoming-request-to-docker-container-port

# http {
#  include       /etc/nginx/mime.types;
#  default_type  application/octet-stream;

  server {
    listen 80;
    # listen  [::]:80; # causes an error when not supporting IPv6
    # server_name microservices.company.com;
    # server_name  localhost; # causes an error when not supporting IPv6
    server_name 127.0.0.1;

    resolver 127.0.0.11;

    add_header Access-Control-Allow-Origin *;

    location / {
      resolver 127.0.0.11 ipv6=off valid=30s;
      set $empty "";
      proxy_set_header Host $http_host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_connect_timeout 300;
      # Default is HTTP/1, keepalive is only enabled in HTTP/1.1
      proxy_http_version 1.1;
      proxy_set_header Connection "";
      chunked_transfer_encoding off;
      #  root   /usr/share/nginx/html;
      #  index  index.html index.htm;
      #  try_files $uri $uri/ /index.html;
      # Re-route to web ui:
      proxy_pass http://microservices-webui-dev:3001$empty;
    }

    location /status {
      stub_status on;
      access_log off;
      allow all;
    }

    location /web {
      resolver 127.0.0.11 ipv6=off valid=30s;
      set $empty "";
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection 'upgrade';    
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_cache_bypass $http_upgrade;        
      proxy_pass http://microservices-webui-dev:3001$empty;
    }

    # For connecting with containers on other docker stacks on the same host, 
    # see https://stackoverflow.com/questions/45255066/create-networks-automatically-in-docker-compose
    # Note that for **external** containers, the path needs to end with a slash (/)!

    location /d2iq/ {
      resolver 127.0.0.11 ipv6=off valid=30s;
      set $empty "";
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_pass http://d2iq-management-d2iq-dev:3000/$empty; # End with a slash
    }

    error_page   500 502 503 504  /50x.html;

    location = /50x.html {
      root   /usr/share/nginx/html;
    }

  }
#}
```
microservices/containers/app/gateway/nginx/nginx.conf.development

## Troubleshooting

- [nginx proxy cannot resolve container hostnames (Version 2 Yaml)](https://github.com/docker/compose/issues/3412)
