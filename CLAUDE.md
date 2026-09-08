# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Panorama is a comprehensive network monitoring stack that combines cable modem signal quality monitoring with traditional network metrics. It consists of:

- **Surveyor**: Custom Prometheus exporter for Surfboard cable modem signal statistics (SNR, power levels, error counts)
- **Full monitoring stack**: Prometheus (2-year retention), Grafana (port 4242), Blackbox exporter, and Node exporter (Docker VM + native macOS)
- **Docker Compose orchestration**: Easy deployment of the entire monitoring infrastructure

The goal is to complement network latency graphs (like Smokeping) with physical layer signal metrics to diagnose whether network issues are due to cable conditions or other factors.

## Architecture

### Component Structure
- `/surveyor/`: Go-based Prometheus exporter for cable modem metrics
  - Uses HNAP (Home Network Administration Protocol) with HMAC-MD5 authentication
  - Targets Surfboard SB6141 modem at `https://192.168.100.1/HNAP1/`
  - Exposes metrics at `/metrics` endpoint
- `/geodesist/`: Go-based Prometheus exporter for AmpliFi wifi usage metrics
  - Exposes metrics at `/metrics` endpoint
- `/prometheus/`: Prometheus configuration and alert rules
- `/blackbox.yml`: Blackbox exporter configuration for network probing
- `/docker-compose.yml`: Main orchestration file for the full stack

### Surveyor Architecture
The surveyor codebase follows clean Go architecture:
- `main.go`: HTTP server setup and Prometheus metrics endpoint
- `surveyor/hnap.go`: HNAP client for secure modem communication
- `surveyor/channelinfo.go`: Parser for channel signal data
- `surveyor/report.go`: Prometheus collector implementation

## Development Commands

### Full Stack Operations
```bash
# Build and run entire monitoring stack
docker compose up --build

# Run in detached mode
docker compose up -d

# Stop all services
docker compose down

# View logs
docker compose logs -f surveyor
```

### Surveyor Development
```bash
# Run locally
cd surveyor
go run main.go

# Build binary
go build -o surveyor

# Run with custom address
go run main.go -addr :8080
```

### Testing
```bash
# Run all tests from root
cd surveyor && go test ./...

# Verbose with coverage
go test -v -cover ./...

# Specific package tests
go test -v ./surveyor/...
```

### Code Quality
```bash
# Format code
cd surveyor && go fmt ./...

# Check for common mistakes
go vet ./...

# Static analysis
staticcheck ./...
```

## Repository Layout

This repo tracks only the stack configuration: `docker-compose.yml`, `prometheus/`, `blackbox.yml`, `scripts/`, and this file.

`surveyor/` and `geodesist/` are **separate git repositories** checked out inside this one, and both are listed in `.gitignore`. Changes to the Go services are commits in `lmarburger/surveyor` and `lmarburger/geodesist`, not here. Because the ignore entries are bare directory names, any new file added under either directory is invisible to this repo's git.

## Service Ports

Only Grafana is published to the host. Everything else talks over the compose network.

- Grafana: http://localhost:4242 and http://192.168.119.4:4242 (container port 3000)
- Prometheus: **no host binding**, container-internal on 9090. Use `scripts/prom`.
- Blackbox Exporter: internal, 9115
- Surveyor: internal, 80
- Geodesist: internal, 80
- Node Exporter (Docker VM): internal, 9100, job `node`
- Node Exporter (macOS host): http://localhost:9100, native Homebrew install, job `macos`

Grafana is deliberately **not** published on `0.0.0.0`. It is bound to loopback plus the Mac's static AmpliFi lease (`192.168.119.4`, overridable with `GRAFANA_LAN_IP`), so it cannot be reached from any other network the laptop joins. Two consequences worth knowing:

- Off the home LAN nothing breaks. OrbStack accepts a binding for an address the host does not currently hold, so the container still starts and stays reachable on `localhost:4242`; the LAN address just serves nothing until the Mac is home. Verified by binding a throwaway container to `203.0.113.7` (TEST-NET-3): it started cleanly and was reachable on no interface, confirming the binding is honored strictly rather than silently widened to `0.0.0.0`.
- The Mac has a Tailscale address (`100.x.y.z`). Adding it as a third binding would give authenticated remote access without exposing Grafana to whatever local network the laptop is on.

## Helper Scripts

Use these instead of hand-written `curl`. They are allowlisted in `.claude/settings.json`, so they run without a permission prompt, and they encapsulate the awkward parts (Prometheus having no host port, URL-encoding PromQL, reading the Grafana credential).

