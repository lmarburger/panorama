# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Panorama is a comprehensive network monitoring stack that combines cable modem signal quality monitoring with traditional network metrics. It consists of:

- **Surveyor**: Custom Prometheus exporter for SURFboard cable modem signal statistics (SNR, power levels, error counts)
- **Full monitoring stack**: Prometheus (2-year retention), Grafana (port 4242), Blackbox exporter, and Node exporter (Docker VM + native macOS)
- **Docker Compose orchestration**: Easy deployment of the entire monitoring infrastructure

The goal is to complement network latency graphs (like Smokeping) with physical layer signal metrics to diagnose whether network issues are due to cable conditions or other factors.

## Architecture

### Component Structure

In this repo:
- `prometheus/`: Prometheus configuration and alert rules
- `blackbox.yml`: Blackbox exporter configuration for network probing
- `docker-compose.yml`: Main orchestration file for the full stack
- `scripts/`: helper scripts (see below); use these rather than raw `curl`

In sibling repos (see Repository Layout):
- `../surveyor/`: Go-based Prometheus exporter for cable modem metrics
  - Uses HNAP (Home Network Administration Protocol) with HMAC-MD5 authentication
  - Targets an Arris SURFboard DOCSIS 3.1 modem at `https://192.168.100.1/HNAP1/`
- `../geodesist/`: Go-based Prometheus exporter for AmpliFi wifi usage metrics

### Surveyor Architecture
Surveyor polls the modem in a background loop (every 30s, backing off to 5m while failing) and `/metrics` serves the latest result. **A scrape never touches the modem**, so reading surveyor's `/metrics` by hand is free. `../surveyor/CLAUDE.md` has the file layout and the modem quirks found by probing it.

## Development Commands

All paths below are relative to this repo (`smokeping/panorama`).

### Full Stack Operations
```bash
scripts/stack up          # docker compose up -d
scripts/stack build       # rebuild the two Go service images
scripts/stack down
scripts/stack logs -f surveyor
scripts/stack health      # run this after any change
```

### Service Development
```bash
cd ../surveyor            # or ../geodesist
go run main.go            # runs locally; -addr :8080 to change the port
go build -o surveyor
```

### Testing
```bash
cd ../surveyor && go test ./...
go test -v -cover ./...
```

`go test` runs vet as part of the build, so a vet finding fails the run outright.
`../geodesist` has no tests; verify it with `go build ./... && go vet ./...` plus
`scripts/prom query 'amplifi_clients_count'`.

### Code Quality
```bash
cd ../surveyor && go fmt ./... && go vet ./...
staticcheck ./...
```

## Repository Layout

Three independent public repos live side by side under a plain `smokeping/` workspace directory. None of them is nested inside another.

```
smokeping/
  panorama/    lmarburger/panorama    compose, prometheus/, blackbox.yml, scripts/
  surveyor/    lmarburger/surveyor    cable modem exporter
  geodesist/   lmarburger/geodesist   AmpliFi exporter
```

`docker-compose.yml` builds the two services from `../surveyor` and `../geodesist`, so **both siblings must be checked out for the stack to build.** Run `scripts/setup` to see what is missing, or `scripts/setup --apply` to clone them and create the data volumes.

Changes to the Go services are commits in their own repos, not this one. There is deliberately no submodule pointer: the repos are independent, and the tradeoff is that this repo does not record which service commit is deployed.

### Why not nested

The services used to be checked out *inside* this repo and hidden with bare `.gitignore` entries. That layout had a silent data-loss footgun: `git clean -xdf` here would have deleted both service repos, uncommitted work included, because ignored directories are fair game for clean. Siblings make that impossible.

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

scripts/stack health                   # full post-change verification
scripts/stack ports                    # prove Grafana is not exposed beyond loopback + LAN
scripts/stack versions [latest]        # running versions; `latest` checks upstream
scripts/stack deps                     # Go module updates in both service repos
scripts/stack ps | logs | pull | build | up | restart | down

