# omada (upstream image, no custom build)

This service uses the upstream `mbentley/omada-controller` image directly —
there is no `Dockerfile`, `docker/` context, or `entrypoint.sh` here.

## Why upstream

Per compose spec rule 6, an upstream image is acceptable when it *is* the
service. The Omada Controller is a closed-source TP-Link application; there
is nothing to build from source. `mbentley/omada-controller` is the
community-standard containerization of it (multi-arch, version-tagged,
handles PUID/PGID and port env vars internally).

## Why no entrypoint

Per rules 8–9, a custom entrypoint is only needed for first-run seeding in
custom-built images. The upstream image already seeds its own state on first
start and manages its own user/privileges via `PUID`/`PGID`. Adding another
entrypoint layer would only add failure modes, so there is none — and
therefore no `su-exec` privilege-drop logic of ours either.

## Environment variables

| Variable | Required | Purpose |
| --- | --- | --- |
| `OMADA_VERSION` | yes | Image tag; major or major.minor pin (e.g. `6`, `6.3`). Never `latest` (maintainer-deprecated, still points at v5). |
| `PUID` / `PGID` | yes | IDs the controller process runs as. |
| `TZ` | yes | Timezone (e.g. `Europe/Berlin`). |
| `MANAGE_HTTP_PORT` | yes | Management UI HTTP port (default `8088`). |
| `MANAGE_HTTPS_PORT` | yes | Management UI HTTPS port (default `8043`). |
| `PORTAL_HTTP_PORT` | yes | Captive portal HTTP port. |
| `PORTAL_HTTPS_PORT` | yes | Captive portal HTTPS port (`8843`). |
| `SHOW_SERVER_LOGS` | no | Verbose controller logs (`true`/`false`). |
| `SHOW_MONGODB_LOGS` | no | Verbose MongoDB logs (`true`/`false`). |
| `OMADA_GEN_SELF_SIGNED_CERT` | no | Self-signed TLS opt-in for the proxy (`GEN_SELF_SIGNED_CERT`), default `true` in compose (LAN/self-signed deployment). |

Proxy vars (`VIRTUAL_HOST`, `VIRTUAL_PORT`, `VIRTUAL_PROTO`, `ACME_HOST`)
are wired in `docker-compose.yml` and need no per-deployment tuning beyond
`BASE_DOMAIN`. Note the backend is HTTPS (`VIRTUAL_PORT: 8043` +
`VIRTUAL_PROTO: https`): the controller 302-redirects its HTTP endpoint to
its own HTTPS port, so proxying to plain HTTP would bounce the browser back
out to `https://<host>:8043` with the controller's TP-Link certificate. The
proxy does not verify the backend's self-signed cert, so this just works.

## Reverse proxy status

TP-Link does not officially support running the controller behind a reverse
proxy — the documented access is `https://<host>:8043` directly, and the
image maintainer's preferred networking is host mode. Proxying is
community territory with known quirks (the HTTP→HTTPS redirect this project
works around, blank pages if internal/external ports are remapped). The
setup in `docker-compose.yml` follows the community-accepted pattern:
HTTPS backend (`VIRTUAL_PORT: 8043` + `VIRTUAL_PROTO: https`), default
internal ports unchanged.

The image-supported way to get a proper certificate on the controller
itself is mounting `/cert/tls.key` + `/cert/tls.crt` (full chain, see
`SSL_CERT_NAME`/`SSL_KEY_NAME`) or installing certs via the web UI
(4.1+) — pick one method, never mix them.

## Security notes

- No secrets in env: initial admin credentials are created via the setup
  wizard on first run, not via environment variables.
- The controller runs as `PUID`/`PGID`, not root, per the upstream image.
- Device adoption ports are LAN-reachable by design; do not expose this host
  to the internet without firewalling everything except the proxied UI.
