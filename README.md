# Omada Controller

TP-Link Omada SDN controller in Docker, wired as a downstream service of the
shared nginx reverse proxy (LAN / self-signed variant).

Upstream image: `mbentley/omada-controller` (see `omada/README.md`). No custom
build — the upstream image *is* the service.

## Architecture

- `omada-controller` — Omada Software Controller (management UI + device
  adoption). Management UI is reached through the proxy at
  `omada.<BASE_DOMAIN>`; device discovery/adoption ports are published
  directly on the host (see below).
- Volumes: `omada-data`, `omada-logs`, `omada-work` (named volumes; state
  survives restarts and rebuilds).
- Network: joins the proxy's external network (`NGINX_PROXY_NETWORK`,
  default `web-proxy`), which must exist before startup.

## Why `ports:` exist here (rule 4 exception)

Rule 4 says no host port publishing — everything through the proxy. That
covers HTTP(S) traffic, i.e. the management UI (port 8088, proxy-only via
`VIRTUAL_HOST`/`VIRTUAL_PORT`). But Omada devices adopt via raw TCP/UDP
ports (29810/udp, 29811–29814, 27001/udp, portal 8843) that an HTTP reverse
proxy cannot forward. Those ports are published directly so APs/switches can
reach the controller. Only the management UI goes through the proxy.

## Quickstart

```sh
cp env.example .env
# edit .env (BASE_DOMAIN, PUID/PGID, TZ, OMADA_VERSION)
docker compose config   # fails fast on missing vars
docker compose up -d
docker compose ps
```

Start the proxy cluster first so the external `web-proxy` network exists.

Open `https://omada.<BASE_DOMAIN>` (self-signed certificate on LAN) and
follow the controller's setup wizard. Default credentials are set on first
run via the wizard, not via env vars.

## Operational notes

- Adopt devices by pointing them at the Docker host IP on the LAN (L3
  adoption) — the published UDP/TCP ports must be reachable from the APs.
- Upgrade: bump `OMADA_VERSION` in `.env`, then
  `docker compose pull && docker compose up -d`. Named volumes keep state.
- Logs: `docker compose logs omada-controller`.
- Reset (reseed): `docker compose down -v` removes all controller state
  (devices, settings), then `up -d` starts fresh.
