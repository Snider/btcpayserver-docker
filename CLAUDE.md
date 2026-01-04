# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

BTCPay Server Docker is the deployment infrastructure for BTCPay Server, a self-hosted Bitcoin payment processor. It uses a docker-compose generator system to produce customized deployment configurations based on environment variables.

## Architecture

### Docker-Compose Generation Pipeline

The core architecture is a **fragment-based docker-compose generator**:

1. **Environment Variables** → Define what services to include (crypto, lightning, reverse proxy, add-ons)
2. **`build.sh`** → Runs the docker-compose-generator in a container
3. **Docker-Compose Generator** (`docker-compose-generator/src/`) → C# application that:
   - Reads `crypto-definitions.json` for cryptocurrency configurations
   - Selects relevant fragment files from `docker-compose-generator/docker-fragments/`
   - Merges fragments, resolving dependencies via `required` and `recommended` fields
   - Handles mutual exclusivity via `exclusive` and `incompatible` declarations
   - Outputs to `Generated/docker-compose.generated.yml`

### Key Components

- **Fragments** (`docker-compose-generator/docker-fragments/`): 83 YAML files defining individual services (bitcoin.yml, nginx.yml, opt-add-*.yml, etc.)
- **Crypto Definitions** (`docker-compose-generator/crypto-definitions.json`): Maps crypto codes (btc, ltc, etc.) to their fragment names and lightning implementations
- **Helper Functions** (`helpers.sh`): Core bash functions for btcpay_up, btcpay_down, btcpay_restart, docker_compose_update, etc.

### Fragment System

Each fragment can declare:
- `required`: fragments that must be included
- `recommended`: fragments included unless excluded
- `exclusive`: mutual exclusivity groups
- `incompatible`: fragments that cannot coexist

## Common Commands

### Generate Docker-Compose (Local Development)
```bash
# Set environment variables for your configuration
export BTCPAYGEN_CRYPTO1="btc"
export BTCPAYGEN_REVERSEPROXY="nginx"
export BTCPAYGEN_LIGHTNING="clightning"
./build.sh
```

### Build Generator Locally
```bash
export BTCPAYGEN_DOCKER_IMAGE="btcpayserver/docker-compose-generator:local"
./build.sh
```

### Server Management (on deployed server)
```bash
btcpay-up.sh        # Start services
btcpay-down.sh      # Stop services
btcpay-restart.sh   # Restart services
btcpay-update.sh    # Update to latest
btcpay-clean.sh     # Remove unused images
```

### Node CLI Access
```bash
bitcoin-cli.sh              # Bitcoin Core RPC
bitcoin-lightning-cli.sh    # Core Lightning RPC
bitcoin-lncli.sh           # LND RPC
```

## Key Environment Variables

| Variable | Purpose |
|----------|---------|
| `BTCPAYGEN_CRYPTO1-9` | Cryptocurrencies to support (btc, ltc, etc.) |
| `BTCPAYGEN_LIGHTNING` | Lightning implementation: clightning, lnd, phoenixd, eclair |
| `BTCPAYGEN_REVERSEPROXY` | nginx, traefik, or none |
| `BTCPAYGEN_ADDITIONAL_FRAGMENTS` | Semicolon-separated opt-* fragments |
| `BTCPAYGEN_EXCLUDE_FRAGMENTS` | Fragments to forcefully exclude |
| `BTCPAYGEN_SUBNAME` | Output filename suffix (default: "generated") |
| `BTCPAY_HOST` | Domain name for the server |
| `NBITCOIN_NETWORK` | mainnet, testnet, or regtest |

## Adding New Features

### Creating a Custom Fragment

1. Create `docker-compose-generator/docker-fragments/opt-my-feature.yml`
2. Use `required:` for dependencies
3. Use `exclusive:` to prevent conflicts with similar features
4. Test with: `BTCPAYGEN_ADDITIONAL_FRAGMENTS="opt-my-feature" ./build.sh`

### Adding a New Cryptocurrency

1. Add entry to `docker-compose-generator/crypto-definitions.json`
2. Create the crypto fragment in `docker-compose-generator/docker-fragments/`
3. Optionally create lightning fragments if supported
4. Add CLI wrapper script if needed (e.g., `newcoin-cli.sh`)

## File Structure

```
├── btcpay-setup.sh          # Main installation/configuration script
├── build.sh                 # Triggers docker-compose generation
├── helpers.sh               # Shared bash functions
├── docker-compose-generator/
│   ├── src/                 # C# generator source
│   ├── docker-fragments/    # Service definitions (83 fragments)
│   └── crypto-definitions.json
├── Generated/               # Output directory
│   └── docker-compose.generated.yml
├── Production/              # nginx templates and configs
└── *-cli.sh                 # Node access scripts
```
