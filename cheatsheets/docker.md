# Docker cheatsheet (annotated)

[← Cheatsheet index](README.md)

Format: command → what it does and when you reach for it.

## Images

| Command | What it does / when to use |
|---|---|
| `docker build -t app:1.2.3 .` | Builds an image from the Dockerfile in `.` and tags it. Always tag with a version or git SHA, never rely on `latest`. |
| `docker build -f Dockerfile.dev --target build -t app:dev .` | Use an alternate Dockerfile and stop at a named multi-stage target. Handy to get a build-stage image with tools for debugging. |
| `docker build --platform linux/amd64 -t app .` | Build for a specific CPU architecture. Required on Apple Silicon Macs when the image will run on x86 nodes (otherwise `exec format error`). |
| `docker build --no-cache .` | Ignore layer cache. Use when a `RUN apt-get update` layer is stale or you suspect cache poisoning. |
| `docker build --build-arg VERSION=1.2 .` | Pass a value to an `ARG` in the Dockerfile. Build-time only; does not persist in the image unless copied into `ENV`. |
| `docker build --secret id=npmrc,src=$HOME/.npmrc .` | Mount a secret during build without baking it into a layer (`RUN --mount=type=secret,id=npmrc`). Use for private registry tokens. |
| `docker buildx create --use` | Create a BuildKit builder that can target multiple platforms. One-time setup. |
| `docker buildx build --platform linux/amd64,linux/arm64 -t repo/app:1.2 --push .` | Multi-arch image pushed as one manifest list. Needed for Graviton + x86 fleets. |
| `docker images` / `docker image ls --filter dangling=true` | List images; dangling = untagged leftovers from rebuilds. |
| `docker image prune -a` | Remove all unused images. Frees disk on CI runners and laptops. |
| `docker tag app:1.2.3 123.dkr.ecr.ap-south-1.amazonaws.com/app:1.2.3` | Add a registry-qualified tag so you can push to ECR. |
| `docker push REPO:TAG` / `docker pull REPO:TAG` | Upload/download. |
| `docker pull nginx@sha256:abc...` | Pull by digest: immutable, exactly the bits you tested. Use digests in prod manifests. |
| `docker history app:1.2.3` | Shows each layer, its size and the instruction that created it. Use to find what bloats the image. |
| `docker inspect app:1.2.3 \| jq '.[0].Config'` | Entrypoint, CMD, env, exposed ports, labels as built. Use to check what the image will actually run. |
| `docker save app:1.2.3 \| gzip > app.tgz` / `docker load < app.tgz` | Export/import an image as a tarball. For air-gapped transfer. |
| `docker scout cves app:1.2.3` / `trivy image --severity HIGH,CRITICAL app:1.2.3` | Vulnerability scan. Put in CI and fail on CRITICAL. |
| `dive app:1.2.3` | Interactive layer explorer showing wasted space. Use when optimising image size. |
| `docker manifest inspect nginx:latest` | Shows which architectures a tag provides. Check before pulling onto arm64. |

## ECR

| Command | What it does / when to use |
|---|---|
| `aws ecr get-login-password --region ap-south-1 \| docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-south-1.amazonaws.com` | Authenticates Docker to ECR with a 12-hour token. Run before push/pull; "no basic auth credentials" means it expired. |
| `aws ecr create-repository --repository-name app --image-tag-mutability IMMUTABLE --image-scanning-configuration scanOnPush=true --encryption-configuration encryptionType=KMS` | Create a repo where tags can't be overwritten (so `1.2.3` always means the same bits), scanned on push, encrypted with KMS. The bank-grade default. |
| `aws ecr describe-images --repository-name app --query 'sort_by(imageDetails,&imagePushedAt)[-5:].[imageTags[0],imagePushedAt]' --output table` | Last five pushed images. Use to confirm a pipeline pushed. |
| `aws ecr put-lifecycle-policy --repository-name app --lifecycle-policy-text file://lifecycle.json` | Auto-expire untagged images after N days and keep only the last N tagged. Controls storage cost. |
| `aws ecr batch-delete-image --repository-name app --image-ids imageTag=old` | Delete a specific image. |
| `aws ecr describe-image-scan-findings --repository-name app --image-id imageTag=1.2.3` | Read scan results (Inspector enhanced scanning). Gate deploys on this. |

