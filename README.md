# Pons Sniper Bot

Low-latency, self-custodial launch sniper for **Pons V1 and Pons V2** on Robinhood Chain.

The bot watches Pons contracts directly, evaluates each launch against your rules, simulates the trade, and submits a protected buy without waiting for a website, chart, or third-party token feed. V2 execution is **opening-tax aware**: being first is not useful if the opening tax consumes the trade.

> **Status:** Active development. Contract addresses, ABIs, and fee behavior must be verified against the current Pons deployment before live trading.

## Why this bot exists

Most generic snipers optimize only for transaction speed. Pons needs two different execution strategies:

- **Pons V1:** detect the launch and submit as early as safely possible.
- **Pons V2:** detect immediately, but enter only when the address-specific opening tax and expected token output meet your limits.

On V2, the fastest transaction can be the worst transaction. The engine therefore separates **detection speed** from **entry timing**.

## Core features

- Pons V1 and V2 launch detection
- Direct Robinhood Chain WebSocket log stream
- Version-specific contract adapters and ABI decoding
- Tax-aware V2 entry engine
- Configurable tax, slippage, gas, price-impact, and spend limits
- Prebuilt transaction templates and locally managed nonces
- Preflight `eth_call` simulation before submission
- Minimum-output protection on every supported trade
- Optional creator, quote-token, symbol, and liquidity filters
- Honeypot and sell-path simulation where the contract permits it
- Automatic take-profit, stop-loss, trailing-stop, and timed exits
- Position and transaction journal with explorer links
- Dry-run and paper-trading modes
- Telegram or webhook notifications
- Multi-wallet support with isolated queues and nonce state
- Automatic reconnect, provider failover, and duplicate-event protection

## V1 and V2 execution

| Capability | Pons V1 | Pons V2 |
| --- | --- | --- |
| Launch detection | Direct factory logs | Direct factory logs |
| Primary entry goal | Earliest safe inclusion | Earliest economically acceptable inclusion |
| Opening-tax handling | Standard configured checks | Live tax/output polling and threshold trigger |
| Entry trigger | Launch event + successful simulation | Launch event + acceptable tax/output + successful simulation |
| Price protection | Slippage and minimum output | Tax cap, slippage, price impact, and minimum output |
| Contract calls | V1 adapter | V2 curve/router adapter |

### Pons V1 mode

V1 mode is optimized for rapid event-to-transaction execution:

1. Subscribe to the verified V1 factory event.
2. Decode the token and curve/router addresses from the receipt or event.
3. Apply local filters without additional third-party API requests.
4. Simulate the prepared transaction against the latest block.
5. Sign locally and broadcast when every limit passes.

### Pons V2 tax-aware mode

Pons V2 may apply a very high opening tax that decays shortly after launch. The exact schedule can change and should be read or inferred from current on-chain state rather than hard-coded.

The V2 engine detects the launch immediately, then repeatedly evaluates:

- effective buy tax for the trading address;
- expected tokens received after fees and tax;
- curve price and price impact;
- estimated gas and maximum total cost;
- sell-path availability, when simulation is possible;
- whether the configured entry deadline has expired.

It broadcasts only when all configured limits pass. If the tax never reaches the required level within the entry window, the bot skips the launch.

```text
Launch detected
      │
      ▼
Resolve V1/V2 adapter
      │
      ▼
Run filters and simulation
      │
      ├── V1: submit when safe
      │
      └── V2: poll tax/output → threshold met → re-simulate → submit
```

## Why it is fast

The bot does not claim a guaranteed block position or universal latency advantage. Its design removes avoidable delay from the execution path:

1. **On-chain detection** — listens to Pons factory logs instead of polling the Pons website, a chart, or an indexer.
2. **Persistent WebSocket connections** — avoids creating a new RPC connection after a launch appears.
3. **Prebuilt calldata** — static transaction fields are prepared before the trigger.
4. **Local signing** — transactions are signed in-process; no browser confirmation is required for an armed strategy.
5. **Warm nonce state** — each wallet has a serialized nonce queue to prevent last-second nonce lookups and collisions.
6. **Local filtering** — creator lists, sizing rules, and token constraints run in memory.
7. **Parallel RPC reads** — independent safety and quote checks are requested concurrently.
8. **Provider race/failover** — optional submission to multiple trusted RPC endpoints reduces dependency on one gateway.
9. **V2 adaptive polling** — high-frequency checks run only during the short opening window, then stop automatically.
10. **No blind first-block buy** — V2 optimizes time-to-valid-entry, not merely time-to-broadcast.

Actual results depend on RPC quality, network congestion, validator ordering, contract state, configuration, and geographic distance. Publish benchmark numbers only after testing the same transaction and endpoints under controlled conditions.

