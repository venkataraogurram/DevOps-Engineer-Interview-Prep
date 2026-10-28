# Docker cheatsheet

[← Cheatsheet index](README.md)

## Images

```bash
docker build -t app:1.2.3 .; docker build -t app:dev -f Dockerfile.dev --target build .
docker build --platform linux/amd64 -t app .                # M1 Mac → x86 nodes
docker build --no-cache --build-arg VERSION=1.2 --secret id=npmrc,src=$HOME/.npmrc .
docker buildx create --use; docker buildx build --platform linux/amd64,linux/arm64 -t repo/app:1.2 --push .
docker images; docker image ls --filter dangling=true; docker image prune -a
docker tag app:1.2.3 123.dkr.ecr.ap-south-1.amazonaws.com/app:1.2.3
docker push ...; docker pull ...; docker pull nginx@sha256:abc...    # by digest
docker history app:1.2.3; docker inspect app:1.2.3 | jq '.[0].Config'
docker save app:1.2.3 | gzip > app.tgz; docker load < app.tgz
docker scout cves app:1.2.3; trivy image --severity HIGH,CRITICAL app:1.2.3; dive app:1.2.3
docker manifest inspect nginx:latest
```

## ECR

```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-south-1.amazonaws.com
aws ecr create-repository --repository-name app --image-tag-mutability IMMUTABLE --image-scanning-configuration scanOnPush=true --encryption-configuration encryptionType=KMS
aws ecr describe-images --repository-name app --query 'sort_by(imageDetails,&imagePushedAt)[-5:].[imageTags[0],imagePushedAt]' --output table
aws ecr put-lifecycle-policy --repository-name app --lifecycle-policy-text file://lifecycle.json   # expire untagged >7d, keep last 30
aws ecr batch-delete-image --repository-name app --image-ids imageTag=old
```

## Containers

```bash
docker run -d --name api -p 8080:8080 --env-file .env -e LOG_LEVEL=debug -v $PWD/data:/data --restart unless-stopped --memory 512m --cpus 1 app:1.2.3
docker run --rm -it --entrypoint sh app:1.2.3            # debug shell
docker run --rm -it --network host nicolaka/netshoot    # network debug toolbox
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock ...   # (avoid in prod)
docker ps; docker ps -a --filter status=exited; docker ps --format '{{.Names}}\t{{.Status}}\t{{.Ports}}'
docker logs -f --tail 100 --since 10m api; docker logs api 2>&1 | grep ERROR
docker exec -it api sh; docker exec api env; docker exec -u root api sh
docker inspect api | jq '.[0].State, .[0].NetworkSettings.Networks'
docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}}' api
docker stats --no-stream; docker top api; docker port api; docker diff api
docker stop api (SIGTERM, 10 s) ; docker kill api; docker rm -f api; docker restart api
docker cp api:/app/log.txt .; docker cp f.txt api:/tmp/
docker update --restart=always api; docker rename a b; docker pause/unpause
docker events --since 10m
docker container prune; docker system prune -af --volumes; docker system df
```

## Networks and volumes

```bash
docker network ls; docker network create --driver bridge appnet; docker network inspect appnet
docker run --network appnet --name db postgres; docker run --network appnet app   # resolves "db" by name
docker network connect appnet api; docker network disconnect
# drivers: bridge (default, NAT), host (share host netns), none, overlay (swarm), macvlan
docker volume ls; docker volume create pgdata; docker volume inspect pgdata; docker volume prune
-v pgdata:/var/lib/postgresql/data     # named volume
-v $PWD/conf:/etc/app:ro               # bind mount
--mount type=tmpfs,destination=/tmp
```

## Dockerfile reference

```dockerfile
# syntax=docker/dockerfile:1
ARG NODE=20
FROM node:${NODE}-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci --omit=dev

FROM node:${NODE}-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
ENV NODE_ENV=production PORT=8080
COPY --from=build --chown=nonroot:nonroot /app/dist ./dist
COPY --from=deps  --chown=nonroot:nonroot /app/node_modules ./node_modules
USER nonroot
EXPOSE 8080
LABEL org.opencontainers.image.source="https://github.com/org/app" org.opencontainers.image.revision="$GIT_SHA"
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 CMD ["/nodejs/bin/node","dist/healthcheck.js"]
CMD ["dist/server.js"]
```