## Containers

| Command | What it does / when to use |
|---|---|
| `docker run -d --name api -p 8080:8080 app:1.2.3` | Start detached, name it, map host port 8080 → container 8080. |
| `docker run --env-file .env -e LOG_LEVEL=debug ...` | Inject environment: from a file and/or inline. Never bake config into the image. |
| `docker run -v $PWD/data:/data ...` | Bind-mount a host directory. For local dev; in prod use named volumes or external storage. |
| `docker run --restart unless-stopped ...` | Auto-restart on crash or daemon restart. For single-host deployments without an orchestrator. |
| `docker run --memory 512m --cpus 1 ...` | Resource limits via cgroups. Same idea as Kubernetes limits; test OOM behaviour locally. |
| `docker run --rm -it --entrypoint sh app:1.2.3` | Override the entrypoint to get a shell in the image. Use to inspect files when a container won't start. |
| `docker run --rm -it --network host nicolaka/netshoot` | Network troubleshooting toolbox (dig, curl, tcpdump, nmap) sharing the host network. |
| `docker ps` / `docker ps -a --filter status=exited` | Running containers / all including exited ones. |
| `docker ps --format '{{.Names}}\t{{.Status}}\t{{.Ports}}'` | Custom columns. |
| `docker logs -f --tail 100 --since 10m api` | Follow logs from the last 10 minutes. `2>&1 \| grep ERROR` to filter (logs go to both streams). |
| `docker exec -it api sh` | Shell into a running container. `-u root` to escalate when the image runs as non-root. |
| `docker exec api env` | Print the container's environment: verifies what config it received. |
| `docker inspect api \| jq '.[0].State, .[0].NetworkSettings.Networks'` | Exit code, OOMKilled flag, IP addresses. First stop when a container died. |
| `docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}}' api` | Just the two fields that matter after a crash. |
| `docker stats --no-stream` | CPU/memory/network per container snapshot. |
| `docker top api` | Processes inside the container. |
| `docker port api` | Port mappings. |
| `docker diff api` | Files changed since the image started. Finds apps that write where they shouldn't (why read-only root FS breaks). |
| `docker stop api` | SIGTERM, wait 10 s, SIGKILL. Use `-t 30` for apps that need longer. |
| `docker kill api` | SIGKILL immediately. |
| `docker rm -f api` | Remove, killing if needed. |
| `docker restart api` | Stop + start. |
| `docker cp api:/app/log.txt .` | Copy files in/out of a container. |
| `docker update --restart=always api` | Change restart policy on a running container. |
| `docker events --since 10m` | Daemon event stream: starts, dies, OOMs. Use to correlate crashes. |
| `docker container prune` | Remove all stopped containers. |
| `docker system prune -af --volumes` | Remove everything unused including volumes. Destructive; good for CI runners, dangerous on a dev box with data. |
| `docker system df` | Disk used by images, containers, volumes, build cache. |

## Networks and volumes

| Command | What it does / when to use |
|---|---|
| `docker network ls` / `docker network inspect appnet` | List networks; inspect shows which containers are attached and their IPs. |
| `docker network create --driver bridge appnet` | User-defined bridge: containers on it resolve each other by name. The default bridge does not do DNS. |
| `docker run --network appnet --name db postgres` then `docker run --network appnet app` | `app` can reach `db:5432` by name. Fixes "connection refused to localhost" mistakes. |
| `docker network connect appnet api` / `disconnect` | Attach/detach a running container. |
| Drivers | `bridge` (default NAT), `host` (share host stack; no port mapping, Linux only), `none` (isolated), `overlay` (multi-host Swarm), `macvlan` (container gets a LAN IP). |
| `docker volume create pgdata` / `docker volume ls` / `inspect` / `prune` | Named volumes managed by Docker; survive container removal. |
| `-v pgdata:/var/lib/postgresql/data` | Named volume mount: persistent data. |
| `-v $PWD/conf:/etc/app:ro` | Bind mount read-only: inject config from the host. |
| `--mount type=tmpfs,destination=/tmp` | In-memory scratch space; pairs with a read-only root filesystem. |

