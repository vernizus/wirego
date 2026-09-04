# Deploying wirego with Docker Compose

This guide covers deploying wirego on any Linux host (bare metal or VM) using Docker Compose.

## Prerequisites

- Linux host — Debian 12, Ubuntu 22.04+, or equivalent
- `tun` kernel module loaded: `modprobe tun` (add to `/etc/modules` for persistence)
- Docker Engine + Docker Compose plugin installed
- Ports available: `51821/tcp` (web UI, default listen address) and `51820/udp` (WireGuard)

## Step 1 — Download the compose file

```bash
mkdir /opt/wirego && cd /opt/wirego
curl -fsSL https://raw.githubusercontent.com/vernizus/wirego/main/docker/docker-compose.yml \
     -o docker-compose.yml
```

## Step 2 — Generate secrets

These two keys are generated once and must never change after the first start.

```bash
# AES-256 key for encrypting peer private keys at rest
DB_KEY=$(head -c 32 /dev/urandom | base64 | tr -d '\n')

# Ed25519 seed for signing JWT tokens
JWT_KEY=$(head -c 64 /dev/urandom | base64 | tr -d '\n')
```

## Step 3 — Create the .env file

```bash
cat > /opt/wirego/.env <<EOF
WIREGO_DB_ENCRYPTION_KEY=${DB_KEY}
WIREGO_JWT_SIGNING_KEY=${JWT_KEY}
EOF

chmod 600 /opt/wirego/.env
```

A ready-to-copy template with both secrets (plus the Docker-secrets
file-based alternative) is at [`docker/.env.example`](../docker/.env.example).

Everything beyond the two secrets — firewall backend, log level,
public IP/domain, DNS, TLS, and more — is configured from the web
panel after first login (**System → Settings**), not through `.env`.

## Step 4 — Start the service

```bash
cd /opt/wirego
docker compose up -d
```

Open `http://<your-server-ip>:51821/` in a browser (or `https://` on the same port if you later enable TLS from the panel). The admin password is printed to stdout on first run — save it immediately (there is no fixed default password).

## Pinning a specific version

```bash
# In your .env or inline:
VERSION=1.2.0 docker compose up -d
```

## Updating

```bash
cd /opt/wirego
docker compose pull
docker compose up -d
```

wirego runs database migrations automatically on startup — no manual SQL required.

## Useful commands

```bash
# Follow logs
docker compose logs -f

# Check status
docker compose ps

# Stop the service
docker compose down

# Stop and remove all data (irreversible)
docker compose down -v
```

## Backup

The only data that needs to be backed up is:

- `/opt/wirego/.env` — contains your instance secrets. **This file cannot be recovered.**
- The `wirego_data` Docker volume — contains the SQLite database with all peers, groups, and config.

```bash
# Backup the volume
docker run --rm -v wirego_data:/data -v $(pwd):/backup alpine \
    tar czf /backup/wirego-data-$(date +%Y%m%d).tar.gz /data
```

## Advanced (optional)

Five environment variables exist for specific edge cases. Unlike firewall
backend, log level, TLS, or public IP/domain — which you set later from
the web panel (**System → Settings**) — these five have **no panel
equivalent**: `.env` is the only way to change them, and each needs a
container restart to take effect. Most deployments never need to touch any
of these; add them to `.env` only if you hit the specific case described.

| Variable | Default | When you need it |
|---|---|---|
| `WIREGO_WG_PORT` | `51820` | Port `51820/udp` is already used by something else on this host |
| `WIREGO_WG_SUBNET` | `10.8.0.0/24` | `10.8.0.0/24` collides with a network you already route to |
| `WIREGO_WG_INTERFACE` | `wg0` | Interface name `wg0` collides with one already in use on the host |
| `WIREGO_API_LISTEN_ADDR` | `0.0.0.0:51821` | You need the panel bound to one specific interface/IP instead of all of them |
| `WIREGO_DB_PATH` | `/data/wirego.db` | You're changing the volume mount layout in `docker-compose.yml` and need the DB path to match |

## Host firewall

Open the required ports on the host (example with ufw):

```bash
ufw allow 51821/tcp # web UI + API
ufw allow 51820/udp # WireGuard
```

> The VPN subnet traffic (`10.8.0.0/24` by default) is managed internally by wirego — no host firewall rules needed for peer-to-peer traffic.
