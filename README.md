# Polymarket Trading Bot — Rust Implementation

A professional-grade Rust trading automation system for [Polymarket](https://polymarket.com), enabling sophisticated algorithmic trading strategies for cryptocurrency prediction markets. This enterprise-ready solution provides automated execution, risk management, and backtesting capabilities for 15-minute and 5-minute market cycles across BTC, ETH, SOL, XRP, and sports markets.

---

## Overview

This trading bot implements advanced strategies including dual-limit order placement, arbitrage-style hedging, trailing stop mechanisms, and comprehensive backtesting. Built with Rust for performance, reliability, and memory safety, it offers institutional-quality automation for Polymarket prediction markets.

---

## Key Features

### Trading Strategies

- **Dual Limit Same-Size Strategy (0.45)** — Automated limit order placement at $0.45 for Up/Down positions at market start, with intelligent hedging mechanisms (2-minute, 4-minute, early, and standard triggers)
- **Dual Limit 5-Minute BTC Strategy** — Specialized implementation for BTC 5-minute markets with time-based bands and trailing stop protection
- **Trailing Stop Strategy** — Dynamic position management with trailing stop-loss and profit protection mechanisms
- **Comprehensive Backtesting Engine** — Historical strategy validation using recorded price data
- **Test Suite** — Complete testing utilities for order placement, redemption, merging, allowance management, and market operations

### Technical Capabilities

- High-performance Rust implementation with async/await support
- Real-time market monitoring and order execution
- Automated risk management and position hedging
- Configurable trading parameters and strategy customization
- Extensive logging and monitoring capabilities

---

## Quick Start

### Prerequisites

- Rust 1.70+ (install via [rustup](https://rustup.rs))
- Polymarket API credentials (API key, secret, passphrase)
- Ethereum-compatible private key for signing transactions
- Sufficient USDC balance for trading operations

### Installation

1. **Clone the repository:**
   ```bash
   git clone <repository-url>
   cd polymarket-arbitrage-trading-bot-rust
   ```

2. **Build the project:**
   ```bash
   cargo build --release
   ```

3. **Configure credentials:**
   ```bash
   cp config.example.json config.json
   # Edit config.json with your Polymarket API credentials and private key
   ```

4. **Set on-chain approvals (one-time setup):**
   ```bash
   cargo run --bin test_allowance -- --approve-only
   ```

---

## Available Binaries

| Binary | Description | Use Case |
|--------|-------------|----------|
| `main_dual_limit_045_same_size` | Dual limit order strategy at $0.45 with same-size hedging | Default production strategy |
| `main_dual_limit_045_5m_btc` | BTC 5-minute market specialization | BTC-focused trading |
| `main_trailing` | Trailing stop-loss strategy | Risk-averse position management |
| `backtest` | Historical strategy validation | Strategy development and testing |
| `test_limit_order` | Limit order placement testing | Order execution validation |
| `test_redeem` | Token redemption utilities | Position management |
| `test_merge` | Complete set merging | Portfolio optimization |
| `test_allowance` | Balance and approval management | Account setup |
| `test_sell` | Market sell operations | Exit strategy testing |
| `test_predict_fun` | Prediction logic validation | Strategy development |

---

## Strategy Documentation

### 1. Dual Limit Same-Size Strategy (0.45)

**Binary:** `main_dual_limit_045_same_size`

**Execution Flow:**
1. At market start (first ~5 seconds), places limit buy orders for BTC and enabled ETH/SOL/XRP Up/Down positions at $0.45
2. **Both positions fill:** Strategy completes for that market
3. **Single position fills:** Applies intelligent hedging:
   - **2-minute hedge:** Triggered within 2-3 minutes of market start
   - **4-minute hedge:** Triggered within 4-5 minutes
   - **Early hedge:** Triggered after 5 minutes if unfilled position price ≥ $0.85
   - **Standard hedge:** Triggered after 10 minutes if unfilled position price ≥ $0.85
4. Hedging mechanism: Market buy on unfilled side (same size), cancel unfilled $0.45 limit order

**Low-Price Exit Strategy:**
Two limit sell orders ($0.05/$0.99 or $0.02/$0.99) are placed when:
- Minimum 10 minutes have elapsed since market start
- Market was hedged via 4-minute, early, or standard mechanism (not 2-minute)
- One side's bid price falls below 0.10 (or 0.03 for $0.02/$0.99 path when hedge price < 0.60)

**Usage:**
```bash
# Simulation mode (no real orders)
cargo run --bin main_dual_limit_045_same_size -- --simulation

# Production mode
cargo run --bin main_dual_limit_045_same_size -- --no-simulation
```

### 2. Dual Limit 5-Minute BTC Strategy

**Binary:** `main_dual_limit_045_5m_btc`

Specialized implementation for BTC 5-minute markets featuring:
- Dual limit orders at $0.45
- Time-based trading windows (2-minute: 2-3 min, 3-minute: ≥3 min)
- Price band monitoring with trailing stop (e.g., buy when ask ≥ lowest_ask + 0.03)

**Usage:**
```bash
cargo run --bin main_dual_limit_045_5m_btc -- --config config.json --simulation
cargo run --bin main_dual_limit_045_5m_btc -- --config config.json --no-simulation
```

### 3. Trailing Stop Strategy

**Binary:** `main_trailing`

**Execution Flow:**
1. Monitors market prices until one token's price falls below $0.45
2. Initiates trailing stop on the identified token (with $0.45 cap)
3. After first buy, implements stop-loss and trailing stop mechanisms for the opposite token

**Usage:**
```bash
cargo run --bin main_trailing -- --simulation
cargo run --bin main_trailing -- --no-simulation
```

### 4. Backtesting Engine

**Binary:** `backtest`

Replays dual-limit strategy on historical price data stored in `history/market_*_prices.toml`:
- Simulates limit buy orders at $0.45
- Models order fills based on historical prices
- Applies hedging logic
- Calculates profit and loss (PnL)

**Usage:**
```bash
cargo run --bin backtest -- --backtest
```

---

## Configuration

### Configuration File Structure

The `config.json` file contains two main sections:

#### Polymarket Configuration (`polymarket`)

| Field | Required | Description |
|-------|----------|-------------|
| `gamma_api_url` | Yes | Polymarket Gamma API endpoint |
| `clob_api_url` | Yes | Polymarket CLOB API endpoint |
| `api_key` | Yes | Polymarket API key (UUID format) |
| `api_secret` | Yes | Polymarket API secret |
| `api_passphrase` | Yes | Polymarket API passphrase |
| `private_key` | Yes | Ethereum-compatible private key (hex format, with or without 0x prefix) |
| `proxy_wallet_address` | No | Proxy wallet address (for POLY_PROXY or GNOSIS_SAFE signature types) |
| `signature_type` | No | Signature type: `0` = EOA, `1` = POLY_PROXY, `2` = GNOSIS_SAFE |

#### Trading Configuration (`trading`)

| Field | Description |
|-------|-------------|
| `check_interval_ms` | Market monitoring interval in milliseconds |
| `fixed_trade_amount` | Base trade size in USDC |
| `enable_btc_trading` | Enable BTC market trading |
| `enable_eth_trading` | Enable ETH market trading |
| `enable_solana_trading` | Enable SOL market trading |
| `enable_xrp_trading` | Enable XRP market trading |
| `dual_limit_price` | Limit order price (typically 0.45) |
| `dual_limit_shares` | Shares per limit order |
| `dual_limit_hedge_after_minutes` | Minutes before standard hedge trigger |
| `dual_limit_hedge_price` | Price threshold for hedge trigger |
| `dual_limit_early_hedge_minutes` | Minutes before early hedge trigger |
| `trailing_stop_point` | Trailing stop distance (e.g., 0.03) |
| `trailing_shares` | Shares for trailing stop orders |

### Command-Line Options

- `--simulation` — Run in simulation mode (no real orders placed)
- `--no-simulation` — Run in production mode (real orders executed)
- `--config <path>` — Specify custom configuration file path (default: `config.json`)

---

## Testing Utilities

### Order Management
```bash
# Place a test limit order
cargo run --bin test_limit_order -- --price-cents 60 --shares 10

# Test market sell
cargo run --bin test_sell
```

### Position Management
```bash
# List redeemable positions
cargo run --bin test_redeem -- --list

# Redeem all winning positions
cargo run --bin test_redeem -- --redeem-all

# Merge complete sets to USDC
cargo run --bin test_merge -- --merge
```

### Account Management
```bash
# Check balance and allowance
cargo run --bin test_allowance -- --list

# Set on-chain approval (one-time setup)
cargo run --bin test_allowance -- --approve-only
```

### Strategy Development
```bash
# Test prediction logic
cargo run --bin test_predict_fun
```

---

## Security Best Practices

### Credential Management

- **Never commit `config.json`** containing real API keys or private keys to version control
- Use environment variables or secure credential management systems in production
- Rotate API credentials regularly
- Use separate API keys for testing and production environments

### Operational Security

- Start with simulation mode to validate strategy behavior
- Use small trade sizes during initial testing
- Monitor logs and account balances continuously
- Set appropriate stop-loss and position size limits
- Implement rate limiting and error handling for production deployments

### Private Key Security

- Store private keys securely using hardware wallets or key management systems when possible
- Use proxy wallets (POLY_PROXY or GNOSIS_SAFE) for enhanced security
- Never share private keys or API credentials
- Regularly audit access logs and trading activity

---

## Monitoring and Logging

The bot provides comprehensive logging for:
- Market monitoring and price updates
- Order placement and execution status
- Position management and hedging decisions
- Error conditions and retry attempts
- Performance metrics and PnL tracking

Logs are written to standard output and can be redirected to files for analysis:
```bash
cargo run --bin main_dual_limit_045_same_size -- --no-simulation 2>&1 | tee trading.log
```

---

## Performance Considerations

- **Async Architecture:** Built on Tokio runtime for high-concurrency market monitoring
- **Memory Efficiency:** Rust's zero-cost abstractions ensure minimal memory footprint
- **Network Optimization:** Efficient API request handling with connection pooling
- **Error Recovery:** Robust retry mechanisms and error handling for network issues

---

## Troubleshooting

### Common Issues

**Authentication Failures:**
- Verify API credentials are correct and in UUID format
- Ensure private key is valid hex format
- Check network connectivity to Polymarket APIs

**Order Execution Failures:**
- Verify sufficient USDC balance
- Check on-chain approvals are set (`test_allowance --approve-only`)
- Confirm market is active and accepting orders

**Symbol Loading Errors:**
- Ensure `lib/libclob_sdk.so` is present and has correct permissions
- Verify library was built with FFI features enabled
- Check library path in error messages

---

## Development

### Building from Source

```bash
# Debug build
cargo build

# Release build (optimized)
cargo build --release

# Run tests
cargo test

# Check code formatting
cargo fmt --check

# Run linter
cargo clippy
```

### Project Structure

```
polymarket-arbitrage-trading-bot-rust/
├── src/
│   ├── api.rs              # Polymarket API client
│   ├── clob_sdk.rs         # CLOB SDK FFI bindings
│   ├── monitor.rs          # Market monitoring
│   ├── trader.rs           # Trading logic
│   ├── simulation.rs       # Simulation engine
│   └── bin/                 # Binary entry points
├── lib/                     # Shared libraries
├── config.example.json      # Configuration template
└── README.md                # This file
```

---

## Support and Contributions

For questions, feature requests, or support inquiries, please refer to the project's issue tracker or contact the maintainers.

---

## License

[Specify your license here]

---

## Disclaimer

This software is provided for educational and research purposes. Trading cryptocurrency prediction markets involves substantial risk of loss. Users are responsible for understanding the risks involved and for compliance with all applicable laws and regulations. The authors and maintainers assume no liability for trading losses or damages resulting from the use of this software.

---

## Keywords

**Primary:** Polymarket trading bot, Polymarket arbitrage bot, Polymarket copytrading bot, Polymarket crypto bot, Polymarket sports bot, automated Polymarket trading

**Secondary:** Prediction markets bot, crypto prediction markets, dual limit order strategy, trailing stop trading bot, BTC trading bot, ETH trading bot, SOL trading bot, XRP trading bot, Rust trading bot, open source Polymarket bot, automated Polymarket strategies, Polymarket market making bot, Polymarket high-frequency trading, Polymarket scalping bot
