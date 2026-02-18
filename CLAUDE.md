# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Prometheus exporter that scrapes Modbus TCP devices and exposes metrics via HTTP. It acts as a proxy: Prometheus sends scrape requests with `target`, `module`, and `sub_target` query parameters, and the exporter connects to the Modbus device, reads registers, and returns Prometheus-formatted metrics.

## Build & Test Commands

```bash
make build              # Build binary using promu (installs promu if missing)
go test ./...           # Run all tests
go test ./modbus/       # Run tests for modbus package only
go test ./config/       # Run tests for config package only
go test -run TestName ./...  # Run a specific test
go test -race ./...     # Run tests with race detector
make format             # Format code (gofmt)
make vet                # Vet code
make lint               # Run golangci-lint (v2.2.1)
```

The build uses [promu](https://github.com/prometheus/promu) (Prometheus utility for building), configured in `.promu.yml`. It downloads automatically via `make build` if not present. The binary is output as `./modbus_exporter`.

## Architecture

### Scrape Flow

1. **`modbus_exporter.go`** (main): HTTP server on `:9602`. Two endpoints:
   - `/metrics` — exporter's own telemetry
   - `/modbus` — scrape handler requiring `target`, `module`, `sub_target` query params

2. **`modbus/modbus.go`** (`Exporter.Scrape`): Opens a TCP connection to the Modbus target, reads registers per the module's metric definitions, parses raw bytes into float64 values, registers them as Prometheus gauges/counters, and returns a `prometheus.Gatherer`.

3. **`config/`**: YAML config loading and validation.
   - `config.go` — Type definitions (`Config`, `Module`, `MetricDef`, `Workarounds`) and validation logic
   - `load.go` — Loads and merges multiple config files (supports glob patterns)

### Key Design Details

- **Modbus address encoding**: The first digit of `address` in config is the function code (1=coils, 2=discrete inputs, 3=holding registers, 4=input registers), remaining digits are the register address. Example: `300022` → function 3 (holding registers), address 22.
- **Data types**: bool, int16/32/64, uint16/32/64, float16/32/64. Register count per read depends on type width.
- **Endianness**: Supports big, little, mixed (byte-swapped words), and yolo (word-swapped bytes). Default is big endian.
- **Retry logic**: On scrape failure, retries up to `scrapeErrorRetryCount` times (default 3) with `scrapeErrorWait` ms delay (default 100ms). Configured per-module in `workarounds`.
- **Metric types**: Only `gauge` and `counter` are supported.
- **Factor scaling**: Optional `factor` field multiplied with raw value (not valid for bool type).

### Test Infrastructure

- `tests/fake_server/` — Standalone Modbus TCP server (using `tbrandon/mbserver`) for manual testing on `127.0.0.1:1502`
- Tests use table-driven patterns with `t.Run` subtests

## Config File Format

See `modbus.yml` for the annotated example. Config is YAML with a top-level `modules` array. The `--config.file` flag supports multiple files and glob patterns.

## Conventions

- Follows Prometheus exporter conventions (Makefile.common from prometheus/prometheus)
- All Go source files must have Apache 2.0 license headers (enforced by `make check_license`)
- Logging uses `go-kit/log` with level-based logging
- CLI flags use `kingpin/v2`
