# wirego

**Self-hosted WireGuard VPN manager with a web dashboard.**

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
![Platforms](https://img.shields.io/badge/platforms-amd64%20%7C%20arm64%20%7C%20armv7-informational)
![Go](https://img.shields.io/badge/go-1.26-00ADD8?logo=go&logoColor=white)

wirego lets you run your own VPN server without depending on any third-party service. Manage peers, apply group-based firewall rules, monitor latency, and get structured audit logs — all from a browser.

> Built and maintained by [vernizus](https://github.com/vernizus).

---

## Features

- **Web dashboard** — add/remove peers, configure groups, visualise topology, all from the UI
- **Group-based firewall** — isolate traffic between peer groups using nftables (or iptables fallback)
- **DNS proxy** — internal name resolution for peers, with optional Magic DNS
- **Latency monitoring** — real-time ICMP latency per peer
- **Structured audit log** — every auth and admin action logged as structured JSON, SIEM-ready
- **Encrypted peer keys** — private keys AES-256 encrypted at rest in SQLite
- **JWT with refresh token rotation** — short-lived access tokens; stolen refresh tokens are detectable
- **Single binary** — statically linked, no runtime dependencies, runs on Linux (amd64 / arm64 / armv7)

---

## Quick start

### Docker Compose

```bash
mkdir /opt/wirego && cd /opt/wirego

curl -fsSL https://raw.githubusercontent.com/vernizus/wirego/main/docker/docker-compose.yml \
     -o docker-compose.yml

DB_KEY=$(head -c 32 /dev/urandom | base64 | tr -d '\n')
JWT_KEY=$(head -c 64 /dev/urandom | base64 | tr -d '\n')
printf "WIREGO_DB_ENCRYPTION_KEY=%s\nWIREGO_JWT_SIGNING_KEY=%s\n" "$DB_KEY" "$JWT_KEY" > .env
chmod 600 .env

docker compose up -d
```

Open `http://<your-server-ip>:51821/`. There's no fixed default password — a random one is generated on first start and printed once to the container logs (`docker compose logs`). Save it immediately.

→ Full guide: [docs/deploy-docker.md](docs/deploy-docker.md)

---

## Configuration

| Variable | Required | Default | Description |
|---|---|---|---|
| `WIREGO_DB_ENCRYPTION_KEY` | **yes** | — | AES-256 key for peer private keys. `head -c 32 /dev/urandom \| base64` |
| `WIREGO_JWT_SIGNING_KEY` | **yes** | — | Ed25519 seed for JWT signing. `head -c 64 /dev/urandom \| base64` |

> **Never change these two keys after first start** — peer keys become unreadable and all sessions are invalidated.

Everything else — firewall backend, log level, public IP/domain, DNS, TLS, and more — is configured from the web panel after first login (**System → Settings**). The panel marks which settings apply instantly and which need a `docker compose restart`.

A handful of edge-case-only variables (custom WireGuard port/subnet, etc.) have no panel equivalent — see [Advanced](docs/deploy-docker.md#advanced-optional) in the full guide if you need them.

---

## Updating

```bash
docker compose pull && docker compose up -d
```

Migrations run automatically on startup.

---

## Security notes

- Container runs as `root` with `CAP_NET_ADMIN` (required by WireGuard and nftables). All other capabilities are dropped. Filesystem is read-only except `/tmp` (tmpfs) and the `/data` volume.
- Refresh tokens are single-use. A stolen token causes a detectable collision on next use.
- Back up `.env` securely — it cannot be recovered if lost.

---

## Privacy & telemetry

wirego can optionally send anonymous crash reports and usage metrics to
help improve the software — **disabled by default**, toggled from
**System → Settings**, and never including peer IPs, keys, names, or any
VPN traffic data. Full details: [docs/privacy.md](docs/privacy.md).

---

## Repository layout

```
docker/
  docker-compose.yml      # Docker Compose file (pulls from Docker Hub)
  .env.example            # Template for the two required secrets
docs/
  deploy-docker.md        # Full Docker Compose deployment guide
  privacy.md              # What telemetry data is sent, and what never is
```

---

## Author

**vernizus** — [github.com/vernizus](https://github.com/vernizus)

---

## License

Deploy scripts and configuration files in this repository are released under the [MIT License](LICENSE).  
The wirego application binary is proprietary software — the Docker image is provided for use as-is.