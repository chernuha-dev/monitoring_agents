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
                     127.0.0.1:9000 127.0.0.1:9101
                          │             │
                     ┌────▼───┐    ┌────▼────────┐
                     │cadvisor│    │node-exporter│
                     └────────┘    └─────────────┘
```

- **node-exporter** runs with `network_mode: host` (needed to read the host's real network interfaces) but binds to `127.0.0.1:9101` — not reachable from outside the host.
- **cadvisor** runs on the default Compose network with `--docker_only=true`. Docker publishes its container port `8080` at `127.0.0.1:9000` on the host.
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
   | `LISTEN_IP`            | This agent host's VPN or LAN address                       | `10.10.0.1`  |
   | `NODE_EXPORTER_PORT`   | NGINX port for Node Exporter                               | `9101`       |
   | `CADVISOR_PORT`        | NGINX port for cAdvisor                                    | `9100`       |

   Use an IP assigned to this host. With the default Node Exporter port, `0.0.0.0` conflicts with the backend already bound to `127.0.0.1:9101`.

   `.env` is gitignored — it's local/host-specific and never committed.

2. Configure the IP allowlist in `nginx/allowlist.conf` (also gitignored, so it needs to exist locally before starting nginx):

   ```bash
   cp nginx/allowlist.conf.template nginx/allowlist.conf
   ```

   ```nginx
   allow 10.10.0.5;
   allow 10.10.0.0/24;

   deny all;
   ```

   Keep `deny all;` last. If Prometheus runs in a bridge container on the same host and scrapes the host IP, NGINX may see the container's bridge IP as the source. Allow the source IP that NGINX sees and, if the host firewall filters incoming traffic, allow that source to reach both NGINX ports on `LISTEN_IP`. Recheck the rules if the container's IP changes.

3. Start the stack:

   ```bash
   docker compose up -d
   ```

4. Verify (from an allowlisted IP):

   ```bash
   curl http://<LISTEN_IP>:<NODE_EXPORTER_PORT>/metrics
   curl http://<LISTEN_IP>:<CADVISOR_PORT>/metrics
   ```

   NGINX returns HTTP 403 to clients outside the allowlist.

## Changing ports

- **External ports** (what clients/scrapers hit): just change `NODE_EXPORTER_PORT` / `CADVISOR_PORT` in `.env` and restart (`docker compose up -d`) — nginx's listen ports are generated from these via `envsubst` at container start (see `nginx/templates/monitoring.conf.template`).
- **Backend ports** are set in `docker-compose.yml`. For Node Exporter, change `--web.listen-address`. For cAdvisor, change the host port in the `ports` mapping. Update the matching `proxy_pass` in `nginx/templates/monitoring.conf.template`, apply the Compose changes, and recreate NGINX as shown below.

## Restarting after config changes

- Changes to `.env` or `docker-compose.yml`: `docker compose up -d`
- Changes to `nginx/allowlist.conf` or `nginx/templates/*.template`: `docker compose up -d --no-deps --force-recreate nginx`
