# Nginx strangler routing: local verification notes

Use this when requests appear to hit the wrong backend in local dev.

## Observed local port layout (this repo)

- client Vite proxy → nginx-strangler `localhost:4000`
- legacy Fastify → `localhost:4001`
- NestJS → `localhost:3001`

Source of truth: `docker/docker-compose.local.yml` — the `nginx-strangler` service publishes host port **4000** → container port 8080.

## Key pitfall: container not running

**Always verify the nginx-strangler container is running before investigating route rules.**

If the container is down, the Vite proxy (which targets `localhost:4000`) gets nothing — resulting in 502 Bad Gateway for *all* API requests, not just migrated ones. This looks like a routing problem but is actually an infrastructure availability problem.

Quick check:
```bash
docker ps --filter name=nginx-strangler --format '{{.Names}} {{.Status}}'
```

If it's not running:
```bash
docker compose -f docker/docker-compose.local.yml up -d nginx-strangler
```

## Verification steps

1. **Check container is running** (see above)
2. **Read the active routing config from container logs** — the entrypoint prints `LEGACY_UPSTREAM` and `NESTJS_UPSTREAM`
3. **Check runtime listeners on the host** — `lsof -i -P -n | grep -E ':(3001|4001|4000) '`
4. **Compare compose file, env example, and live container ports**
5. **If the container cannot reach host services**, test host-gateway resolution from inside the strangler container

## Rancher Desktop note

If `host.docker.internal` or `host-gateway` behaves differently than Docker Desktop, the strangler may be healthy while upstream forwarding fails. In that case, inspect container logs first — the routing config may be correct while the host path is not.

## Port mapping reference

| Service | Host port | Container port | Compose service |
|---------|-----------|----------------|-----------------|
| nginx-strangler | 4000 | 8080 | `nginx-strangler` |
| legacy server | 4001 | 8080 | `server` (native) |
| server-nestjs | 3001 | 3001 | `server-nestjs` (native) |
| client Vite | 8080 | 8080 | `client` (native) |

Keep the route file and the docs in sync with `docker/docker-compose.local.yml`; that file is the source of truth for local ports.