## Technology

- **Language:** TypeScript
- **Runtime:** Node.js 20+
- **EVM client:** `viem`
- **Transport:** WebSocket for events; HTTP/WebSocket RPC for reads and submission
- **Validation:** `zod`
- **Logging:** structured JSON logs with `pino`
- **Testing:** Vitest with forked-chain integration tests

TypeScript provides reliable ABI inference, safer configuration, and a mature EVM ecosystem while remaining fast enough for Robinhood Chain event-driven execution. Performance-sensitive work stays off the trigger path; RPC and block inclusion latency normally dominate local TypeScript execution time.

## Suggested architecture

```text
src/
├── adapters/
│   ├── pons-v1.ts
│   └── pons-v2.ts
├── chain/
│   ├── contracts.ts
│   ├── providers.ts
│   └── watcher.ts
├── execution/
│   ├── broadcaster.ts
│   ├── nonce-manager.ts
│   ├── simulator.ts
│   └── transaction-builder.ts
├── risk/
│   ├── filters.ts
│   ├── sizing.ts
│   └── tax-monitor.ts
├── exits/
│   └── position-manager.ts
├── notifications/
├── config.ts
└── index.ts
```

## Requirements

- Node.js 20 or later
- A Robinhood Chain wallet funded with the correct gas and quote assets
- At least one reliable Robinhood Chain WebSocket RPC endpoint
- Verified Pons V1/V2 contract addresses and ABIs
- A dedicated trading wallet—never use your primary treasury wallet

## Installation

```bash
git clone https://github.com/YOUR_USERNAME/pons-sniper-bot.git
cd pons-sniper-bot
npm install
cp .env.example .env
npm run typecheck
npm test
```

Replace `YOUR_USERNAME` and all placeholder links before publishing.

## Configuration

Example `.env`:

```dotenv
# Network
ROBINHOOD_CHAIN_ID=4663
RPC_HTTP_URL=https://your-private-http-rpc
RPC_WS_URL=wss://your-private-websocket-rpc
BACKUP_RPC_HTTP_URL=

# Wallet
# Use a dedicated low-balance wallet. Never commit this value.
TRADER_PRIVATE_KEY=

# Mode: dry-run | paper | live
BOT_MODE=dry-run
PONS_VERSION=auto

# Entry controls
BUY_AMOUNT_ETH=0.01
MAX_SLIPPAGE_BPS=300
MAX_PRICE_IMPACT_BPS=500
MAX_GAS_GWEI=2
ENTRY_TIMEOUT_MS=10000

# V2 tax-aware entry
V2_MAX_EFFECTIVE_TAX_BPS=100
V2_TAX_POLL_INTERVAL_MS=100
V2_REQUIRE_FRESH_BLOCK=true

# Risk controls
MAX_OPEN_POSITIONS=3
MAX_DAILY_SPEND_ETH=0.10
REQUIRE_SELL_SIMULATION=true
ALLOW_UNKNOWN_CREATOR=false

# Optional exits
TAKE_PROFIT_PERCENT=50
STOP_LOSS_PERCENT=20
MAX_HOLD_SECONDS=300
```

The values above are examples, not recommended trading settings. Confirm the units and supported fee model in the current implementation.

## Running the bot

Start with dry-run mode:

```bash
npm run start -- --mode dry-run --version auto
```

Then use paper mode to record hypothetical entries without signing:

```bash
npm run start -- --mode paper --version v2
```

Live mode should require an explicit acknowledgement:

```bash
npm run start -- --mode live --i-understand-the-risk
```

Example startup output:

```text
Network          Robinhood Chain (4663)
Pons adapter     Auto-detect (V1 + V2)
Execution mode   DRY RUN
Wallet           0x12…89ab
Max buy          0.01 ETH
V2 max tax       1.00%
Sell simulation  Required
Status           Watching verified factory contracts
```

## Recommended safety flow

Before enabling live mode:

1. Verify every factory, router, curve, hook, and quote-token address from an authoritative source.
2. Test event decoding against historical V1 and V2 launches.
3. Run unit tests and forked-chain simulations.
4. Confirm the buy and sell paths with a very small amount.
5. Compare quoted output with the actual transaction receipt.
6. Force RPC disconnections, replacement transactions, reorgs, and nonce conflicts in testing.
7. Set wallet, per-trade, open-position, and daily-loss limits.
8. Keep automatic approvals limited to the exact contracts and amounts required.

## Risk controls

The execution engine should reject a trade when any enabled rule fails:

- unsupported or unverified Pons contract version;
- effective tax above the configured maximum;
- output below `minAmountOut`;
- price impact or slippage above its limit;
- gas price or estimated total cost above its ceiling;
- failed buy or sell simulation;
- unapproved creator or quote token;
- duplicate launch or already-open position;
- stale block/RPC response;
- entry timeout reached;
- daily spend, loss, or exposure limit reached.