```bash
scripts/grafana list                   # dashboards: uid + title
scripts/grafana get <uid> [outfile]    # fetch dashboard JSON
scripts/grafana save <file> [--label]  # save back (overwrite: true)
scripts/grafana panels <uid>           # panel titles, types, and PromQL exprs
scripts/grafana api <path> [curl args] # raw call, e.g. api /api/health

scripts/prom query '<promql>'          # instant query
scripts/prom range '<promql>' 24h 15m  # range query with summary stats
scripts/prom metrics [substring]       # list metric names
scripts/prom labels <metric>           # label sets for a metric
scripts/prom targets [up|down]         # scrape target health
scripts/prom api <path>                # raw call
```

`scripts/grafana panels <uid>` is the fastest way to find which query drives a panel. `scripts/prom targets down` is the fastest health check.

Both embed Python inside single-quoted shell strings, so **the embedded Python must not contain a single quote**. Use `"..."` and `.format()` rather than f-strings with nested quotes.

### Grafana Auth

Basic auth `admin:smokeping`. `scripts/grafana` sources `.env` and honors `GRAFANA_ADMIN_PASSWORD`, falling back to `smokeping`. Override the endpoint or credential with `GRAFANA_URL` / `GRAFANA_AUTH`.

`GF_SECURITY_ADMIN_PASSWORD` in `docker-compose.yml` **only applies when the `grafana` volume is first created** (verified empirically). To change the password on an existing volume:

```bash
scripts/grafana api /api/admin/users/1/password \
  -X PUT -H 'Content-Type: application/json' -d '{"password":"<new>"}'
```

Grafana shows an "update your password" prompt while the admin password is literally `admin`. There is no config option to suppress it; the only fix is to use a different password.

### Known Dashboards

| Name | UID |
|------|-----|
| Smokeping | `adgn11db97ym8b` |
| System | `ad9scbj` |

## Gotchas

- **The "memory %" and "cpu %" panels under the Smokeping dashboard's `surveyor` row do not measure surveyor.** They query `node_memory_*` / `node_cpu_*` from `job="node"`, which is the OrbStack Linux VM the containers run in. Surveyor's own footprint is ~13 MB RSS. Use `process_resident_memory_bytes{job="surveyor"}` for the service itself.
- Docker here is **OrbStack**, not Docker Desktop. It balloons VM memory to track actual usage instead of pre-allocating, so a high in-VM memory percentage is a weak pressure signal. Prefer `rate(node_vmstat_pgmajfault[5m])` or `node_memory_SwapFree_bytes`.
- `node_memory_MemFree_bytes` and friends are **Linux-only** and exist only for `job="node"`. The macOS host exporter (`job="macos"`) uses different names: `node_memory_total_bytes`, `node_memory_free_bytes`, `node_memory_wired_bytes`.
- Claude Code cannot read or write `~/Library` (macOS TCC blocks it even with the sandbox disabled), so `brew services` commands must be run by the user. Homebrew also needs `HOMEBREW_CACHE` relocated to a writable path to run at all under the sandbox.
- Claude Code cannot read `.env` or `.env.example`. Ask the user to edit them, and have scripts source `.env` at runtime instead.

## Upgrading

Image versions are pinned in `docker-compose.yml` so a restart cannot silently pull a new major version.

```bash
docker compose pull                    # respects the pinned tags
docker run --rm --entrypoint /bin/prometheus prom/prometheus --version   # check what latest is
```

To move to a newer release, back up dashboards first (`scripts/grafana get <uid> backups/<name>.json`), bump the tag, then `docker compose up -d --build` and verify with `scripts/prom targets`.

## Key Technical Details

1. **HNAP Authentication**: Uses HMAC-MD5 with specific header requirements for secure modem communication
2. **Metrics Collection**: Parses HTML from modem's signal data page to extract channel statistics
3. **Prometheus Retention**: Configured for 2 years of data retention
4. **Network Probing**: Blackbox exporter configured for ICMP, TCP, and DNS probes
5. **Go Version**: Both modules declare `go 1.26.0`. Docker builds use `golang:1.27-trixie` and run on `debian:trixie-slim`.

## Testing Guidelines

- Always run tests before committing changes to surveyor
- Tests use testify for assertions
- Focus on HNAP client and channel info parser functionality
- Mock HTTP responses for unit tests to avoid dependency on actual modem
- `geodesist` has **no tests at all**. Verify it with `go build ./... && go vet ./...`, and confirm it is live with `scripts/prom query 'amplifi_clients_count'`.
- `go test` runs vet as part of the build, so a vet finding fails the test run rather than merely warning. Go 1.24 added the non-constant format string check, which is worth knowing when bumping the toolchain.
