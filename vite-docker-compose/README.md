# Docker Compose — Detailed Guide

A practical reference for working with Docker Compose, written against this Vite + Bun/Node project but useful for any stack.

---

## 1. What is Docker Compose?

Docker Compose is a tool for defining and running multi-container applications. You describe your services (containers), networks, and volumes in a single YAML file (`compose.yaml` or `docker-compose.yml`), then manage everything with one command.

**Why use it over plain `docker run`?**

| Plain `docker run` | Docker Compose |
|---|---|
| One container at a time | Many services in one file |
| Long CLI flags | Declarative YAML |
| Hard to reproduce | Version-controlled spec |
| Manual network wiring | Auto-created network |
| No startup ordering | `depends_on`, `healthcheck` |

---

## 2. File Naming & Discovery

Compose looks for, in order:

1. `compose.yaml` (preferred, newer spec)
2. `compose.yml`
3. `docker-compose.yaml`
4. `docker-compose.yml`

Override file: `compose.override.yaml` is merged automatically. Useful for dev-only tweaks.

Explicit file: `docker compose -f prod.yaml up`

---

## 3. Anatomy of `compose.yaml`

```yaml
services:          # containers
  web:
    build: .
    ports:
      - "5173:5173"
    environment:
      NODE_ENV: development
    volumes:
      - .:/app
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:           # named volumes
  db-data:

networks:          # optional custom networks
  backend:

secrets:           # file-based secrets
  db-password:
    file: ./db/password.txt
```

Top-level keys you'll use most: `services`, `volumes`, `networks`, `secrets`.

---

## 4. The `services` Block

### 4.1 `build` vs `image`

```yaml
# Build from local Dockerfile
build:
  context: .
  dockerfile: Dockerfile       # optional, default "Dockerfile"
  args:
    NODE_VERSION: "22.17.1"
  target: dev                   # multi-stage target

# Or use a prebuilt image
image: nginx:1.27-alpine
```

You can combine both: `build` produces the image and `image:` tags it.

### 4.2 `ports`

```yaml
ports:
  - "5173:5173"        # host:container
  - "127.0.0.1:8080:80" # bind to loopback only
  - "3000"             # random host port → 3000
```

`expose:` publishes only to other services on the compose network, not the host.

### 4.3 `environment` and `env_file`

```yaml
environment:
  NODE_ENV: development
  API_URL: http://api:3000      # "api" resolves via compose DNS
env_file:
  - .env                         # key=value per line
```

### 4.4 `volumes` — three forms

```yaml
volumes:
  - .:/app                       # bind mount (host path → container)
  - /app/node_modules            # anonymous volume (shadows host path)
  - db-data:/var/lib/postgres    # named volume (declared at top level)
```

**Why the anonymous-volume trick for `node_modules`?**
When you bind-mount `.` into `/app`, the host's empty or mismatched `node_modules` overwrites what you installed during `docker build`. The anonymous volume on `/app/node_modules` takes precedence and preserves the image's installed modules.

### 4.5 `command` and `entrypoint`

```yaml
entrypoint: ["/bin/sh", "-c"]
command: npm run dev -- --host 0.0.0.0
```

`command` overrides Dockerfile `CMD`. `entrypoint` overrides `ENTRYPOINT`.

### 4.6 `depends_on`

```yaml
depends_on:
  db:
    condition: service_healthy   # wait for healthcheck to pass
  cache:
    condition: service_started   # default — just "running"
```

`depends_on` does **not** wait for the app inside the container to be ready unless you add a `healthcheck` and use `service_healthy`.

### 4.7 `healthcheck`

```yaml
healthcheck:
  test: ["CMD", "pg_isready", "-U", "postgres"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 20s
```

### 4.8 `restart`

```yaml
restart: unless-stopped   # or: no | always | on-failure
```

### 4.9 `user`

```yaml
user: "1000:1000"   # run as uid:gid — fixes host/container perm mismatches
```

### 4.10 `networks`

By default all services join a single network named `<project>_default` and can reach each other by service name (`db`, `web`, etc.). Custom networks:

```yaml
services:
  web:
    networks: [frontend, backend]
  db:
    networks: [backend]

networks:
  frontend:
  backend:
    internal: true   # no outbound internet
```

---

## 5. CLI Commands

