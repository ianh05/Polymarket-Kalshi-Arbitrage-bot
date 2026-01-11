# Trading Endpoints & Safeguards Research

## Polymarket Trading Endpoints

**REST API**: `https://clob.polymarket.com/post_order`
- **Order Type**: FAK (Fill-And-Kill) - fills available liquidity, cancels rest
- **Pre-requisites**:
  - Ethereum private key for EIP-712 typed data signing
  - Wallet address (funder) on Polygon chain (ID: 137)
  - HMAC-SHA256 headers for API authentication
  - Token ID of the market outcome

**WebSocket**: `wss://ws-subscriptions-clob.polymarket.com/ws/market`
- Real-time orderbook updates for price monitoring

**Gamma API**: `https://gamma-api.polymarket.com`
- Market discovery (slugs, token IDs, outcomes)

## Kalshi Trading Endpoints

**REST API**: `https://api.elections.kalshi.com/trade-api/v2/portfolio/orders`
- **Order Type**: IOC (Immediate-Or-Cancel) - same concept as FAK
- **Pre-requisites**:
  - RSA private key (PKCS#1 format) for RSA-PSS SHA-256 signing
  - Kalshi API key ID
  - Ticker symbol of the market

**WebSocket**: `wss://api.elections.kalshi.com/trade-api/ws/v2`
- Real-time price feeds for YES/NO outcomes

## How Dry-Mode Works

Dry-mode is controlled by the `DRY_RUN` environment variable (defaults to `true` for safety).

**Implementation** (`src/main.rs:72`):
```rust
let dry_run = std::env::var("DRY_RUN").map(|v| v == "1" || v == "true").unwrap_or(true)
```

**What happens in dry-mode** (`src/execution.rs:181-190`):
- The system detects arbitrage opportunities normally
- WebSocket connections remain active, monitoring prices
- When an arbitrage is found, it logs: `"[DRY RUN] Would execute..."`
- **No actual orders are placed** - the function returns immediately
- Position tracking and P&L calculations still work for simulation

**API Key Interaction**:
- Keys are still loaded - the system initializes clients with real credentials
- WebSocket connections are real - you receive live market data
- No trading calls made - buy/sell functions are bypassed entirely
- This allows full testing of detection logic without financial risk

## Safeguards in Non-Dry Mode

The project uses a **Circuit Breaker** system (`src/circuit_breaker.rs`):

| Safeguard | Default | Purpose |
|-----------|---------|---------|
| Max Position Per Market | 50,000 contracts | Prevents over-concentration |
| Max Total Position | 100,000 contracts | Limits overall exposure |
| Max Daily Loss | $500 | Halts trading if losses exceed limit |
| Max Consecutive Errors | 5 | Stops after repeated failures |
| Cooldown Period | 300s (5 min) | Auto-reset pause after circuit trips |

**Additional Safeguards**:
- **Auto-Close**: If one leg fills but the other doesn't, automatically closes excess position
- **Minimum Profit Threshold**: Only executes if profit exceeds 0.5%
- **Liquidity Check**: Requires minimum 100 cents per side
- **Deduplication**: Prevents duplicate orders on the same opportunity
- **Test Mode Cap**: Limits positions to 10 contracts when testing

## Fact-Check Against Official Documentation

| Component | Polymarket | Kalshi |
|-----------|------------|--------|
| Base URL | ✅ Correct | ✅ Correct |
| Order Endpoint | ✅ `/post_order` verified | ✅ `/portfolio/orders` verified |
| WebSocket URL | ✅ Matches docs | ✅ Matches docs |
| Order Type | ✅ FAK documented | ✅ IOC documented |
| Auth Method | ✅ EIP-712 correct | ✅ RSA-PSS correct |
| Fees | ⚠️ 0% for most markets | ✅ Formula matches |

**Note on Polymarket Fees**: Standard markets have 0% fees, but 15-minute crypto markets have dynamic taker fees.

## Sources
- [Polymarket CLOB Docs](https://docs.polymarket.com/developers/CLOB/introduction)
- [Polymarket Authentication](https://docs.polymarket.com/developers/CLOB/authentication)
- [Kalshi API Reference](https://docs.kalshi.com/api-reference/orders/create-order)
- [Kalshi Fees](https://help.kalshi.com/trading/fees)