scripts/setup [--apply]                # check/fix sibling checkouts, volumes, .env
```

**Run `scripts/stack health` after any change.** It checks container state, all 19 scrape targets, Grafana auth, that surveyor's last modem poll succeeded (`surveyor_modem_up`), and that geodesist emits real metrics. A target being `up` just means `/metrics` answered; it does not mean surveyor reached the modem.

`up`, `restart`, and `down` are deliberately **not** in the permission allowlist, since they interrupt collection. Everything else in `scripts/stack` runs unprompted.

`scripts/grafana panels <uid>` is the fastest way to find which query drives a panel. `scripts/prom targets down` is the fastest health check.

Both embed Python inside single-quoted shell strings, so **the embedded Python must not contain a single quote**. Use `"..."` and `.format()` rather than f-strings with nested quotes.

### Grafana Auth

Basic auth `admin:panorama`. `scripts/grafana` sources `.env` and honors `GRAFANA_ADMIN_PASSWORD`, falling back to `panorama`. Override the endpoint or credential with `GRAFANA_URL` / `GRAFANA_AUTH`.

`GF_SECURITY_ADMIN_PASSWORD` in `docker-compose.yml` **only applies when the `grafana` volume is first created** (verified empirically). To change the password on an existing volume:

```bash
scripts/grafana api /api/admin/users/1/password \
  -X PUT -H 'Content-Type: application/json' -d '{"password":"<new>"}'
```

Grafana shows an "update your password" prompt while the admin password is literally `admin`. There is no config option to suppress it; the only fix is to use a different password. The compose default is `panorama` rather than unset precisely so a fresh clone comes up already past that prompt.

The credential is in a public repo on purpose. Grafana is bound to loopback and one LAN address, so reaching it means already being on the home network. The residual risk is joining another network that also uses AmpliFi's default `192.168.119.0/24` range and being handed `.4`, which would activate the LAN binding there.

### Known Dashboards

| Name | UID |
|------|-----|
| Smokeping | `adgn11db97ym8b` |
| System | `ad9scbj` |

Dashboards live in Grafana, which keeps their version history (dashboard settings, then Versions). They are deliberately not checked in to git, so edit them in the UI or with `scripts/grafana save`, and roll back from Grafana's version list.

## Metrics Reference

| Exporter | Metrics |
|----------|---------|
| surveyor (`job="surveyor"`) | `surveyor_downstream_{snr_db,power_dbmv,frequency_hertz,locked}`, `surveyor_downstream_{corrected,uncorrectable}_codewords_total`, `surveyor_upstream_{power_dbmv,frequency_hertz,symbol_rate,locked}`, labelled by `channel_id`, `frequency_mhz`, and `modulation` or `type`. Health: `surveyor_modem_up`, `surveyor_last_success_timestamp_seconds`, `surveyor_poll_errors_total{stage}`, `surveyor_poll_duration_seconds`, `surveyor_modem_info{firmware}` |
| geodesist (`job="geodesist"`) | `amplifi_clients_count`, `amplifi_happiness_score`, `amplifi_signal_quality`, `amplifi_global_rx_bitrate`, `amplifi_global_tx_bitrate`, `amplifi_total_rx_bytes`, `amplifi_total_tx_bytes`. Per-host ones labelled by `host` |

Surveyor's metrics were renamed on 2026-09-29. History before that is under the old unprefixed names (`snratio`, `power_level`, `frequency`, `correctable_count`, `uncorrectable_count`, `hmac_collect_duration_seconds`, labelled only by `channel_id`). The dashboard queries both, so the old half of each query can be deleted once that data ages out of the 2-year retention.

geodesist does **not** namespace its metrics with the exporter name: guessing `geodesist_*` returns nothing.

Blackbox probes are `probe_*` under jobs `icmp`, `tcp`, `dns`. Use `scripts/prom metrics <substring>` to search, and `scripts/prom labels <metric>` for the label sets.

## Environment

`docker-compose.yml` reads these from `.env` (gitignored, and unreadable to Claude):

| Variable | Default | Purpose |
|----------|---------|---------|
| `AMPLIFI_PASSWORD` | none, required | geodesist router login |
| `SURVEYOR_MODEM_PASSWORD` | none, required | surveyor modem login; surveyor exits at startup without it |
| `AMPLIFI_ROUTER_ADDR` | `http://192.168.119.1` | router URL |
| `GRAFANA_ADMIN_PASSWORD` | `panorama` | admin password; also read by `scripts/grafana` |
| `GRAFANA_LAN_IP` | `192.168.119.4` | the LAN address Grafana binds to |

## Gotchas

- **The compose project name is pinned to `panorama` and the data volumes are pinned by name.** Do not remove either. Compose derives the project name from the directory by default and prefixes volume names with it, so moving or renaming the checkout would have silently reparented `surveyor_grafana` and `surveyor_prometheus` and started the stack against empty volumes. It looks exactly like losing 200+ days of history. The volumes are also declared `external`, so `docker compose down -v` cannot destroy them.

