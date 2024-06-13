# Traefik, WordPress, and Keycloak Setup with Docker Compose

This repository contains Docker Compose files to set up Traefik as a reverse proxy, along with WordPress and Keycloak services.

## Prerequisites

- Docker
- Docker Compose

## Overview

- **Traefik**: A reverse proxy and load balancer that routes traffic to different services based on defined rules.
- **WordPress**: A content management system (CMS) used to create websites and blogs.
- **Keycloak**: An open-source identity and access management solution.

## Directory Structure

```plaintext
.
├── traefik
│   ├── conf
│   │   └── traefik.toml
│   └── docker-compose.yml
├── wordpress
│   └── docker-compose.yml
└── keycloak
    └── docker-compose.yml
```

## Traefik Setup

### `traefik/docker-compose.yml`

```yaml
version: '3.8'
services:
  traefik:
    image: traefik:v2.4
    ports:
      - "80:80"
      - "443:443"
    labels:
      - "traefik.enable=true"

      - "traefik.http.routers.http-catchall.rule=hostregexp(`{host:[a-z-.]+}`)"
      - "traefik.http.routers.http-catchall.entrypoints=web"
      - "traefik.http.routers.http-catchall.middlewares=redirect-to-https"
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"

      - "traefik.http.routers.api.tls=true"
      - "traefik.http.routers.api.entrypoints=websecure"
      - "traefik.http.routers.api.tls.certresolver=basic"

      - "traefik.http.routers.api.rule=Host(`traefik.domain.de`)"
      - "traefik.http.routers.api.service=api@internal"
      - "traefik.http.routers.api.middlewares=auth"
      - "traefik.http.middlewares.auth.basicauth.users=test:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "letsencrypt:/letsencrypt"
      - "./conf/traefik.toml:/traefik.toml"
    networks:
      - traefik

volumes:
  letsencrypt:

networks:
  traefik:
    external: true
```

#### Explanation of Traefik Configuration

- **Image**: Specifies the Traefik image to use.
- **Ports**: Exposes ports 80 and 443 for HTTP and HTTPS traffic.
- **Labels**: Configures various Traefik settings using Docker labels:
  - `traefik.enable=true`: Enables Traefik for this service.
  - `traefik.http.routers.http-catchall.rule=hostregexp({host:[a-z-.]+})`: Catches all HTTP traffic with the specified host pattern.
  - `traefik.http.routers.http-catchall.entrypoints=web`: Uses the `web` entry point for HTTP traffic.
  - `traefik.http.routers.http-catchall.middlewares=redirect-to-https`: Redirects HTTP traffic to HTTPS.
  - `traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https`: Defines the middleware for redirecting to HTTPS.
  - `traefik.http.routers.api.tls=true`: Enables TLS for the Traefik dashboard.
  - `traefik.http.routers.api.entrypoints=websecure`: Uses the `websecure` entry point for HTTPS traffic.
  - `traefik.http.routers.api.tls.certresolver=basic`: Uses the `basic` certificate resolver for TLS.
  - `traefik.http.routers.api.rule=Host(traefik.domain.de)`: Routes traffic to the Traefik dashboard based on the specified host.
  - `traefik.http.routers.api.service=api@internal`: Uses the internal API service for the dashboard.
  - `traefik.http.routers.api.middlewares=auth`: Applies the `auth` middleware for basic authentication.
  - `traefik.http.middlewares.auth.basicauth.users=test:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/`: Configures basic authentication with a user and hashed password.

- **Volumes**: 
  - `/var/run/docker.sock:/var/run/docker.sock:ro`: Mounts the Docker socket to allow Traefik to interact with Docker.
  - `letsencrypt:/letsencrypt`: Stores Let's Encrypt certificates.
  - `./conf/traefik.toml:/traefik.toml`: Mounts the Traefik configuration file.

### `traefik/conf/traefik.toml`

```toml
[providers.docker]

[api]
  insecure = false
  dashboard = true

[entryPoints]
  [entryPoints.web]
    address = ":80"
  [entryPoints.websecure]
    address = ":443"
    [entryPoints.websecure.http.tls]
      certResolver = "basic"

[certificatesResolvers.basic.acme]
  email = "email@domain.de"
  storage = "/letsencrypt/acme.json"
  [certificatesResolvers.basic.acme.httpchallenge]
    entrypoint = "web"
```

#### Explanation of `traefik.toml`

