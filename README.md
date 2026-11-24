# Monitoring Agent

Minimal host-monitoring stack: [node-exporter](https://github.com/prometheus/node_exporter) (host metrics) + [cadvisor](https://github.com/google/cadvisor) (container metrics), both fronted by an nginx reverse proxy that restricts access to an IP allowlist.

## Architecture

```
                 ┌──────────────────────────────┐
scraper ───────► │ nginx (LISTEN_IP)             │
 (allowlisted    │  :NODE_EXPORTER_PORT ─┐       │
  IP only)       │  :CADVISOR_PORT ─┐    │       │
                 └───────────────┼──┼────┘       │
                                  │  │
                     127.0.0.1:8081 127.0.0.1:9101
                          │             │
                     ┌────▼───┐    ┌────▼────────┐
                     │cadvisor│    │node-exporter│
                     └────────┘    └─────────────┘
```

- **node-exporter** runs with `network_mode: host` (needed to read the host's real network interfaces) but binds to `127.0.0.1:9101` — not reachable from outside the host.
- **cadvisor** runs unprivileged on the default compose network, with `--docker_only=true`, and only publishes to `127.0.0.1:8081`.
- **nginx** is the only service actually exposed on the network. It also runs with `network_mode: host` (so it can reach the two `127.0.0.1` backends above) and listens on `LISTEN_IP:NODE_EXPORTER_PORT` / `LISTEN_IP:CADVISOR_PORT`, rejecting any client IP not in `nginx/allowlist.conf`.

## Requirements

- Docker Engine with the Compose plugin (`docker compose`)
- Linux host (uses `network_mode: host`, `/dev/kmsg`, host `/proc`, `/sys`, etc. — won't work on Docker Desktop/macOS/Windows)

## Setup

1. Copy the env template and fill in your values:

   ```bash
   cp .env.example .env
   ```

   | Variable              | Description                                              | Example      |
   |------------------------|-----------------------------------------------------------|--------------|
   | `LISTEN_IP`            | IP nginx binds its public listeners to (e.g. a WireGuard/VPN IP — avoid `0.0.0.0` in production) | `10.10.0.1`  |
   | `NODE_EXPORTER_PORT`   | External port for node-exporter, proxied by nginx          | `9100`       |
   | `CADVISOR_PORT`        | External port for cadvisor, proxied by nginx                | `8080`       |

   `.env` is gitignored — it's local/host-specific and never committed.

2. Configure the IP allowlist in `nginx/allowlist.conf` (also gitignored, so it needs to exist locally before starting nginx):

   ```nginx
   allow 10.10.0.5;
   allow 10.10.0.0/24;

   deny all;
   ```

   By default the file ends in `deny all;`, so nginx rejects everyone until you add `allow` lines. Keep `deny all;` last.

3. Start the stack:

   ```bash
   docker compose up -d
   ```

4. Verify (from an allowlisted IP):

   ```bash
   curl http://<LISTEN_IP>:<NODE_EXPORTER_PORT>/metrics
   curl http://<LISTEN_IP>:<CADVISOR_PORT>/metrics
   ```

   A request from a non-allowlisted IP should get connection refused/reset by nginx.

## Changing ports

- **External ports** (what clients/scrapers hit): just change `NODE_EXPORTER_PORT` / `CADVISOR_PORT` in `.env` and restart (`docker compose up -d`) — nginx's listen ports are generated from these via `envsubst` at container start (see `nginx/templates/monitoring.conf.template`).
- **Internal ports** (what node-exporter/cadvisor actually listen on behind nginx, currently `9101` and `8081`): these are not templated from `.env`. To change them you need to update three places consistently:
  1. `--web.listen-address` in the `node-exporter` command (docker-compose.yml)
  2. the `ports` mapping under `cadvisor` (docker-compose.yml)
  3. the matching `proxy_pass` targets in `nginx/templates/monitoring.conf.template`

## Restarting after config changes

- Changes to `.env` or `docker-compose.yml`: `docker compose up -d`
- Changes to `nginx/allowlist.conf` or `nginx/templates/*.template`: `docker compose restart nginx` (both are read at container start, no live reload)