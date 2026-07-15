# forgectl — Agent Guide

Go CLI for administering Forge network smart contracts on Filecoin. Manages payment channels, provider registration/approval, and exports payment/fault metrics via OTLP for monitoring. Self-contained admin tool: nothing else depends on this repo, so blast radius from changes here is limited to operators and monitoring dashboards.

## Build / Test

```bash
make build      # go build -o forgectl .
make test       # go test ./...
make install    # go install
make clean
```

Go 1.25. Module path: `github.com/fil-forge/forgectl`. There are currently no `_test.go` files in the repo; `make test` compiles packages but exercises nothing — if you add behavior worth keeping, add tests (testify is the convention across Forge Go repos).

## Layout

```
main.go                     # Entry point
cli/
  cmd/
    root.go                 # Cobra root: global flags, viper wiring, subcommand registration
    providers/              # list, get, approve
    payments/               # deposit, status, approveoperator, revokeoperator,
                            #   authorizesession, calculate, settlerail
    metrics/                # payments, faults (OTLP exporters, run-once)
  config/config.go          # Config struct, Load() / LoadReadOnly(), validation
  printer/                  # table.go, json.go — output formatting helpers
pkg/
  services/
    inspector/              # Read-only contract queries (payments, provider registry, service)
    operator/               # Owner-signed writes (e.g. ApproveProvider); embeds inspector
    payer/                  # Payer-signed ops (deposits, session keys, EIP-712 signing)
    chain/                  # Transactor (keystore/private-key auth), tx execution with
                            #   backoff retry, receipt waiting, error decoding
    types/                  # Shared result types
  telemetry/                # OTel meter provider setup + OTLP HTTP metric export
```

Service layering: `inspector.Service` holds the ethclient and contract bindings for read paths; `operator.Service` and `payer.Service` embed it and add a `chain.Transactor` for signed writes. New commands should follow this split — read-only commands use inspector + `config.LoadReadOnly()`, transacting commands use operator/payer + `config.Load()`.

## Configuration

Precedence: flags > `FORGECTL_*` env vars > config file. Env keys use the viper prefix with dashes replaced by underscores (e.g. `FORGECTL_RPC_URL`). Default config file is `./config.yaml` (`--config` overrides); a missing config file is not an error as long as required values arrive via flags/env. See `config.template.yaml` for the full set.

Required contract addresses (all validated as hex addresses): FilecoinWarmStorageService (`service_contract_address`, proxy), PDPVerifier, ServiceProviderRegistry, Payments, USDFC token, SessionKeyRegistry. Two Ethereum keystores: owner (`keystore_path`/`keystore_password`) for contract-owner operations and payer (`payer_keystore_path`/`payer_keystore_password`) for payment operations.

`config.Load()` requires both keystores; `config.LoadReadOnly()` skips keystore validation. Pick the right one per command so read-only queries don't demand wallet credentials.

## Commands

| Command | Purpose |
|---------|---------|
| `providers list` / `get` / `approve` | Query and approve registered storage providers |
| `payments deposit` | Deposit USDFC into the Payments contract |
| `payments status` | Interactive payment status view (bubbletea TUI — the only TUI command) |
| `payments approveoperator` / `revokeoperator` | Manage payment operator approval |
| `payments authorizesession` | Create a session key (SessionKeyRegistry) |
| `payments calculate` | Compute payment amounts |
| `payments settlerail` | Settle a payment rail |
| `metrics payments` | Export payer funds/lockup/runway metrics via OTLP, then exit |
| `metrics faults` | Export missed-proving-period metrics via OTLP, then exit |

Metrics commands are run-once (cron-friendly); scheduled GitHub Actions workflows live in `.github/workflows/metrics-payments.yaml` and `metrics-faults.yaml`. Exported metric names are `forgectl_payer_funds`, `forgectl_payer_lockup_current`, `forgectl_payer_runway_seconds`, `forgectl_rail_missed_periods` — renaming them breaks monitoring dashboards.

## Key Dependencies

- `github.com/fil-forge/filecoin-services/go` — generated contract bindings (`bindings` package). Contract interaction must match the deployed ABIs; when contracts change, bump this dependency rather than hand-editing call sites.
- `github.com/ethereum/go-ethereum` — ethclient, keystore, ABI, EIP-712 signing.
- `github.com/spf13/cobra` + `viper` — CLI and config.
- `github.com/charmbracelet/bubbletea` (+ bubbles, lipgloss) — TUI, used only by `payments status`.
- `go.opentelemetry.io/otel` — metrics; export via OTLP HTTP (`--otlp-endpoint`, `--otlp-insecure`).
- `github.com/cenkalti/backoff/v5` — retry logic in `pkg/services/chain`.

## Gotchas

- The root command's `Use` string reads `forgctl` (missing "e") and the `--config` flag help text mentions `service-operator.yaml`; actual default config name is `config.yaml`. Cosmetic inconsistencies in `cli/cmd/root.go` — don't take help text as ground truth.
- Amounts are USDFC with 18 decimals; formatting helpers live in `cli/printer`.
- Transactions go through `chain.ExecuteContractCall` + receipt waiting with backoff; new write commands should reuse that path rather than calling bindings directly.