```bash
docker compose up                 # start (foreground)
docker compose up -d              # start detached
docker compose up --build         # rebuild images before start
docker compose up --force-recreate  # recreate even if unchanged
docker compose up web             # only start "web" and its deps

docker compose down               # stop + remove containers, networks
docker compose down -v            # also remove named volumes
docker compose down --rmi all     # also remove images

docker compose ps                 # list services
docker compose logs -f web        # follow logs for "web"
docker compose logs --tail=100
docker compose exec web sh        # shell in running container
docker compose run --rm web npm test  # one-off command in new container

docker compose build              # build/rebuild images
docker compose pull               # pull remote images
docker compose restart web
docker compose stop / start
docker compose config             # validate & print merged config
docker compose top                # running processes
docker compose port web 5173      # show mapped host port
docker compose cp ./file web:/app/file
```

`exec` vs `run`:
- `exec` — runs in the **existing** container. Requires it to be up.
- `run` — starts a **new** one-off container with the service's config.

---

## 6. Development Workflow

### 6.1 Hot reload with bind mounts

```yaml
services:
  web:
    build: .
    command: npm run dev -- --host 0.0.0.0
    volumes:
      - .:/app                  # edit files locally, see changes in container
      - /app/node_modules       # keep image's node_modules
    ports:
      - "5173:5173"
    environment:
      NODE_ENV: development
```

Vite/Webpack need `--host 0.0.0.0` to bind to all interfaces, otherwise the port forward from the container is unreachable from the host.

### 6.2 Multi-stage Dockerfile for dev + prod

```dockerfile
FROM node:22-alpine AS base
WORKDIR /app
COPY package*.json ./

FROM base AS dev
RUN npm ci
COPY . .
CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0"]

FROM base AS build
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:1.27-alpine AS prod
COPY --from=build /app/dist /usr/share/nginx/html
```

Select stage in compose:

```yaml
build:
  context: .
  target: dev     # or "prod"
```

### 6.3 Override files for dev vs prod

`compose.yaml` (base):
```yaml
services:
  web:
    image: myapp:latest
    ports: ["5173:5173"]
```

`compose.override.yaml` (auto-merged in dev):
```yaml
services:
  web:
    build:
      context: .
      target: dev
    volumes:
      - .:/app
      - /app/node_modules
```

`compose.prod.yaml` (explicit):
```yaml
services:
  web:
    build:
      target: prod
    restart: unless-stopped
```

Prod run: `docker compose -f compose.yaml -f compose.prod.yaml up -d`

---

## 7. Common Pitfalls (Hit in This Repo!)

### 7.1 `vite: not found` in container

**Cause:** Dockerfile had `npm ci --omit=dev` and/or `NODE_ENV=production`. Both tell npm to skip devDependencies. Vite is a devDependency.
**Fix:** Drop `--omit=dev` and set `NODE_ENV=development` for dev images, or use a multi-stage build where the dev stage installs everything.

### 7.2 `EACCES: permission denied, mkdir '/app/node_modules/.vite/...'`

**Cause:** The image's `WORKDIR` and `npm ci` run as root, so `/app/node_modules` is root-owned. When the Dockerfile then switches to `USER node`, Vite can't write to `.vite/` cache.
**Fix options:**
- `RUN npm ci && chown -R node:node /app/node_modules` before `USER node`.
- Stay as root in the dev image (`node` is only a hardening concern in production).
- Set `user: "1000:1000"` in compose to match your host UID, and `chown -R 1000:1000 /app/node_modules` in the Dockerfile.

**Gotcha:** `docker compose down` keeps anonymous volumes. If you rebuilt the image but the anon volume on `/app/node_modules` still holds old root-owned data, you'll keep hitting EACCES. Use `docker compose down -v` to drop volumes, then `up --build`.

### 7.3 WORKDIR vs bind mount path mismatch

If the Dockerfile has `WORKDIR /usr/src/app` but compose mounts `.:/app`, your source is in `/app` and the container runs in `/usr/src/app`. Symptoms: "file not found", stale code, missing `node_modules`.
**Fix:** Keep paths consistent. Pick one (e.g. `/app`) everywhere.

### 7.4 "Port already allocated"

Something on the host is using that port. Find it:
```bash
lsof -i :5173         # or: ss -ltnp | grep 5173
```
Either kill it or change the host side of the mapping: `"5174:5173"`.

### 7.5 Service can't reach another by name

Inside the compose network, `http://db:5432` works. From the host, use `localhost:<published-port>`. If DNS fails inside a container, check you didn't put the services on different custom networks without overlap.

### 7.6 Changes to `compose.yaml` not applied

`docker compose up` reuses containers whose config hash hasn't changed. When you change volumes/networks/env: `docker compose up --force-recreate` or `down && up`.

### 7.7 Host file permissions after bind mount

Files created by a root-inside-container process will be root-owned on the host. Run the container as your UID:
```yaml
user: "${UID:-1000}:${GID:-1000}"
```
Then `export UID GID` in your shell before `compose up`.

### 7.8 `.env` not loaded

