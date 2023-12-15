# docker-stack-outline-keycloak
### A Docker stack for a selfhosted outline with local storage.<br>
This docker compose will not be using S3 storage or Minio bucket storage. Both storage options add too much complexites and AWS S3 isn't really selfhosted. That being said you should have a backup of your data incase local storage fails. For Authenication we are going to use Keycloak for the OIDC.<br>
**Keycloak** can run using its own container stack, however we are incorperating it into the docker compose stack.<br>

Learn more about Outline from [outline/outline GitHub](https://github.com/outline/outline).
<br><br>

<h2>Changes</h2>
 
 - Seprated network, `backend` for the postgres databases and redis containers. 
 - `frontend` network for containers, `outline-app` and `keycloak-app` to be exposed to internet using a proper proxy. 
 - `container_name` is added to all services, `outline-app`, `outline-redis`, `outline-postgress`, `keycloak-app` and `keycloak-db`. You can choose to rename for your deploy.
 - Changed all doceker volumes to bind-mounts for better backup.
 - added a volume for redis `/data`.
 - added keycloak as the primary auth provider.
 - changed docker.env file to be local storage only.
<br><br>

## Requirements
 - A reverse proxy such as `Nginx Proxy Manager`. You can grab instructions to install one [here](https://github.com/ravencore17/Docker-Stack-Nginx-MariaDB).<br>
 - A domain and subdomains at the ready. I recommend `outline.domain.com` and `keycloak.domain.com`. You can use others but make sure you change the URLS in the compose and .env<br>

## Deployment
1. A) You can clone repo [here](https://github.com/ravencore17/docker-stack-outline-keycloak.git)<br>
 B)  Download the `docker-compose.yml` and `docker.env` file and put them into a folder to run <br>
 C) Copy the following into your own `docker-compose.yml` and the `docker.env` to your own files and folder.<br>

#### docker compose, to download go [here](https://github.com/ravencore17/docker-stack-outline-keycloak/blob/main/docker-compose.yml)

