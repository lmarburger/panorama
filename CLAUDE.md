# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Panorama is a comprehensive network monitoring stack that combines cable modem signal quality monitoring with traditional network metrics. It consists of:

- **Surveyor**: Custom Prometheus exporter for Surfboard cable modem signal statistics (SNR, power levels, error counts)
- **Full monitoring stack**: Prometheus (2-year retention), Grafana (port 4242), Blackbox exporter, and Node exporter
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

## Service Ports

- Grafana: http://localhost:4242
- Prometheus: http://localhost:9090
- Blackbox Exporter: http://localhost:9115
- Surveyor: Internal (exposed to Prometheus only)
- Geodesist: Internal (exposed to Prometheus only)
- Node Exporter: Internal (port 9100)

## Key Technical Details

1. **HNAP Authentication**: Uses HMAC-MD5 with specific header requirements for secure modem communication
2. **Metrics Collection**: Parses HTML from modem's signal data page to extract channel statistics
3. **Prometheus Retention**: Configured for 2 years of data retention
4. **Network Probing**: Blackbox exporter configured for ICMP, TCP, and DNS probes
5. **Go Version**: Requires Go 1.22 with minimal external dependencies

## Testing Guidelines

- Always run tests before committing changes to surveyor
- Tests use testify for assertions
- Focus on HNAP client and channel info parser functionality
- Mock HTTP responses for unit tests to avoid dependency on actual modem