| Instruction | Notes |
|---|---|
| `FROM` | pin tag or digest; `AS name` for stages |
| `ARG` vs `ENV` | ARG build-time only (before FROM needs re-declare); ENV persists in image |
| `COPY` vs `ADD` | prefer COPY; ADD unpacks tars / fetches URLs |
| `RUN` | chain with `&&`, clean caches in same layer (`rm -rf /var/lib/apt/lists/*`) |
| `WORKDIR` | creates dir; avoid `cd` |
| `USER` | non-root; create user with fixed UID (`adduser -u 10001`) |
| `EXPOSE` | documentation only |
| `ENTRYPOINT`/`CMD` | exec form `["bin","arg"]`; ENTRYPOINT fixed, CMD default args |
| `HEALTHCHECK` | K8s ignores it (uses probes) |
| `.dockerignore` | `.git`, `node_modules`, `*.md`, secrets, `.env` |

Layer caching: order instructions least → most changing (deps manifest before source). Each `RUN/COPY/ADD` = layer; `ENV/LABEL/EXPOSE` = metadata.

## Compose

```yaml
services:
  api:
    build: { context: ., target: build }
    image: app:dev
    ports: ["8080:8080"]
    env_file: .env
    environment: { DB_HOST: db }
    depends_on:
      db: { condition: service_healthy }
    volumes: [./src:/app/src]
    healthcheck: { test: ["CMD","curl","-f","http://localhost:8080/healthz"], interval: 10s, retries: 5 }
    restart: unless-stopped
  db:
    image: postgres:16
    environment: { POSTGRES_PASSWORD: dev }
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck: { test: ["CMD-SHELL","pg_isready -U postgres"], interval: 5s }
volumes: { pgdata: {} }
```

```bash
docker compose up -d --build; docker compose ps; docker compose logs -f api; docker compose exec api sh
docker compose down -v; docker compose pull; docker compose config; docker compose --profile debug up
```

## Security

```bash
docker run --read-only --tmpfs /tmp --cap-drop ALL --cap-add NET_BIND_SERVICE --security-opt no-new-privileges --user 10001 app
docker run --pids-limit 100 --memory 256m --memory-swap 256m app
# rootless docker; never mount docker.sock into app containers; no --privileged
cosign sign repo/app@sha256:...; cosign verify ...
syft app:1.2 -o spdx-json > sbom.json
```

## Troubleshooting

| Symptom | Check |
|---|---|
| Exits immediately (code 0) | CMD finished; needs foreground process (`nginx -g 'daemon off;'`) |
| Exit 137 | OOMKilled (`docker inspect` OOMKilled true) or SIGKILL; raise `--memory` |
| Exit 139 | segfault; 143 = SIGTERM handled |
| `exec format error` | arch mismatch → `--platform linux/amd64` |
| `permission denied` on bind mount | UID mismatch / SELinux (`:z` or `:Z` suffix) |
| Can't reach other container via localhost | use service/container name on same user-defined network |
| Port already allocated | `ss -tlnp`; another container; change host port |
| Image pull: `no basic auth credentials` | ECR login expired (12 h) |
| `too many requests` from Docker Hub | rate limit → ECR pull-through cache / login |
| Build slow / cache busting | COPY order; `.dockerignore`; BuildKit cache mounts |
| Container clock/timezone | `-e TZ=Asia/Kolkata` / mount `/etc/localtime` |
| Signals ignored (slow stop) | shell-form CMD → exec form or `tini` as PID 1 |
| Logs filling disk | `--log-opt max-size=10m --log-opt max-file=3` / daemon.json |

`/etc/docker/daemon.json` → `{"log-driver":"json-file","log-opts":{"max-size":"10m","max-file":"3"},"default-address-pools":[...],"registry-mirrors":[...]}` then `systemctl restart docker`.

## Docker vs containerd vs Podman (one-liners)

Docker = CLI + daemon + BuildKit on top of containerd. Kubernetes uses containerd directly (`crictl ps`, `crictl logs`, `ctr -n k8s.io images ls`), dockershim removed in 1.24. Podman = daemonless, rootless, Docker-compatible CLI. Finch (AWS) = Docker-compatible CLI on Lima + nerdctl for Mac.