Compose reads `.env` from the project directory for **variable interpolation** (`${VAR}` in compose.yaml). It does **not** automatically pass those vars into the container — use `env_file:` or `environment:` for that.

---

## 8. Volumes — Deeper Look

### 8.1 Bind mount

```yaml
volumes:
  - ./src:/app/src            # relative path on host
  - /etc/localtime:/etc/localtime:ro   # :ro = read-only
```

Good for: source code during dev, config files.
Bad for: database data (slow on Mac/Windows due to filesystem translation).

### 8.2 Named volume

```yaml
services:
  db:
    volumes:
      - db-data:/var/lib/postgresql/data
volumes:
  db-data:
```

Managed by Docker. Survives `down`, removed by `down -v`. Fast on all platforms. Use for databases, caches, persistent state.

Inspect: `docker volume inspect <project>_db-data`

### 8.3 Anonymous volume

```yaml
volumes:
  - /app/node_modules
```

No name. Docker generates one. Used to shadow a bind-mounted path. Removed by `down -v` or `docker volume prune`.

### 8.4 tmpfs

```yaml
tmpfs:
  - /tmp
  - /run
```

In-memory, not persisted. Useful for scratch dirs.

---

## 9. Networking Deep Dive

- Every project gets a default bridge network: `<project-name>_default`.
- Service names are DNS entries on that network.
- Project name = current directory name by default; override with `-p myproj` or `name:` top-level key.
- Exposing a port (`ports:`) publishes to the host. `expose:` is service-to-service only.
- Use `network_mode: host` to skip Docker's network (Linux only) — service shares the host's network stack.

Debug:
```bash
docker compose exec web ping db
docker compose exec web wget -qO- http://api:3000/health
docker network inspect <project>_default
```

---

## 10. Secrets and Sensitive Config

### 10.1 File-based secrets

```yaml
services:
  db:
    secrets:
      - db-password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db-password

secrets:
  db-password:
    file: ./db/password.txt
```

Inside the container the file appears at `/run/secrets/<name>`.

### 10.2 Don't commit `.env`

Add to `.gitignore`. Commit `.env.example` as a template.

---

## 11. Build Caching Tricks

### 11.1 Layer order

Copy dependency manifests and install **before** copying source, so a source change doesn't bust the deps layer:

```dockerfile
COPY package*.json ./
RUN npm ci
COPY . .
```

### 11.2 Cache mounts (BuildKit)

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=cache,target=/root/.npm \
    npm ci
```

Reuses the npm cache across builds without baking it into the image.

### 11.3 Bind mounts in RUN

```dockerfile
RUN --mount=type=bind,source=package.json,target=package.json \
    --mount=type=bind,source=package-lock.json,target=package-lock.json \
    npm ci
```

Avoids an extra `COPY` layer.

### 11.4 `.dockerignore`

Exclude `node_modules`, `.git`, build artifacts, `.env` — faster builds and smaller context.

---

## 12. Profiles (optional services)

```yaml
services:
  web:
    ...
  debug-tools:
    image: alpine
    profiles: ["debug"]
```

Normal: `docker compose up` — skips `debug-tools`.
With profile: `docker compose --profile debug up`.

---

## 13. Resource Limits

```yaml
services:
  web:
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          memory: 128M
```

Note: `deploy.resources` is honored by plain `docker compose up` in recent Compose versions — no Swarm required.

---

## 14. Debugging Checklist

1. `docker compose config` — syntax/merge issues?
2. `docker compose ps` — did anything start?
3. `docker compose logs --tail=50 <svc>` — what's the container saying?
4. `docker compose exec <svc> sh` — poke around inside.
5. Inside: check `ls`, `env`, `cat /etc/resolv.conf`, `ping other-service`, `id`.
6. For permission errors: `ls -la` the path, compare to `id` of the process.
7. For stale state: `docker compose down -v && docker compose up --build`.

---

## 15. Minimal Production-Ready Example

```yaml
services:
  web:
    build:
      context: .
      target: prod
    image: myapp:${TAG:-latest}
    restart: unless-stopped
    ports:
      - "80:80"
    environment:
      NODE_ENV: production
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost/health"]
      interval: 30s
      timeout: 5s
      retries: 3
    deploy:
      resources:
        limits:
          memory: 512M

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD_FILE: /run/secrets/db-password
    secrets:
      - db-password
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  db-data:

secrets:
  db-password:
    file: ./db/password.txt
```

---

## 16. Further Reading

- Compose spec: https://compose-spec.io
- Compose file reference: https://docs.docker.com/compose/compose-file/
- Awesome Compose samples: https://github.com/docker/awesome-compose
- BuildKit Dockerfile: https://docs.docker.com/build/buildkit/