- **[providers.docker]**: Enables Docker as a provider for Traefik, allowing it to discover services automatically.
- **[api]**: Configures the Traefik API and dashboard.
  - `insecure = false`: Disables the insecure API.
  - `dashboard = true`: Enables the Traefik dashboard.

- **[entryPoints]**: Defines entry points for Traefik.
  - **[entryPoints.web]**: Configures the HTTP entry point.
    - `address = ":80"`: Listens on port 80 for HTTP traffic.
  - **[entryPoints.websecure]**: Configures the HTTPS entry point.
    - `address = ":443"`: Listens on port 443 for HTTPS traffic.
    - `[entryPoints.websecure.http.tls]`: Enables TLS for the HTTPS entry point.
      - `certResolver = "basic"`: Uses the `basic` certificate resolver for TLS.

- **[certificatesResolvers.basic.acme]**: Configures the ACME (Automatic Certificate Management Environment) provider.
  - `email = "email@domain.de"`: Email address used for registration with Let's Encrypt.
  - `storage = "/letsencrypt/acme.json"`: Path to store the ACME account keys and certificates.
  - `[certificatesResolvers.basic.acme.httpchallenge]`: Configures the HTTP-01 challenge for certificate validation.
    - `entrypoint = "web"`: Uses the HTTP entry point for the challenge.

## WordPress Setup

### `wordpress/docker-compose.yml`

```yaml
version: '3.8'

services:
  wordpress:
    image: wordpress
    restart: always
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: password
      WORDPRESS_DB_NAME: wordpressdb
    volumes:
      - wordpress:/var/www/html
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.wordpress.rule=Host(`domain.de`)"
      - "traefik.http.routers.wordpress.entrypoints=websecure"
      - "traefik.http.routers.wordpress.tls.certresolver=basic"
    networks:
      - traefik
  
  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: wordpressdb
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: password
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db:/var/lib/mysql

volumes:
  wordpress:
  db:

networks:
  traefik:
    external: true
```

## Keycloak Setup

### `keycloak/docker-compose.yml`

```yaml
version: '3'

services:
  keycloak:
    image: quay.io/keycloak/keycloak:legacy
    container_name: keycloak
    environment:
      DB_VENDOR: MYSQL
      DB_ADDR: keycloak-db
      DB_DATABASE: keycloak
      DB_USER: keycloak
      DB_PASSWORD: password
      KEYCLOAK_USER: admin
      KEYCLOAK_PASSWORD: Pa55word
      PROXY_ADDRESS_FORWARDING: true
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=traefik"
      - "traefik.http.routers.keycloak.rule=Host(`keycloak.domain.de`)"
      - "traefik.http.routers.keycloak.tls=true"
      - "traefik.http.routers.keycloak.tls.certresolver=basic"
      - "traefik.http.routers.keycloak.entrypoints=websecure"
      - "traefik.http.services.keycloak.loadbalancer.server.port=8080"
    networks:
      - keycloak
      - traefik
    depends_on:
      - keycloak-db

  keycloak-db:
    image: mysql:latest
    container_name: keycloak-db
    environment:
      - MYSQL_DATABASE=keycloak
     

 - MYSQL_USER=keycloak
      - MYSQL_PASSWORD=password
      - MYSQL_ROOT_PASSWORD=root
    volumes:
      - keycloak_data:/var/lib/mysql
    networks:
      - keycloak

volumes:
  keycloak_data:

networks:
  traefik:
    external: true
  keycloak:
```

## Usage

1. Clone this repository.
2. Ensure Docker and Docker Compose are installed on your machine.
3. Navigate to the `traefik` directory and start Traefik:
   ```sh
   docker-compose up -d
   ```
4. Navigate to the `wordpress` directory and start WordPress:
   ```sh
   docker-compose up -d
   ```
5. Navigate to the `keycloak` directory and start Keycloak:
   ```sh
   docker-compose up -d
   ```

## Access

- **Traefik Dashboard**: [https://traefik.domain.de](https://traefik.domain.de) (Basic Auth credentials: `test / <password>`)
- **WordPress**: [https://domain.de](https://domain.de)
- **Keycloak**: [https://keycloak.domain.de](https://keycloak.domain.de)

## Notes

- Replace `domain.de` with your actual domain.
- Ensure DNS records are correctly configured to point to your server.
- Certificates will be automatically managed by Let's Encrypt.