## Dockerfile reference

```dockerfile
# syntax=docker/dockerfile:1                      # enable BuildKit features (cache/secret mounts)
ARG NODE=20                                       # build-time variable; available to FROM
FROM node:${NODE}-alpine AS deps                  # stage 1: install deps only; cached until package*.json changes
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci --omit=dev   # cache mount: npm cache persists across builds, not in the image

FROM node:${NODE}-alpine AS build                 # stage 2: compile
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM gcr.io/distroless/nodejs20-debian12          # stage 3: runtime; no shell, no package manager = small attack surface
WORKDIR /app
ENV NODE_ENV=production PORT=8080                 # runtime env defaults (overridable with -e)
COPY --from=build --chown=nonroot:nonroot /app/dist ./dist          # copy only artifacts, owned by the runtime user
COPY --from=deps  --chown=nonroot:nonroot /app/node_modules ./node_modules
USER nonroot                                      # never run as root in prod
EXPOSE 8080                                       # documentation only; publishing is done at run time
LABEL org.opencontainers.image.source="https://github.com/org/app" org.opencontainers.image.revision="$GIT_SHA"   # traceability
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 CMD ["/nodejs/bin/node","dist/healthcheck.js"]   # docker/compose only; K8s uses probes
CMD ["dist/server.js"]                            # exec form: process is PID 1 and receives SIGTERM directly
```

| Instruction | What it does / gotcha |
|---|---|
| `FROM` | Base image. Pin a tag or digest; `AS name` names a stage for `COPY --from`. |
| `ARG` vs `ENV` | ARG exists only during build (re-declare after FROM to use it); ENV persists into the running container. Don't put secrets in either. |
| `COPY` vs `ADD` | COPY copies files. ADD also unpacks tar archives and fetches URLs; prefer COPY for predictability. |
| `RUN` | Executes at build time, creates a layer. Chain with `&&` and clean caches in the same RUN so deleted files don't stay in a lower layer. |
| `WORKDIR` | Sets and creates the working directory. Use instead of `RUN cd`. |
| `USER` | Switch to a non-root user for subsequent instructions and runtime. Create it with a fixed UID (`adduser -u 10001`) so Kubernetes `runAsUser` matches. |
| `EXPOSE` | Metadata only. |
| `ENTRYPOINT` / `CMD` | ENTRYPOINT is the fixed executable, CMD its default arguments (overridden by `docker run image args`). Use exec (JSON array) form for both so signals reach the process. |
| `HEALTHCHECK` | Container-level health for Docker/Compose/Swarm. Kubernetes ignores it. |
| `.dockerignore` | Exclude `.git`, `node_modules`, secrets, `.env`, docs from the build context: faster builds and no accidental secret leaks. |

Layer caching: order instructions least → most frequently changing (dependency manifests before source code). `RUN`/`COPY`/`ADD` create layers; `ENV`/`LABEL`/`EXPOSE` are metadata.

## Compose

```yaml
services:
  api:
    build: { context: ., target: build }        # build from Dockerfile, stop at a stage (dev image with tools)
    image: app:dev                              # tag for the built image
    ports: ["8080:8080"]
    env_file: .env                              # local secrets stay out of the file
    environment: { DB_HOST: db }                # service name resolves via Compose DNS
    depends_on:
      db: { condition: service_healthy }        # wait for db healthcheck, not just "started"
    volumes: [./src:/app/src]                   # live-reload source in dev
    healthcheck: { test: ["CMD","curl","-f","http://localhost:8080/healthz"], interval: 10s, retries: 5 }
    restart: unless-stopped
  db:
    image: postgres:16
    environment: { POSTGRES_PASSWORD: dev }
    volumes: [pgdata:/var/lib/postgresql/data]  # persist across `compose down`
    healthcheck: { test: ["CMD-SHELL","pg_isready -U postgres"], interval: 5s }
volumes: { pgdata: {} }
```

