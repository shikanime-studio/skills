# nginx-strangler routing notes

This repo uses `nginx-strangler` as the single API entrypoint in local migration work.

Key facts:
- client proxy should point to the strangler, not directly to either backend
- strangler routes migrated `/api/v1/*` prefixes to `server-nestjs`
- everything else under `/api/` should fall back to the legacy server
- local dev uses native backend processes plus a Dockerized strangler

Ports observed in this workspace:
- legacy server: `4001`
- NestJS server: `3001`
- nginx-strangler: `4000`

Pitfall:
- if requests appear to "all hit NestJS", verify the client proxy target and the local server ports before changing routing rules
- if strangler cannot forward traffic, verify `host.docker.internal` resolution from inside the container; Rancher Desktop may differ from Docker Desktop here

Quick checks:
1. `curl -I http://localhost:4000/health`
2. `curl -I http://localhost:4000/api/v1/version`
3. `curl -I http://localhost:4000/api/v1/health-services`
4. If needed, inspect the resolved nginx config and upstreams inside the container.
