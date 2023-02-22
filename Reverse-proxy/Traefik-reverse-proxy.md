# Traefik

### 1. Traefik is a modern reverse-proxy and api gateway particularly excells at routing traffic to microservices

### 2. It  consists of three parts routers, middlewares and services

### 3. Router create a route using domain name which receives incomig requests and forwards it to middlewares

### 4. Middlewares is used to authenticate and alter incoming requests before forwarding it to backend services which is configured by services.

# Docker-compose setup

### 1. Docker-compose file to start traefik reverse proxy. This docker-compose setup requires docker-socket proxy for exposing docker socket to traefik

### 2. Traefik uses labels on (microservices)docker containers and services for creating routes, middlewares and services 

```yaml
# Filename: /opt/traefik/docker-compose.yml

# docker-compose file to start up traefik as a local service
# to start the container run:
#    docker-compose up -d
#
# Note: Bind mounts for config file, acme ssl certs, etc
#
# Note2: docker container name is sourced from the .env file in this directory.
#
# Note3: containrrr/watchtower service will auto update this container


version: "2.4"

services:
  traefik:
    image: traefik:latest
    container_name: traefik.localhost
    restart: always

    mem_limit: 512M

    volumes:
      - ./:/opt/traefik
      - ./:/etc/traefik

    network_mode: host

    labels:
      - com.centurylinklabs.watchtower.enable=true
```

### 3. Traefik.yaml static config file contains all the basic config needed to run traefik

```yaml
# Traefik v2.4.0 main configuration file
# Filename: /opt/traefik/traefik.yaml
# Note: Create /opt/traefik/acme/acme.json file before starting traefik
# File permission: chmod 600 /opt/traefik/acme/acme.json

# Listen on http, https ports. Forward all http to https using middleware.
entryPoints:
  https:
    address: ':443'

  http:
    address: ':80'
    http:
      redirections:
        entryPoint:
          to: https
          scheme: https
  ssh:
    address: ':2222'

  monitoring:
    address: ':8000'


# Use lets-encrypt CA to auto create SSL certificates on demand
certificatesResolvers:
  lets-encrypt:
    acme:
      email: alita@gmail.com
      storage: /opt/traefik/acme/acme.json
      tlsChallenge: {}

# Publish API dashboard, see conf.d/10-app.yaml
api:
  dashboard: false

# Health check endpoint to test this traefik service
ping:
  entryPoint: monitoring


# Export prometheus metrics
metrics:
  prometheus:
    addServicesLabels: true
    addEntryPointsLabels: true
    entryPoint: monitoring

# Web access log location. See /etc/logrotate.d/traefik.
accessLog:
  filePath: /dev/stdout

# Serve applications configured by file and docker providers
# Docker provider uses labels in docker-compose for auto discovery
providers:
  file:
    directory: /opt/traefik/conf.d
    watch: true

  docker:
    exposedByDefault: false
    endpoint: "tcp://localhost:2375"
    watch: true
```

### 4. Traefik uses letsencrypt CA for ssl certificates. It handles ssl termination , certificate autorenewal without any additional configs 

### 5. Dynamic config for allowed tls cipher. The file change requires no traefik restart to start affect

```yaml
# Common dynamic config options for traefik v2.x
# Filename: /opt/traefik/conf.d/00-common.yaml

# Set minimum TLS 1.2 with secure ciphers
tls:
  options:
    default:
      minVersion: VersionTLS12
      cipherSuites:
        - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
        - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
        - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
        - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
        - TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256
        - TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256
```

### 6. Traefik has a build in dashboard for viewing http, tcp and udp routers, services, middlewares 

```yaml
# Traefik v2.4.0 dynamic config fragment to expose the api dasbboard
# Filename: /opt/traefik/conf.d/00-app.yaml
# URL: https://app.goagain.me
# Basic Auth: user/pass: alita/battleangel
# Generate using echo $(htpasswd -nb user password) | sed -e s/\\$/\\$\\$/g

http:
  middlewares:
    apiAuth:
      basicAuth:
        users:
          - 'alita:$$apr1$$aVoaJOmF$$R1SbxEWRLy8yO.xggwFkG/'
  routers:
    api:
      rule: Host(`traefik.smounesh.in`)
      entrypoints:
        - https
      middlewares:
        - apiAuth
      service: api@internal
      tls:
        certResolver: lets-encrypt
```

### 7. Config file for redirect abc.com/* to xyz.com/something  

```yaml
# Traefik v2.4.0 dynamic config to redirect abc.com/* to https://xyz.com/something  
# Filename: /opt/traefik/conf-enabled.d/20-redirect-to-another-domain.yaml
# Note1: Don't use 169.254.169.254 as it is used by oci for hosting metadata about running instances.
# Note2: 169.254.0.0-169.254.255.255 is not routable.
# Reference: https://learn.microsoft.com/en-us/windows-server/troubleshoot/how-to-use-automatic-tcpip-addressing-without-a-dh

http:
  services:
   redirect-to-another-domain:
      loadBalancer:
        servers:
          - url: http://169.254.1.1

  middlewares:
    redirect-to-another-domain:
      redirectRegex:
        regex: "^https://abc.com/(.*)"
        replacement: "https://xyz.com/something"
        permanent: true

  routers:
    redirect-to-another-domain:
      rule: Host(`xyz.com`)
      entrypoints:
        - https
      middlewares:
        - redirect-to-another-domain
      service: redirect-to-another-domain
      tls:
        certResolver: lets-encrypt
```
	
### 8. Config file for proxing traffic to another servers

```yaml
# Traefik v2 dynamic config fragment to expose alita.smounesh.in
# Filename: /opt/traefik/conf.d/30-alita.smounesh.in.yaml
# Middleware: This service is configured with traefik forward auth middleware
# URL: https://alita.smounesh.in

http:
  services:
    alita:
      loadBalancer:
        passHostHeader: true
        servers:
          - url: http://10.0.0.1:80

  routers:
    alita:
      rule: Host(`alita.smounesh.in`)
      entrypoints:
        - http
        - https
      service: alita
      middlewares: google-oauth2@file
      tls:
        certResolver: lets-encrypt
```