| Command | What it does / when to use |
|---|---|
| `docker compose up -d --build` | Build and start everything in the background. |
| `docker compose ps` / `logs -f api` / `exec api sh` | Status, logs, shell for one service. |
| `docker compose down -v` | Stop and remove containers, networks and named volumes (data loss). Omit `-v` to keep data. |
| `docker compose pull` | Refresh images. |
| `docker compose config` | Print the fully resolved file (variables substituted). Debugs "why isn't my env var set". |
| `docker compose --profile debug up` | Start services tagged with a profile (e.g. a debug sidecar). |

## Security flags

| Flag | What it does / when to use |
|---|---|
| `--read-only --tmpfs /tmp` | Immutable root filesystem with writable scratch. Blocks many exploits; matches K8s `readOnlyRootFilesystem`. |
| `--cap-drop ALL --cap-add NET_BIND_SERVICE` | Remove all Linux capabilities, add back only what's needed (bind port <1024). |
| `--security-opt no-new-privileges` | Prevents setuid binaries from escalating. |
| `--user 10001` | Run as a specific non-root UID even if the image doesn't set `USER`. |
| `--pids-limit 100` | Caps processes: stops fork bombs. |
| `--memory 256m --memory-swap 256m` | Hard memory limit with no swap. |
| Never | `--privileged`, mounting `/var/run/docker.sock` into app containers (host root equivalent). Use rootless Docker where possible. |
| `cosign sign repo/app@sha256:...` / `cosign verify` | Sign and verify images; admission controllers can require signatures. |
| `syft app:1.2 -o spdx-json > sbom.json` | Generate a software bill of materials for compliance and vulnerability tracking. |

## Troubleshooting

| Symptom | Cause → check/fix |
|---|---|
| Exits immediately with code 0 | CMD finished (e.g. a daemon forked to background) → run the process in the foreground (`nginx -g 'daemon off;'`). |
| Exit 137 | SIGKILL: OOMKilled (`docker inspect` → `OOMKilled: true`) or `docker kill` → raise `--memory` or fix the leak. |
| Exit 139 / 143 | 139 = segfault; 143 = SIGTERM handled cleanly (normal stop). |
| `exec format error` | Image built for another CPU arch → `--platform linux/amd64`. |
| `permission denied` on a bind mount | UID inside ≠ owner outside, or SELinux → match UIDs, or `:z`/`:Z` mount suffix on SELinux hosts. |
| App can't reach `localhost:5432` | Each container has its own localhost → use the service/container name on a shared network. |
| `port is already allocated` | Host port in use → `ss -tlnp`, stop the other container or change the host port. |
| `no basic auth credentials` | ECR token expired (12 h) → re-run the login command. |
| `toomanyrequests` from Docker Hub | Anonymous pull rate limit → authenticate, or use an ECR pull-through cache. |
| Slow builds / cache misses | COPY of whole source before dependency install, no `.dockerignore` → reorder, add ignore file, use cache mounts. |
| Wrong time/timezone in logs | Container defaults to UTC → `-e TZ=Asia/Kolkata` or mount `/etc/localtime`. |
| `docker stop` takes 10 s | Shell-form CMD: PID 1 is `sh`, which ignores SIGTERM → exec form or `tini` as init. |
| Disk full from logs | json-file driver unbounded → `--log-opt max-size=10m --log-opt max-file=3` or set in daemon.json. |

`/etc/docker/daemon.json` (restart dockerd after): `{"log-driver":"json-file","log-opts":{"max-size":"10m","max-file":"3"},"default-address-pools":[{"base":"172.80.0.0/16","size":24}],"registry-mirrors":["https://mirror.example"]}`.

## Docker vs containerd vs Podman

Docker = CLI + daemon + BuildKit on top of containerd. Kubernetes talks to containerd directly (dockershim removed in 1.24), so on nodes use `crictl ps`, `crictl logs`, `ctr -n k8s.io images ls`. Podman = daemonless, rootless, Docker-compatible CLI. Finch (AWS) = Docker-compatible CLI for macOS built on Lima + nerdctl.
