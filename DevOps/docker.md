# Docker — Interview Fundamentals

## What is Docker?
Docker is an open platform for developing, shipping, and running applications in lightweight, portable **containers**. Containers package code and dependencies together, ensuring consistent behaviour across environments.

---

## Core Architecture

```
Docker Client  ──(REST API)──►  Docker Daemon (dockerd)
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                   ▼
               Images            Containers           Volumes
               (read-only        (running             (persistent
               layers)           instances)           storage)
```

### Key Components
| Component | Description |
|-----------|-------------|
| **Image** | Read-only template of layers; built from Dockerfile |
| **Container** | Runnable instance of an image |
| **Registry** | Storage for images (Docker Hub, ECR, Harbor) |
| **Volume** | Persistent, managed storage outside the container |
| **Network** | Virtual network for container communication |
| **Docker Compose** | Multi-container app definition via YAML |

---

## Images

### Dockerfile
```dockerfile
# Multi-stage build — keeps final image small
FROM python:3.11-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# --- Final stage ---
FROM python:3.11-slim

# Non-root user (security best practice)
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app

# Copy only what's needed from builder
COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appgroup . .

ENV PATH=/home/appuser/.local/bin:$PATH \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

ENTRYPOINT ["python"]
CMD ["-m", "gunicorn", "app:app"]
```

### Dockerfile Instructions
| Instruction | Purpose |
|-------------|---------|
| `FROM` | Base image |
| `RUN` | Execute command during build |
| `COPY` | Copy files from build context |
| `ADD` | Like COPY but supports URLs and tar extraction |
| `ENV` | Set environment variables |
| `ARG` | Build-time variables |
| `EXPOSE` | Document port (does not publish) |
| `WORKDIR` | Set working directory |
| `USER` | Set runtime user |
| `ENTRYPOINT` | Fixed executable |
| `CMD` | Default arguments to ENTRYPOINT |
| `VOLUME` | Declare mount point |
| `HEALTHCHECK` | Health check command |
| `LABEL` | Metadata key-value pairs |

### Image Commands
```bash
docker build -t myapp:1.0 .
docker build -t myapp:1.0 -f Dockerfile.prod .
docker build --no-cache -t myapp:latest .
docker build --build-arg VERSION=2.0 -t myapp .

docker images
docker image ls
docker image inspect myapp:1.0
docker image history myapp:1.0      # show layers

docker pull nginx:1.25
docker push myregistry.example.com/myapp:1.0
docker tag myapp:1.0 myregistry.example.com/myapp:1.0

docker rmi myapp:1.0
docker image prune -a               # remove all unused images
```

---

## Containers

### Run & Manage
```bash
docker run -d \
  --name myapp \
  -p 8080:8000 \
  -e DB_HOST=db \
  -e DB_PASSWORD="$(cat /run/secrets/db_pass)" \
  -v /data/myapp:/app/data \
  --network app-net \
  --restart unless-stopped \
  --memory 512m \
  --cpus 1.5 \
  myapp:1.0

docker ps                           # running containers
docker ps -a                        # all containers
docker stop myapp
docker start myapp
docker restart myapp
docker rm myapp
docker rm -f myapp                  # force remove running

docker exec -it myapp bash          # open interactive shell
docker exec myapp cat /etc/hosts    # run one-off command
docker logs myapp -f                # follow logs
docker logs myapp --since 1h
docker logs myapp --tail 100

docker inspect myapp
docker stats                        # live resource usage
docker top myapp                    # processes inside container
docker cp myapp:/app/config.yml .   # copy file out
docker cp local.conf myapp:/etc/    # copy file in
```

### Container Lifecycle
```
created → running → paused → running → stopped → removed
                       ↑                   ↑
                  docker pause         docker stop
```

---

## Networking

### Network Types
| Driver | Use Case |
|--------|---------|
| `bridge` | Default; containers on same host communicate |
| `host` | Container shares host network stack |
| `none` | No networking |
| `overlay` | Multi-host (Docker Swarm / Kubernetes) |
| `macvlan` | Assign MAC address; appear as physical device |

```bash
docker network create app-net
docker network create --driver bridge --subnet 172.20.0.0/16 app-net
docker network ls
docker network inspect app-net
docker network connect app-net myapp
docker network disconnect app-net myapp
```

### DNS in Docker
Containers on the same user-defined bridge network can resolve each other by **container name**. The default bridge network does NOT support DNS — use custom bridge networks.

---

## Volumes & Storage

```bash
# Named volumes (Docker manages location)
docker volume create mydata
docker run -v mydata:/app/data myapp

# Bind mounts (host path)
docker run -v /host/path:/container/path myapp

# tmpfs (in-memory, not persisted)
docker run --tmpfs /tmp myapp

docker volume ls
docker volume inspect mydata
docker volume rm mydata
docker volume prune               # remove unused volumes
```

### Choosing Storage Type
| Type | Persistence | Performance | Use Case |
|------|-------------|-------------|---------|
| Volume | Yes | Best | Production data |
| Bind mount | Yes | Good | Dev (live reload) |
| tmpfs | No | Fastest | Secrets, temp files |