```yaml
version: "3.2"

networks:
  frontend:
    external: true
  backend:

services:
  outline:
    container_name: outline-app
    image: docker.getoutline.com/outlinewiki/outline:latest
    env_file: ./docker.env
    environment:
      - PUID=1000
      - PGID=1000
    ports:
      - "3000:3000"
    volumes:
      - ./outline-data:/var/lib/outline/data
    depends_on:
      - postgres
      - redis
    networks:
      - frontend
      - backend

  redis:
    container_name: outline-redis
    image: redis
    env_file: ./docker.env
    ports:
      - "6379:6379"
    volumes:
      - ./outline-redis.conf:/redis.conf
      - ./outline-redis-data:/data
    command: ["redis-server", "/redis.conf"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 30s
      retries: 3
    networks:
      - backend  

  postgres:
    container_name: outline-postgres
    image: postgres
    env_file: ./docker.env
    ports:
      - "5432:5432"
    volumes:
      - ./outline-database-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready"]
      interval: 30s
      timeout: 20s
      retries: 3
    environment:
      POSTGRES_USER: 'user'
      POSTGRES_PASSWORD: 'pass'
      POSTGRES_DB: 'outline'
    networks:
      - backend  

  keycloak:
    container_name: keycloak-app
    image: quay.io/keycloak/keycloak:latest
    ports:
      - "8080:8080"
      - "8443:8443"
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://keycloak-db:5432/keycloakdb
      KC_DB_USERNAME: keycloakdbuser
      KC_DB_PASSWORD: #change me
      KEYCLOAK_ADMIN: 'admin'
      KEYCLOAK_ADMIN_PASSWORD: #change me to a secure password
      KC_DB_PORT: 5432
      KEYCLOAK_DATABASE_SCHEMA: public
      KEYCLOAK_ENABLE_HEALTH_ENDPOINTS: 'true'
      KEYCLOAK_ENABLE_STATISTICS: 'true'
      KC_HOSTNAME: 'keycloak.domain.com'
      KC_PROXY: edge
      KC_PROXY_ADDRESS_FORWARDING: 'true'
      KC_HTTP_ENABLED: 'true'
    depends_on:
      - key-postgres
    networks:
      - frontend
      - backend
    command: start

  key-postgres:
    image: postgres:15
    container_name: keycloak-db
    volumes:
      - ./keycloak-db:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: 'keycloakdb'
      POSTGRES_USER: 'keycloakdbuser'
      POSTGRES_PASSWORD: #same as KC_DB_PASSWORD
    networks:
      - backend
    healthcheck:
      test: [ "CMD", "pg_isready", "-q", "-d", "keycloakdb", "-U", "keycloakdbuser" ]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 60s
    restart: unless-stopped
```
#### docker.env condensed, non-condensed version with more options [here](https://github.com/ravencore17/docker-stack-outline-keycloak/blob/main/docker.env)
```yaml
# –––––––––––––––– REQUIRED ––––––––––––––––
NODE_ENV=production
# Generate a hex-encoded 32-byte random key. You should use `openssl rand -hex 32`
# in your terminal to generate a random value.
SECRET_KEY=
# Generate a unique random key. The format is not important but you could still use
# `openssl rand -hex 32` in your terminal to produce this.
UTILS_SECRET=

DATABASE_URL=postgres://user:pass@outline-postgres:5432/outline
DATABASE_URL_TEST=postgres://user:pass@outline-postgres:5432/outline-test
DATABASE_CONNECTION_POOL_MIN=
DATABASE_CONNECTION_POOL_MAX=
PGSSLMODE=disable
REDIS_URL=redis://outline-redis:6379

URL=https://outline.domain.com
PORT=3000

FILE_STORAGE=local
FILE_STORAGE_LOCAL_ROOT_DIR=/var/lib/outline/data
FILE_STORAGE_UPLOAD_MAX_SIZE=26214400

# –––––––––––––– AUTHENTICATION ––––––––––––––

# To configure generic OIDC auth, you'll need some kind of identity provider.
# See documentation for whichever IdP you use to acquire the following info:
# Redirect URI is https://<URL>/auth/oidc.callback
OIDC_CLIENT_ID=outline
OIDC_CLIENT_SECRET=
OIDC_AUTH_URI=https://keycloak.domain.com/realms/outline/protocol/openid-connect/auth
OIDC_TOKEN_URI=https://keycloak.domain.com/realms/outline/protocol/openid-connect/token
OIDC_USERINFO_URI=https://keycloak.domain.com/realms/outline/protocol/openid-connect/userinfo

OIDC_USERNAME_CLAIM=email
OIDC_DISPLAY_NAME=KeyCloak
OIDC_SCOPES=openid profile email


# –––––––––––––––– OPTIONAL ––––––––––––––––

FORCE_HTTPS=true
ENABLE_UPDATES=false
WEB_CONCURRENCY=1
MAXIMUM_IMPORT_SIZE=5120000
# Configure lowest severity level for server logs. Should be one of
# error, warn, info, http, verbose, debug and silly
LOG_LEVEL=info

# To support sending outgoing transactional emails such as "document updated" or
# "you've been invited" you'll need to provide authentication for an SMTP server
SMTP_HOST=
SMTP_PORT=
SMTP_USERNAME=
SMTP_PASSWORD=
SMTP_FROM_EMAIL=hello@example.com
SMTP_REPLY_EMAIL=hello@example.com
SMTP_TLS_CIPHERS=
SMTP_SECURE=true

DEFAULT_LANGUAGE=en_US

RATE_LIMITER_ENABLED=true

RATE_LIMITER_REQUESTS=1000
RATE_LIMITER_DURATION_WINDOW=60

DEVELOPMENT_UNSAFE_INLINE_CSP=false
```


2. After you have cloned or nested the `docker-compose.yml` and `docker.env` into a folder. We will need to edit the two files before running them. <br><br>
3. Change the following `KC_DB_PASSWORD` and `KEYCLOAK_ADMIN_PASSWORD` to secure passwords. Make sure they are **different**, you can use a terminal command `openssl rand -hex 12`
```yaml
      KC_DB_USERNAME: keycloakdbuser
      KC_DB_PASSWORD: #change me
      KEYCLOAK_ADMIN: 'admin'
      KEYCLOAK_ADMIN_PASSWORD: #change me to a secure password
```
<br>

5. Change the `POSTGRES_PASSWORD` tothe same password you set for `KC_DB_PASSWORD`
```yaml
      POSTGRES_USER: 'keycloakdbuser'
      POSTGRES_PASSWORD: #same as KC_DB_PASSWORD
```
6. Connect to the keycloak container by either going to `keycloak.yourdomain.com` or by connecting to the port `8080` at you LAN ip for admin controls. ex. `127.0.0.1:8080`
7. the login is:
```
Email:      admin
Password:   [KEYCLOAK_ADMIN_PASSWORD]
```
<br>

8. After the login screen, you seel the `Weclome to KeyCloak` panel. ``
<p align="center"><img width=596px src=./images/admin-1.png></p>

## Notes 
- Change the `DB_MYSQL_PASSWORD=` to a secure password
- Change the `MYSQL_ROOT_PASSWORD=` to a secure password
- Create the external network `frontend` before deployment otherwise it will not find the network and docker will error out.
- For more documentation go to the [Nginx Proxy Manager Docs](https://nginxproxymanager.com/guide/)
