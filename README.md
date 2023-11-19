# Homelab vaulwarden testing

## Docker

Install docker + portainer

## Network stack (can be replace by traefik)

#### Mkcert

1. Install mkcert (here ubuntu): 

```bash
apt install libnss3-tools mkcert
```

2. Generate certificate:

```bash
mkdir ~/cert
cd ~/cert
# Create new local CA in ~/.local/share/mkcert/ dir
mkcert -install
# Create domain certif (here wildcard) in local dir
mkcert *.home.arpa
```

1. Add CA cert to your browser

Example for firefox: 

Prefences -> Privacy & Security -> Certificates -> View Certificates -> Authorities (Tab) -> Import (File to import => ~/.local/share/mkcert/rootCA.pem)

#### Coredns

create Corefile file

```text
(snip) {
  log
  errors
}

home.arpa {
  hosts {
    192.168.64.10 portainer.home.arpa
    192.168.64.10 vault.home.arpa
    192.168.64.10 email.home.arpa
    192.168.64.10 pgadmin.home.arpa
    fallthrough
  }
  import snip
}

.:53 {
  whoami
  health
  cache
  forward . /etc/resolv.conf
  import snip
}
```

#### Caddy

create Caddyfile

```text
portainer.home.arpa {
    tls /usr/local/etc/caddy/certs/_wildcard.home.arpa.pem /usr/local/etc/caddy/certs/_wildcard.home.arpa-key.pem
    encode gzip zstd
    log
    reverse_proxy 192.168.64.10:9000 {
        header_up Host                {host}
        header_up X-Real-IP           {remote}
        header_up X-Forwarded-Host    {host}
        header_up X-Forwarded-Server  {host}
        header_up X-Forwarded-Port    {port}
        header_up X-Forwarded-For     {remote}
        header_up X-Forwarded-Scheme  {scheme}
    }
}

pgadmin.home.arpa {
    tls /usr/local/etc/caddy/certs/_wildcard.home.arpa.pem /usr/local/etc/caddy/certs/_wildcard.home.arpa-key.pem
    encode gzip zstd
    log
    reverse_proxy 192.168.64.10:8088 {
        header_up Host                {host}
        header_up X-Real-IP           {remote}
        header_up X-Forwarded-Host    {host}
        header_up X-Forwarded-Server  {host}
        header_up X-Forwarded-Port    {port}
        header_up X-Forwarded-For     {remote}
        header_up X-Forwarded-Scheme  {scheme}
    }
}

email.home.arpa {
    tls /usr/local/etc/caddy/certs/_wildcard.home.arpa.pem /usr/local/etc/caddy/certs/_wildcard.home.arpa-key.pem
    encode gzip zstd
    log
    reverse_proxy 192.168.64.10:8025 {
        header_up Host                {host}
        header_up X-Real-IP           {remote}
        header_up X-Forwarded-Host    {host}
        header_up X-Forwarded-Server  {host}
        header_up X-Forwarded-Port    {port}
        header_up X-Forwarded-For     {remote}
        header_up X-Forwarded-Scheme  {scheme}
    }
}

vault.home.arpa {
    tls /usr/local/etc/caddy/certs/_wildcard.home.arpa.pem /usr/local/etc/caddy/certs/_wildcard.home.arpa-key.pem
    encode gzip zstd
    log
    reverse_proxy 192.168.64.10:8080 {
        header_up Host                {host}
        header_up X-Real-IP           {remote}
        header_up X-Forwarded-Host    {host}
        header_up X-Forwarded-Server  {host}
        header_up X-Forwarded-Port    {port}
        header_up X-Forwarded-For     {remote}
        header_up X-Forwarded-Scheme  {scheme}
    }
}
```

#### Docker-compose