---

## Docker Compose

```yaml
# docker-compose.yml
version: '3.9'

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        APP_VERSION: "2.1"
    image: myapp:2.1
    container_name: myapp-web
    ports:
      - "8080:8000"
    environment:
      - DB_HOST=db
      - REDIS_URL=redis://cache:6379
    env_file:
      - .env.production
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - app-data:/app/data
    networks:
      - frontend
      - backend
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1.0'

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_pass
      MYSQL_DATABASE: appdb
    volumes:
      - db-data:/var/lib/mysql
    networks:
      - backend
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  cache:
    image: redis:7-alpine
    networks:
      - backend

volumes:
  app-data:
  db-data:

networks:
  frontend:
  backend:
    internal: true             # no external access
```

### Compose Commands
```bash
docker compose up -d                        # start all services
docker compose up -d --build                # rebuild images
docker compose down                         # stop and remove containers
docker compose down -v                      # also remove volumes
docker compose ps
docker compose logs -f web
docker compose exec web bash
docker compose scale web=3                  # scale service
docker compose config                       # validate & view config
docker compose pull                         # pull latest images
```

---

## Security Best Practices

1. **Never run as root** — use `USER` instruction and `--user` flag
2. **Use official/minimal base images** — `alpine`, `distroless`, `slim`
3. **Multi-stage builds** — keep build tools out of final image
4. **Scan images** — `docker scout`, `trivy`, `snyk`
5. **Read-only filesystem** — `--read-only` + tmpfs for writable dirs
6. **Drop capabilities** — `--cap-drop ALL --cap-add NET_BIND_SERVICE`
7. **No sensitive data in layers** — use secrets, not ENV
8. **Limit resources** — `--memory`, `--cpus`
9. **Use `.dockerignore`** — prevent leaking `.git`, credentials, etc.
10. **Pin image versions** — avoid `:latest` in production

### Docker Secrets (Swarm)
```bash
echo "mypassword" | docker secret create db_password -
docker service create --secret db_password myapp
# Mounted at /run/secrets/db_password inside container
```

---

## Image Layer Optimization

```dockerfile
# BAD: each RUN is a layer; also apt cache stays in image
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# GOOD: combined, cleaned in one layer
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

# Order COPY to maximize cache reuse
COPY requirements.txt .          # changes rarely → cache hit
RUN pip install -r requirements.txt
COPY . .                         # changes often → only this layer invalidated
```

---

## Common Interview Questions

### Q1: What is the difference between a container and a VM?
| | Container | VM |
|--|-----------|---|
| Isolation | Process-level (namespaces + cgroups) | Full OS isolation (hypervisor) |
| OS | Shares host kernel | Own OS kernel |
| Size | MBs | GBs |
| Startup | Milliseconds | Minutes |
| Overhead | Minimal | Significant |

### Q2: What are Docker namespaces?
Linux namespaces provide isolation for containers:
- `pid` — process isolation
- `net` — network isolation
- `mnt` — filesystem mount isolation
- `uts` — hostname isolation
- `ipc` — IPC isolation
- `user` — user/group isolation

### Q3: What are cgroups?
Control groups limit and monitor resource usage (CPU, memory, disk I/O, network) for groups of processes. Docker uses cgroups to enforce `--memory` and `--cpus` limits.

### Q4: ENTRYPOINT vs CMD?
- `ENTRYPOINT`: The fixed executable — always runs; cannot be overridden with docker run args
- `CMD`: Default arguments to ENTRYPOINT (or command if no ENTRYPOINT); easily overridden
- Combined: `ENTRYPOINT ["python"]` + `CMD ["app.py"]` → `docker run myapp manage.py` runs `python manage.py`

### Q5: How do layers work?
Each Dockerfile instruction creates an immutable read-only layer. Layers are cached and shared across images. Containers add a thin writable layer on top. Changes to files in lower layers use copy-on-write (CoW).

### Q6: How do you reduce image size?
- Use minimal base images (alpine, distroless)
- Multi-stage builds to exclude build deps
- Combine RUN commands to reduce layers
- Use `.dockerignore`
- Remove caches and temp files in the same RUN

### Q7: What is a Docker registry?
A service storing Docker images. Public: Docker Hub, Quay.io. Private: AWS ECR, GCR, Harbor. `docker push/pull` communicate with registries.

### Q8: How do containers communicate?
- Same Docker network: by container name (DNS)
- Expose + port mapping: `host:container` via `-p`
- Host network: direct host networking (`--network host`)
- Linked env vars (legacy): avoid in new deployments

### Q9: What happens to data when a container is removed?
Data in the container's writable layer is lost. Only data stored in **volumes** or **bind mounts** persists. This is why stateful services should always use volumes.

### Q10: What is a Docker health check?
A command run periodically inside the container to report its status (`healthy`, `unhealthy`, `starting`). Used by orchestrators (Compose, Swarm, Kubernetes) to determine if the container is ready to receive traffic.
