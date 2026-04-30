# On-Premises Port Operations Portal — Full Stack Deployment

> A production deployment case study: migrating a maritime port operations web portal from AWS cloud to a hardened, high-availability on-premises infrastructure at a major West African seaport.

**Role:** TechOps Engineer (DevOps & SRE)

**Environment:** On-premises RHEL 9.4 + AWS EC2

**Status:** Production

---

## Table of Contents

- [Project Summary](#project-summary)
- [Architecture](#architecture)
- [Infrastructure Stack](#infrastructure-stack)
- [Server Topology](#server-topology)
- [Service Configuration Deep Dives](#service-configuration-deep-dives)
- [Observability Stack](#observability-stack)
- [Network & Proxy Design](#network--proxy-design)
- [Security Hardening](#security-hardening)
- [Key Lessons Learned](#key-lessons-learned)
- [Known Residual Risks](#known-residual-risks)

---

## Project Summary

This project involved the end-to-end deployment of a browser-based port operations portal onto a private on-premises network, replacing a previous AWS-hosted setup. The portal allows port agents to manage shipping containers, book appointments, view invoices, and interact with a third-party terminal management system (N4/Navis) through a secure, authenticated interface.

The work covered:

- Migrating two PostgreSQL databases from AWS RDS to on-prem PostgreSQL 15
- Deploying a Flutter web frontend, Go REST API, and Keycloak 26.x identity server across a high-availability pair of RHEL 9.4 nodes
- Building an Nginx load balancer with SSL termination
- Configuring a reverse proxy cluster bridging the app network to the port's internal terminal management network
- Deploying a full observability stack — metrics, logs, alerting, and uptime monitoring across all 7 nodes
- Hardening Nginx across on-prem and legacy cloud VMs ahead of a penetration test

---

## Architecture

```
                     ┌──────────────────────────────────┐
                     │          Browser Clients          │
                     │          (HTTPS :443)             │
                     └─────────────┬────────────────────┘
                                   │
                     ┌─────────────▼────────────────────┐
                     │         Load Balancer             │
                     │   Nginx · SSL Termination         │
                     │   Keycloak Admin Proxy            │
                     │   Stream proxy :9090 / :9091      │
                     └──────────┬──────────┬────────────┘
                                │          │
            ┌───────────────────▼──┐  ┌────▼──────────────────┐
            │      App Node 1      │  │      App Node 2 (HA)   │
            │  Flutter (Nginx)     │  │  Flutter (Nginx)       │
            │  Go Backend :8080    │  │  Go Backend :8080      │
            │  Keycloak :8081      │  │  Keycloak :8081        │
            │  Redis :6379         │  │  Redis :6379           │
            └──────────┬───────────┘  └──────┬─────────────────┘
                       │                      │
            ┌──────────▼──────────────────────▼──────────────┐
            │            PostgreSQL 15 — Primary (R/W)        │
            │         + Monitoring Stack (Docker)             │
            └────────────────────┬───────────────────────────┘
                                 │ Streaming replication
            ┌────────────────────▼───────────────────────────┐
            │            PostgreSQL 15 — Replica (Read-only)  │
            └────────────────────────────────────────────────┘

            ┌───────────────────────────────────────────────┐
            │           N4 Proxy Cluster (HA pair)          │
            │   :9090 → Terminal Mgmt System (apex)         │
            │   :9091 → Terminal Mgmt System (billing)      │
            └───────────────────────────────────────────────┘
```

---

## Infrastructure Stack

### On-Premises

| Layer | Technology | Notes |
|---|---|---|
| Frontend | Flutter Web | Custom UAT build target with `--dart-define` env injection |
| Backend | Go (Docker) | Private registry image, multi-node HA |
| Auth | Keycloak 26.x | Custom SPI, `--optimized` build |
| Database | PostgreSQL 15 | Primary + streaming replica |
| Cache | Redis 7 | Password-protected |
| Load Balancer | Nginx | HTTPS, self-signed SSL, upstream round-robin |
| N4 Proxy | Nginx | Cross-subnet reverse proxy to terminal management system |
| Monitoring | Grafana · Prometheus · Loki · Alertmanager · Uptime Kuma | Full stack — see Observability section |
| OS | RHEL 9.4 | SELinux enforcing |
| Containers | Docker + Docker Compose | Private registry, all services containerised |

### Cloud (Legacy — AWS)

| Host | OS | Role |
|---|---|---|
| Cloud VM 1 | Ubuntu 18.04 | Nginx reverse proxy |
| Cloud VM 2 | Ubuntu 16.04 | Portal server + HAProxy |

---

## Server Topology

| Role | Count | Services |
|---|---|---|
| App Nodes (HA pair) | 2 | Keycloak, Go Backend, Redis, Nginx (Flutter frontend) |
| N4 Proxy Nodes (HA pair) | 2 | Nginx reverse proxy to terminal management system |
| Database Primary | 1 | PostgreSQL 15 — read/write + monitoring stack |
| Database Replica | 1 | PostgreSQL 15 — streaming replica, read-only |
| Load Balancer | 1 | Nginx HTTPS, SSL termination, Keycloak admin proxy |
| **Total on-prem nodes** | **7** | |

---

## Service Configuration Deep Dives

### Keycloak — `--optimized` Builds and SPI Configuration

This was the most complex debugging challenge of the project.

**Problem:** A custom SPI (`UserEnabledEventListenerProvider`) was not picking up configuration from `keycloak.conf` at runtime, despite the file containing the correct values.

**Root cause:** Keycloak's `--optimized` flag caches SPI configuration at **image build time**. Runtime changes to `keycloak.conf` are completely ignored.

**Fix:** Pass all SPI config as Docker environment variables, which Keycloak reads regardless of `--optimized` mode:

```yaml
environment:
  KC_SPI_EVENTS_LISTENER_<SPI_NAME>_BACKEND_ENDPOINT: "http://..."
  KC_SPI_EVENTS_LISTENER_<SPI_NAME>_ENCRYPTION_KEY: "..."
```

**Second issue — wrong redirect URLs:** Setting `KC_HOSTNAME` (plain hostname, no protocol) caused Keycloak to generate `http://` redirect URLs instead of `https://`, breaking the entire auth flow.

**Fix:** Remove `KC_HOSTNAME` entirely. Use only:

```yaml
KC_HOSTNAME_URL: https://<load-balancer-ip>
KC_HOSTNAME_ADMIN_URL: https://<load-balancer-ip>
```

**Third issue — `command: start` in docker-compose:** The custom image entrypoint already calls `kc.sh start --optimized`. Adding `command: start` in docker-compose overrides the entrypoint, breaking the startup sequence.

**Fix:** Remove `command: start` from `docker-compose.yml`.

---

### Go Backend — `CREDENTIAL_ENCRYPTION_KEY` Mismatch Across Nodes

**Problem:** Users authenticated successfully on Node 1 but got errors on Node 2. Intermittent failures, hard to reproduce.

**Root cause:** `CREDENTIAL_ENCRYPTION_KEY` was present in Node 1's environment but missing on Node 2. The missing key caused Node 2 to hash credentials against an empty string, producing `e3b0c44...` instead of the correct hash.

**Diagnosis:**
```bash
# Run on each node and compare — output must be identical
echo -n "$CREDENTIAL_ENCRYPTION_KEY" | sha256sum
```

**Fix:** Ensure the environment variable is present and identical across all backend nodes. Treat it like a symmetric encryption key — it must match everywhere.

---

### PostgreSQL Migration — Binary Dump Format Incompatibility

**Problem:** `pg_restore` failed with a format version error when restoring `.dump` files from AWS RDS (PostgreSQL 14) to on-prem PostgreSQL 15.

**Root cause:** The binary dump format version produced by the source server was newer than what the target `pg_restore` supported.

**Fix:** Dump as plain SQL on the source server — plain SQL is version-agnostic and always restores cleanly with `psql`:

```bash
# On source server
pg_dump -U postgres -F p -f /tmp/app_db.sql app_db

# On target server — files must be in /tmp/ (postgres superuser permission requirement)
psql -U postgres app_db < /tmp/app_db.sql
```

---

### Flutter Web Build

The build requires dummy environment files to exist even if empty, and all runtime configuration is injected at build time via `--dart-define`:

```bash
# Create required dummy files
touch assets/config/.env.mobile.development
touch assets/config/.env.mobile.uat
touch assets/config/.env.mobile.production

# Build
flutter build web \
  --target=lib/main_uat.dart \
  --dart-define=KEYCLOAK_BASE_URL=https://<lb-ip>/realms/<realm> \
  --dart-define=API_BASE_URL=https://<lb-ip>/api/v1
```

---

### Nginx Load Balancer — Location Block Ordering

Nginx processes `location` blocks in a specific priority order. Keycloak's `/resources/` and `/admin/` paths must be matched with `^~` prefix blocks **before** any regex-based static asset rules — otherwise the regex catches them first:

```nginx
# Correct order — do NOT reorder these blocks
location ^~ /resources/ {
    proxy_pass http://keycloak_upstream;
    proxy_set_header Host $host;   # must be $host, not $proxy_host
}

location ^~ /admin/ {
    proxy_pass http://keycloak_upstream;
    proxy_set_header Host $host;
}

# Static assets regex — placed AFTER the prefix blocks above
location ~* \.(js|css|png|jpg|woff2)$ {
    root /var/www/html;
}
```

> **`$host` vs `$proxy_host`:** Using `$proxy_host` sends the upstream server's address as the Host header. Keycloak admin console requires the original client hostname (`$host`) to route correctly.

---

## Observability Stack

A full observability stack was deployed on the database primary node, collecting metrics, logs, and uptime data from all 7 on-premises servers.

### Tools Deployed

| Tool | Purpose | Port |
|---|---|---|
| Prometheus | Metrics collection and storage (30-day retention) | 9090 |
| Grafana | Dashboards and visualisation | 3000 |
| Alertmanager | Alert routing via email/webhook | 9093 |
| Loki | Log aggregation from all servers | 3100 |
| Promtail | Log shipping agent (runs on each node) | — |
| cAdvisor | Docker container metrics | 8090 |
| Node Exporter | Host CPU, memory, disk, network metrics | 9100 |
| PostgreSQL Exporter | DB queries, connections, replication lag, locks | 9187 |
| Redis Exporter | Redis memory, connections, hit rate | 9121 / 9122 |
| Nginx Exporter | Request rates, error rates, active connections | 9113 / 9114 |
| Uptime Kuma | Endpoint uptime monitoring with alerting | 3001 |

### Architecture

```
  ┌─────────────────────────────────────────────────────────────┐
  │                   All 7 On-Prem Nodes                       │
  │  Node Exporter :9100  ·  cAdvisor :8090  ·  Promtail       │
  │  + service-specific exporters per node                      │
  └──────────────────────┬──────────────────────────────────────┘
                         │ scrape / ship logs
  ┌──────────────────────▼──────────────────────────────────────┐
  │              Monitoring Stack (DB Primary Node)             │
  │                                                             │
  │  Prometheus ──► Alertmanager ──► Email / Webhook           │
  │       │                                                     │
  │       ▼                                                     │
  │  Grafana ◄── Loki (logs) ◄── Promtail agents               │
  │                                                             │
  │  Uptime Kuma ──► endpoint checks ──► alerts                │
  └─────────────────────────────────────────────────────────────┘
```

### Exporters Installed Per Node

| Node | Exporters |
|---|---|
| App Nodes (.21, .22) | Node Exporter, cAdvisor, Nginx Exporter, Redis Exporter, Promtail |
| N4 Proxy Nodes (.23, .24) | Node Exporter, cAdvisor, Nginx Exporter, Promtail |
| DB Primary (.25) | Node Exporter, cAdvisor, PostgreSQL Exporter, Promtail + full monitoring stack |
| DB Replica (.26) | Node Exporter, PostgreSQL Exporter, Promtail |
| Load Balancer (.27) | Node Exporter, Nginx Exporter, Promtail |

### Nginx Stub Status (Required for Nginx Exporter)

Added to Nginx config on app nodes to expose metrics endpoint:

```nginx
location /stub_status {
    stub_status;
    allow 172.20.0.0/16;   # internal network only
    deny all;
}
```

### Grafana Dashboards

Pre-built dashboards imported from Grafana's dashboard library:

| Dashboard | ID |
|---|---|
| Node Exporter Full (host metrics) | 1860 |
| Docker and system monitoring | 893 |
| PostgreSQL Database | 9628 |
| Redis Dashboard | 763 |
| Nginx Exporter | 12708 |

### Alerting

Alertmanager configured with alert rules covering:

- Host down / unreachable
- High CPU / memory / disk usage
- PostgreSQL connection saturation and replication lag
- Redis memory pressure
- Nginx error rate spikes
- Container restarts

---

## Network & Proxy Design

### Cross-Subnet Routing to Terminal Management System

The app nodes sit on one subnet; the N4/Navis terminal management system lives on a separate internal network. The N4 proxy cluster bridges them.

**SELinux must be configured on RHEL to allow Nginx upstream connections:**
```bash
setsebool -P httpd_can_network_connect 1
```

**Static route to the N4 network must be made persistent** (not just `ip route add`, which is lost on reboot):
```bash
nmcli connection modify <connection-name> +ipv4.routes "<n4-subnet> <gateway>"
nmcli connection up <connection-name>
```

> **Debugging tip:** ICMP (ping) is often blocked by firewall rules between subnets even when TCP on specific ports works fine. Always test with `curl`, `nc -zv`, or `telnet` — not ping.

---

## Security Hardening

### Nginx Hardening (Applied to All Nodes)

| Control | Implementation |
|---|---|
| Hide version | `server_tokens off` |
| TLS versions | TLS 1.2 and 1.3 only |
| Cipher suite | Strong ciphers only; 3DES and RC4 excluded |
| Rate limiting | Separate zones for general, API, and auth endpoints |
| Connection limiting | Per-IP connection cap |
| Security headers | HSTS, `X-Frame-Options`, `X-Content-Type-Options`, `X-XSS-Protection`, `Referrer-Policy` |
| Bad actor blocking | Malicious User-Agents → 444 (silent drop) |
| Exploit path blocking | Common scanner paths → 444 |
| Session tickets | `ssl_session_tickets off` |
| Server info leakage | `proxy_hide_header Server` |

### Penetration Test Readiness

| Node | Status | Residual Findings |
|---|---|---|
| Load Balancer | ✅ Ready | Self-signed certificate |
| App Nodes (×2) | ✅ Ready | Redis not bound to localhost |
| N4 Proxy Nodes (×2) | ✅ Ready | None |
| DB Primary + Replica | ✅ Ready | No TLS on replication stream |
| Cloud VM 1 | ⚠️ Partial | Ubuntu 18.04 EOL · SSL cert expiry imminent |
| Cloud VM 2 | ❌ Findings expected | Ubuntu 16.04 EOL · PHP 7.0 EOL · HAProxy 3DES ciphers |

All on-premises nodes passed the penetration test. Cloud VM findings were tied to EOL operating systems outside the scope of the immediate engagement.

---

## Key Lessons Learned

### 1. Keycloak `--optimized` builds ignore file-based config at runtime
SPI settings must be passed as Docker environment variables. Config in `keycloak.conf` is baked in at image build time and cannot be overridden at runtime.

### 2. `KC_HOSTNAME` without a protocol breaks the HTTPS redirect flow
Remove it entirely. Use `KC_HOSTNAME_URL` and `KC_HOSTNAME_ADMIN_URL` with full `https://` URLs.

### 3. Shared encryption keys must be identical across all nodes
A missing key on one node causes it to hash against an empty string. Intermittent auth failures across a load-balanced cluster are a strong signal to check key parity first.

### 4. `proxy_set_header Host $host` — not `$proxy_host`
For Keycloak and many other proxied services, `$proxy_host` breaks routing by sending the wrong Host header. `$host` preserves what the client originally sent.

### 5. Plain SQL dumps are always portable; binary format versions are not
When migrating across PostgreSQL versions, use `-F p` (plain SQL) and restore with `psql`. Binary format compatibility is not guaranteed.

### 6. Stage dump files in `/tmp/` when restoring as the `postgres` superuser
The `postgres` OS user cannot read files in most home directories. `/tmp/` is always accessible.

### 7. Ping is not a connectivity test across network segments
ICMP is routinely blocked at the firewall level between subnets while TCP passes fine. Use port-level tools to validate connectivity.

### 8. Duplicate `log_format` directives silently prevent Nginx from starting
When multiple files are loaded from `conf.d/`, duplicate `log_format` names cause startup failure with no obvious error. Audit all includes.

### 9. SELinux blocks Nginx upstream connections on RHEL by default
Always run `setsebool -P httpd_can_network_connect 1` when Nginx needs to proxy to upstream services on a RHEL node with SELinux enforcing.

### 10. RHEL aliases `cp` to `cp -i` — this breaks automation scripts
Add `unalias cp 2>/dev/null` at the top of any script that copies files, or scripts will hang waiting for interactive input.

---

## Tech Stack

`Flutter Web` · `Go` · `Keycloak 26.x` · `PostgreSQL 15` · `Redis 7` · `Nginx` · `Docker` · `Docker Compose` · `Prometheus` · `Grafana` · `Loki` · `Alertmanager` · `Uptime Kuma` · `cAdvisor` · `Node Exporter` · `RHEL 9.4` · `AWS EC2` · `N4/Navis`

---

*Infrastructure deployed and maintained independently. Production environment — West Africa.*