```text
version: '3.9'

services:
  coredns:
    # Need add docker host IP to your host DNS
    image: coredns/coredns:1.11.1
    container_name: coredns
    command: -conf /etc/coredns/Corefile
    volumes:
      - /home/ubuntu/docker/Corefile:/etc/coredns/Corefile:ro
    ports:
      - "53:53/udp"
    logging:
      options:
        max-size: "10m"
        max-file: "5"
    restart: always
    networks:
      - homelab

  caddy:
    image: caddy:2.7.5-alpine
    container_name: caddy
    cap_add:
      - NET_ADMIN
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - /home/ubuntu/docker/Caddyfile:/etc/caddy/Caddyfile:ro
      - /home/ubuntu/cert:/usr/local/etc/caddy/certs/
      - caddy_data:/data
      - caddy_config:/config
    logging:
      options:
        max-size: "10m"
        max-file: "5"
    restart: always
    networks:
      - homelab
    depends_on:
      - coredns

volumes:
  caddy_data:
  caddy_config:

networks:
  homelab:
```

## Vaultwarden

#### 

:warning: Change the secret before running the below command ! :warning:

```bash
echo -n 'MySuperSecret' | argon2 "$(openssl rand -base64 32)" -e -id -k 65540 -t 3 -p 4
```

#### Docker-compose

:warning: Please read the file below and change the passwords/tokens before using it ! :warning:

```text
version: '3.9'

services:
  postgres:
    image: postgres:16.1-alpine3.18
    container_name: postgres
    environment:
      # https://www.postgresql.org/docs/16/libpq-envars.
      - PGUSER=postgres
      - DB_SERVER_HOST=postgres
      - DB_SERVER_PORT=5432
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=vaultwarden
      - PGDATA=/var/lib/postgresql/data
      - PGTZ=Europe/Paris
    volumes:
      - postgres:/var/lib/postgresql/data
    deploy:
      resources:
        limits:
          cpus: '0.70'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: on-failure
    networks:
      - vault

  pgadmin:
    image: dpage/pgadmin4:7.8
    container_name: pgadmin
    environment:
      - PGADMIN_DEFAULT_EMAIL=admin@admin.com
      - PGADMIN_DEFAULT_PASSWORD=password
      - PGADMIN_DISABLE_POSTFIX=true
    volumes:
      - pgadmin:/var/lib/pgadmin
    ports:
      - "8088:80"
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - vault

  mailpit:
    image: axllent/mailpit:v1.10.1
    container_name: mailpit
    environment:
      # https://mailpit.axllent.org/docs/configuration/runtime-options/
      - MP_DATA_FILE=/data/mailpit.db
      - MP_UI_AUTH=admin:password
      - TZ=Europe/Paris
    volumes:
      - mailpit:/data
    ports:
      - 8025:8025
      - 1025:1025
    restart: unless-stopped
    networks:
      - vault

  vaultwarden:
    image: vaultwarden/server:1.30.0-alpine
    container_name: vaultwarden
    ports:
      - 8080:80
    environment:
      # https://github.com/dani-garcia/vaultwarden/blob/main/.env.template
      - DOMAIN=https://vault.home.arpa
      - DATA_FOLDER=/data
      - DATABASE_URL=postgresql://postgres:password@postgres:5432/vaultwarden
      - DATABASE_MAX_CONNS=5
      - DATABASE_TIMEOUT=15
      - WEB_VAULT_FOLDER=/web-vault
      - WEB_VAULT_ENABLED=true
      - LOG_LEVEL=Info # "trace", "debug", "info", "warn", "error" and "off"
      - ADMIN_TOKEN=$VAULT_TOKEN
      - SMTP_HOST=mailpit
      - SMTP_FROM=vault@home.arpa
      - SMTP_FROM_NAME=Vaultwarden
      - SMTP_SECURITY=off # "starttls", "force_tls", "off"
      - SMTP_PORT=1025 # Ports 587 (submission) and 25 (smtp) are standard without encryption and with encryption via STARTTLS (Explicit TLS). Port 465 (submissions) is used for encrypted submission (Implicit TLS).
    volumes:
      - vaultwarden-data:/data
      - vaultwarden-web:/web-vault
    restart: unless-stopped
    networks:
      - vault
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  vaultwarden-data:
  vaultwarden-web:
  postgres:
  pgadmin:
  mailpit:

networks:
  vault:
```

Add `VAULT_TOKEN` as environment variable

#### Test

Open vault.home.arpa in your browser
