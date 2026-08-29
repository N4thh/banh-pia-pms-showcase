
- Nginx runs directly on the VPS host (not containerized) to simplify SSL certificate management and renewal.
- PostgreSQL and Redis communicate only through the internal Docker network and do not expose ports to the Internet, reducing the attack surface.
- VPS: Ubuntu 24.04 LTS, 2 vCPU / 3GB RAM / 40GB NVMe.

## Deployment Process

The `deploy.sh` script automates the full production update workflow:

```bash
./deploy.sh
```

It runs in sequence: `git pull` → rebuild the image (`docker compose up -d --build`) → wait for health checks (up to 60s) → report the result with logs if it fails. This removes the risk of missing a required step or deploying in the wrong order during manual operations.

## Security & Hardening

- SSH: key-based authentication only, password login disabled, and a dedicated non-root user for the application.
- Firewall (UFW): only ports 22 (SSH) and 80/443 (HTTP/HTTPS) are opened — all other services are accessible only through the internal Docker network.
- Secrets: managed via environment variables (`.env`), never committed to Git; rotate secrets immediately if a leak is detected.
- The reverse proxy is the only entry point from the Internet — the database and cache are never exposed directly.

## Backup & Disaster Recovery

- Automatic daily `pg_dump` via cron, compressed with gzip, retaining the 7 most recent backups on the VPS and syncing them offsite periodically.
- Verified through real restore testing: backups were restored into a temporary database and matched the original by table count and row count 100% — not just created without confirming they can actually be used.

## Healthcheck & Monitoring

- Docker health checks for all 4 services (frontend, backend, PostgreSQL, Redis) — the backend validates both app health and database connectivity (`GET /health` via `@nestjs/terminus`), not just whether the process is still alive.
- External uptime monitoring from outside the VPS (UptimeRobot) — detects incidents even if the entire VPS stops responding, without depending on the monitored system itself.
- Log rotation (`max-size: 10m, max-file: 3` per service) — prevents logs from accumulating indefinitely and filling the disk.

## Performance & Load Testing

Validated using [k6](https://k6.io/), simulating 20 concurrent users calling the admin dashboard endpoints (the heaviest read workload, with many aggregate queries):

| Metric | Result |
|--------|--------|
| p95    | 318.68ms |
| p99    | 341.09ms |
| Error rate | 0% (0/2914 requests) |

Conclusion: the current infrastructure (2 vCPU / 3GB RAM) has sufficient headroom for real-world operation (peak 50-100 users, with 20-30 concurrent users during the busiest season).

## Database Migration

Migrations run automatically when the backend container starts (`prisma migrate deploy` in the entrypoint script), ensuring the production schema stays synchronized with the deployed code — no separate manual migration steps are required.

## Operating Stack

Containerization: Docker, Docker Compose
Reverse proxy: Nginx
SSL: Let's Encrypt / Certbot
Process/Container health: Docker healthcheck
Uptime monitoring: UptimeRobot
Load testing: k6
Backup: pg_dump + cron