- **`prometheus/prometheus.yml` is a single-file bind mount.** Editors (and Claude's Edit tool) save by replacing the file, which leaves the container holding the old, deleted inode: the file vanishes inside the container and a SIGHUP reloads nothing. After editing it, `scripts/stack restart prometheus`.
- **Downstream channels with `modulation="Unknown"` are not carrying data.** Since the ISP went to 32 channels on 2026-09-21, the lowest ones (555 to 567 MHz) report SNR 0 and power around -40 dBmV. The SNR and power percentile panels filter them out; the per-channel panels and the "unusable downstream channels" stat show them.
- **The "memory %" and "cpu %" panels under the Smokeping dashboard's `surveyor` row do not measure surveyor.** They query `node_memory_*` / `node_cpu_*` from `job="node"`, which is the OrbStack Linux VM the containers run in. Surveyor's own footprint is ~13 MB RSS. Use `process_resident_memory_bytes{job="surveyor"}` for the service itself.
- Docker here is **OrbStack**, not Docker Desktop. It balloons VM memory to track actual usage instead of pre-allocating, so a high in-VM memory percentage is a weak pressure signal. Prefer `rate(node_vmstat_pgmajfault[5m])` or `node_memory_SwapFree_bytes`.
- `node_memory_MemFree_bytes` and friends are **Linux-only** and exist only for `job="node"`. The macOS host exporter (`job="macos"`) uses different names: `node_memory_total_bytes`, `node_memory_free_bytes`, `node_memory_wired_bytes`.
- Claude Code cannot read or write `~/Library` (macOS TCC blocks it even with the sandbox disabled), so `brew services` commands must be run by the user. Homebrew also needs `HOMEBREW_CACHE` relocated to a writable path to run at all under the sandbox.
- Claude Code cannot read `.env` or `.env.example`. Ask the user to edit them, and have scripts source `.env` at runtime instead.
- Every `git` command in these repos prints `error: fsmonitor_ipc__send_query: unspecified error on '.git/fsmonitor--daemon.ipc'` to stderr. It is harmless noise from the fsmonitor daemon and does not indicate the command failed. Check the exit code, not stderr.
- The macOS host exporter is a Homebrew service (`brew services start node_exporter`), already enabled at login. If `job="macos"` goes down, it is that service, not the container stack. The user has to restart it; see the `~/Library` note above.
- `scripts/stack deps` filters `go list -m -u all` down to modules actually named in `go.mod`. The unfiltered list permanently shows four upgradeable modules (`golang/protobuf`, `klauspost/compress`, `x/oauth2`, `x/sync`) that are test dependencies of dependencies. `go get -u ./...` will never touch them, so do not chase them.

## Upgrading

Image versions are pinned in `docker-compose.yml` so a restart cannot silently pull a new major version.

```bash
docker compose pull                    # respects the pinned tags
docker run --rm --entrypoint /bin/prometheus prom/prometheus --version   # check what latest is
```

To move to a newer release, back up dashboards first (`scripts/grafana get <uid> backups/<name>.json`), bump the tag, then `docker compose up -d --build` and verify with `scripts/prom targets`.

## Key Technical Details

1. **HNAP Authentication**: Uses HMAC-MD5 with specific header requirements for secure modem communication
2. **Metrics Collection**: One bundled HNAP `GetMultipleHNAPs` request returns downstream, upstream, and software info as `^`-delimited records
3. **Prometheus Retention**: Configured for 2 years of data retention
4. **Network Probing**: Blackbox exporter configured for ICMP, TCP, and DNS probes
5. **Go Version**: Both modules declare `go 1.26.0`. Docker builds use `golang:1.27-trixie` and run on `debian:trixie-slim`.

## Testing Guidelines

- Always run tests before committing changes to surveyor
- Tests use testify for assertions
- Surveyor's tests include a fake TLS modem (`surveyor/client_test.go`) that reproduces the real login and session behavior, so they never need the actual modem
- `geodesist` has **no tests at all**. Verify it with `go build ./... && go vet ./...`, and confirm it is live with `scripts/prom query 'amplifi_clients_count'`.
- `go test` runs vet as part of the build, so a vet finding fails the test run rather than merely warning. Go 1.24 added the non-constant format string check, which is worth knowing when bumping the toolchain.
