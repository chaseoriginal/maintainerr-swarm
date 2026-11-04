# maintainerr-swarm

Portainer-managed stack. **Git is the source of truth**; Portainer owns the
deployment and polls this repository for changes.

| | |
|---|---|
| Portainer host | ore-dock-110-1 |
| Endpoint | 5 (dock-110-7 swarm) |
| Swarm | True |
| Live service | svc:maintainerr-swarm_maintainerr |
| Git repo | https://github.com/chaseoriginal/maintainerr-swarm |
| Compose file | `docker-compose.yml` |
| Auto-update poll | every 15m |

## What it is

Maintainerr is a library-maintenance tool for Plex/Jellyfin/Emby. It applies
rules over media-server and *arr metadata, collects matching titles into a
grace-period collection, then clears them from the media server, the *arrs and
Seerr.

## Placement

- **Node:** `ore-dock-110-9` in the 110-7 swarm, pinned by the
  `deployment.target.maintainerr` node label. The data volume is node-local,
  so the service must not be rescheduled off that node.
- **Port:** 6246, published in host mode. The proxy reaches it at
  `10.77.110.90:6246`.

## Files

- `docker-compose.yml` - the deployed compose.

  The named volume is declared with an explicit `name:` on purpose. Docker
  Swarm prefixes a plain named volume with the stack name, so declaring
  `maintainerr-swarm_data` without a pinned `name` would silently mount a
  different, EMPTY volume and the service would come up with no config.

  The two media binds are required for the **leftover-folder cleanup** action.
  They are mounted at `/movies` and `/tv` because that is the path Radarr and
  Sonarr report for their root folders - Maintainerr can only delete files
  from disk if it sees the library at the same path the arrs do.

- `.env.example` - placeholder only. All service configuration is entered in
  the Maintainerr UI.

## How a change is deployed

1. Edit `docker-compose.yml` and push to `main`.
2. Portainer polls the repo and redeploys automatically.

## Verifying a deployment

```bash
docker service ls --filter name=maintainerr-swarm
docker service inspect maintainerr-swarm_maintainerr \
  --format '{{json .Spec.TaskTemplate.ContainerSpec.Mounts}}'
curl -s http://10.77.110.90:6246/api/health/ready
```

A healthy stack shows `1/1`, the named volume `maintainerr-swarm_data`, and
`/api/health/ready` returns 200.

## URL

`https://maintain.echoinggreen.net` - **internal only**.

There is deliberately NO public DNS record. Maintainerr has no built-in
login, and `GET /api/settings/database/download` returns the entire config
database (Plex tokens, *arr API keys, Seerr keys) to any client that can
reach it. The name resolves only via AD DNS, and the proxy vhost carries the
"Private Site Access" access list, so it answers only from permitted ranges.
