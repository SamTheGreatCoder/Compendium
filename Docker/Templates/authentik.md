# Authentik

## My version:
NOTE: Backend/frontend networks are for homelab.
## Environment Variables:
```
POSTGRES_DB=
POSTGRES_USER=
POSTGRES_PASSWORD=
AUTHENTIK_SECRET_KEY=
```
``` yaml
services:
  postgresql:
    image: docker.io/library/postgres:12-alpine
    container_name: authentik-postgresql
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 5s
    volumes:
      - /opt/pve/authentik/postgresql/data:/var/lib/postgresql/data
    env_file:
      - stack.env
    environment:
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_DB=${POSTGRES_DB}
    networks:
      - backend
  redis:
    image: docker.io/library/redis:alpine
    container_name: authentik-redis
    command: --save 60 1 --loglevel warning
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping | grep PONG"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 3s
    volumes:
      - /opt/pve/authentik/redis/data:/data
    networks:
      - backend
  server:
    image: ghcr.io/goauthentik/server:2024.4
    container_name: authentik-server
    restart: unless-stopped
    command: server
    env_file:
      - stack.env
    environment:
      - AUTHENTIK_REDIS__HOST=authentik-redis
      - AUTHENTIK_POSTGRESQL__HOST=authentik-postgresql
      - AUTHENTIK_POSTGRESQL__USER=${POSTGRES_USER}
      - AUTHENTIK_POSTGRESQL__NAME=${POSTGRES_DB}
      - AUTHENTIK_POSTGRESQL__PASSWORD=${POSTGRES_PASSWORD}
      - AUTHENTIK_ERROR_REPORTING__ENABLED=true
      - AUTHENTIK_SECRET_KEY=${AUTHENTIK_SECRET_KEY}
    volumes:
      - /opt/pve/authentik/common/media:/media
      - /opt/pve/authentik/common/templates:/templates
    depends_on:
      - postgresql
      - redis
    networks:
      - backend
      - frontend
  worker:
    image: ghcr.io/goauthentik/server:2024.4
    container_name: authentik-worker
    restart: unless-stopped
    command: worker
    env_file:
      - stack.env
    environment:
      - AUTHENTIK_REDIS__HOST=authentik-redis
      - AUTHENTIK_POSTGRESQL__HOST=authentik-postgresql
      - AUTHENTIK_POSTGRESQL__USER=${POSTGRES_USER}
      - AUTHENTIK_POSTGRESQL__NAME=${POSTGRES_DB}
      - AUTHENTIK_POSTGRESQL__PASSWORD=${POSTGRES_PASSWORD}
      - AUTHENTIK_ERROR_REPORTING__ENABLED=true
      - AUTHENTIK_SECRET_KEY=${AUTHENTIK_SECRET_KEY}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /opt/pve/authentik/common/media:/media
      - /opt/pve/authentik/common/certs:/certs
      - /opt/pve/authentik/common/templates:/templates
    depends_on:
      - postgresql
      - redis
    networks:
      - backend
networks:
  frontend:
    external: true
  backend:
    external: true
```

## Based on:
https://goauthentik.io/docker-compose.yml
``` yaml
---

services:
  postgresql:
    image: docker.io/library/postgres:16-alpine
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 5s
    volumes:
      - database:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: ${PG_PASS:?database password required}
      POSTGRES_USER: ${PG_USER:-authentik}
      POSTGRES_DB: ${PG_DB:-authentik}
    env_file:
      - .env
  redis:
    image: docker.io/library/redis:alpine
    command: --save 60 1 --loglevel warning
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping | grep PONG"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 3s
    volumes:
      - redis:/data
  server:
    image: ${AUTHENTIK_IMAGE:-ghcr.io/goauthentik/server}:${AUTHENTIK_TAG:-2024.4.2}
    restart: unless-stopped
    command: server
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER:-authentik}
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB:-authentik}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
    volumes:
      - ./media:/media
      - ./custom-templates:/templates
    env_file:
      - .env
    ports:
      - "${COMPOSE_PORT_HTTP:-9000}:9000"
      - "${COMPOSE_PORT_HTTPS:-9443}:9443"
    depends_on:
      - postgresql
      - redis
  worker:
    image: ${AUTHENTIK_IMAGE:-ghcr.io/goauthentik/server}:${AUTHENTIK_TAG:-2024.4.2}
    restart: unless-stopped
    command: worker
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER:-authentik}
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB:-authentik}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
    # `user: root` and the docker socket volume are optional.
    # See more for the docker socket integration here:
    # https://goauthentik.io/docs/outposts/integrations/docker
    # Removing `user: root` also prevents the worker from fixing the permissions
    # on the mounted folders, so when removing this make sure the folders have the correct UID/GID
    # (1000:1000 by default)
    user: root
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./media:/media
      - ./certs:/certs
      - ./custom-templates:/templates
    env_file:
      - .env
    depends_on:
      - postgresql
      - redis

volumes:
  database:
    driver: local
  redis:
    driver: local

```