Skipping a trade is an expected outcome, not an error.

## Security

- Never commit `.env`, private keys, seed phrases, database files, or raw logs containing secrets.
- Use a new trading wallet with only the amount required for the strategy.
- Prefer an encrypted keystore or external signer over a plaintext private key.
- Validate chain ID before signing and contract addresses before approving.
- Do not grant unlimited token approvals unless they are explicitly required and understood.
- Treat third-party RPC providers as infrastructure dependencies, not trusted trade authorities.
- Review and revoke approvals after testing.
- Add `.env`, `data/`, `logs/`, and keystore files to `.gitignore`.

## Benchmarking

Measure performance instead of advertising an unverifiable “fastest bot” claim. Recommended metrics:

- block timestamp to event received;
- event received to launch decoded;
- event received to simulation completed;
- V2 threshold reached to transaction broadcast;
- transaction broadcast to RPC acknowledgement;
- launch block relative to inclusion block;
- requested versus actual token output;
- effective tax, gas, and price impact per fill;
- skipped trades and rejection reasons.

Example benchmark table for your verified results:

| Metric | p50 | p95 | Sample size |
| --- | ---: | ---: | ---: |
| Event detection latency | TBD | TBD | TBD |
| V1 build + sign latency | TBD | TBD | TBD |
| V2 threshold-to-broadcast | TBD | TBD | TBD |
| RPC acknowledgement | TBD | TBD | TBD |

Do not fill this table with estimates. Record results from the production-like server, RPC endpoints, and version being advertised.

## Troubleshooting

### Launches are detected late

- Use a private WebSocket RPC with stable block propagation.
- Place the server geographically close to the selected RPC infrastructure.
- Confirm the watcher subscribes to the current factory address and event signature.
- Remove third-party REST APIs from the trigger path.

### V2 trades execute with excessive tax

- Require a successful output simulation immediately before signing or broadcasting.
- Lower `V2_MAX_EFFECTIVE_TAX_BPS`.
- Reject stale blocks and cached RPC responses.
- Check whether the tax is address-specific and query using the actual trading address.
- Enforce `minAmountOut` on-chain; do not rely only on a UI quote.

### Transactions fail or replace each other

- Run one serialized nonce queue per wallet.
- Reconcile pending and confirmed nonces after reconnecting.
- Do not operate the same wallet from multiple bot instances without coordination.

## Roadmap

- [x] V1/V2 adapter interface
- [x] Tax-aware V2 entry design
- [ ] Verified production contract registry
- [ ] Historical event replay tests
- [ ] Forked-chain buy and sell test suite
- [ ] Encrypted keystore support
- [ ] Web dashboard
- [ ] Telegram control panel
- [ ] Strategy analytics and CSV export
- [ ] Public reproducible latency report

Update this checklist to match the repository. Do not mark unfinished features as complete.

## FAQ

### Does it always buy in the first block?

No. V1 can target the earliest safe entry. V2 intentionally waits when the opening tax or expected output is unacceptable. Inclusion is never guaranteed.

### Is the fastest entry always the most profitable?

No. On V2, paying an extreme opening tax can outweigh any price advantage. The bot targets the earliest entry that satisfies your economic and safety limits.

### Does it guarantee profit or prevent rugs?

No. Simulations and filters reduce certain risks but cannot eliminate malicious contracts, creator selling, liquidity loss, RPC failures, reorgs, or market losses.

### Can I use multiple wallets?

Yes, if implemented with separate nonce queues and strict exposure limits. Multi-wallet trading does not remove market or contract risk.

### Does the bot store my private key?

The recommended architecture signs locally. Whether a particular distribution stores or transmits secrets must be confirmed through its source code and deployment model.

## Disclaimer

This software is experimental execution infrastructure and is provided for educational and research purposes. It does not provide financial advice or guarantee transaction ordering, token safety, execution, or profit. Newly launched tokens can lose all value. You are responsible for reviewing the code, contracts, fees, permissions, applicable laws, and every transaction signed by your wallet.

Pons Sniper Bot is an independent project and is not affiliated with or endorsed by Pons Family, Robinhood Markets, or any RPC provider unless explicitly stated by those organizations.

## License

Choose a license before publishing:

- Use **MIT** if the repository is fully open source.
- Use a **commercial/proprietary license** if buyers receive binaries or restricted source access.
- Do not add an open-source license if you intend to preserve exclusive commercial rights.

## Contact

- X: [@trustdev_eth](https://x.com/trustdev_eth)
- Telegram: `@trustdev_eth`

Never send funds or private keys to impersonators. Verify all contact accounts from this repository.
