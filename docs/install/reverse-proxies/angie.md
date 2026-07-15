---
sidebar_position: 5
title: Angie
description: Reverse proxy with automatic SSL certificates
---

import PointDomainToIp from '/docs/partials/\_point_domain_to_ip.md';
import OpenLoginPage from '/docs/partials/\_open_login_page.md';

## Overview

In this guide we will be using Angie as a reverse proxy to access the Remnawave panel.
We will point a domain name to our server and configure Angie.

<PointDomainToIp />

### Create a folder for Angie

```bash
mkdir -p /opt/remnawave/angie && cd /opt/remnawave/angie
```

## Angie configuration

### Simple configuration

Create a file called `remnawave.conf` in the `/opt/remnawave/angie/http.d` directory.

```bash
mkdir -p /opt/remnawave/angie/http.d && cd /opt/remnawave/angie/http.d && nano remnawave.conf
```

Paste the following configuration.

:::warning

Please, replace `REPLACE_WITH_YOUR_DOMAIN` with your domain name.

Review the configuration below, look for red highlighted lines.

:::

```angie title="remnawave.conf"
upstream remnawave {
    server remnawave:3000;
}

server {
    // highlight-next-line-red
    server_name REPLACE_WITH_YOUR_DOMAIN;
    
    listen 443 ssl;
    http2 on;
    gzip on;

    location / {
        proxy_http_version 1.1;
        proxy_pass http://remnawave;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # ACME
    acme remnawave;
    ssl_certificate $acme_cert_remnawave;
    ssl_certificate_key $acme_cert_key_remnawave;
}

# Reject unknown SNI
server {
    listen 443 ssl default_server reuseport;
    server_name _;

    ssl_reject_handshake on;
}

# HTTP for ACME
server {
    listen 80;
    return 444; 
}

# Gzip Compression
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_buffers 16 8k;
gzip_http_version 1.1;
gzip_min_length 256;
gzip_types
    application/javascript
    application/json
    application/manifest+json
    application/xml
    font/opentype
    font/eot
    font/otf
    font/ttf
    image/svg+xml
    text/css
    text/javascript
    text/plain
    text/xml;

# SSL 
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
ssl_ecdh_curve X25519MLKEM768:X25519:secp384r1:prime256v1;

ssl_session_timeout 1d;
ssl_session_cache shared:SSL:1m;
ssl_session_tickets off;

resolver 1.1.1.1 1.0.0.1 8.8.8.8 8.8.4.4 208.67.222.222 208.67.220.220;
acme_client remnawave https://acme-v02.api.letsencrypt.org/directory;
```

### Create docker-compose.yml

Create a `docker-compose.yml` file in the `/opt/remnawave/angie` directory.

```bash
cd /opt/remnawave/angie && nano docker-compose.yml
```

Paste the following configuration.

```yaml title="docker-compose.yml"
services:
  remnawave-angie:
    image: docker.angie.software/angie:latest
    container_name: 'remnawave-angie'
    hostname: remnawave-angie
    volumes:
      - angie-ssl-data:/var/lib/angie/acme/
      - ./http.d/:/etc/angie/http.d/:ro
    restart: always
    ports:
      - '0.0.0.0:443:443'
      - '0.0.0.0:80:80'
    networks:
      - remnawave-network  

networks:
  remnawave-network:
    name: remnawave-network
    driver: bridge
    external: true

volumes:
  angie-ssl-data:
    name: angie-ssl-data
    driver: local
    external: false
```

### Start the container

```bash
docker compose up -d && docker compose logs -f -t
```

<OpenLoginPage />
