# ProtoPerps

**Perpetual futures on pre IPO company valuations on Solana.**

Trade long/short exposure to SpaceX, OpenAI, Anthropic, Anduril, Polymarket, Neuralink, and Kalshi without touching real shares, SPVs, or any off-chain settlement. Prices track live [Prestocks](https://prestocks.com) DEX token valuations, updated on-chain every 30 seconds by a keeper bot.

**Live:** [protoperps.vercel.app](https://protoperps.vercel.app)

---

## What is this?

### Perpetual futures

A perpetual future (perp) is a leveraged derivative that tracks an underlying asset's price with no expiry date. Unlike options or futures, you hold it indefinitely — your PnL accumulates in real time as the price moves against your entry.

Price alignment with the underlying is maintained via a **funding rate**: when the perp trades above the oracle price, longs pay shorts (pressure to sell); when it trades below, shorts pay longs (pressure to buy). This creates a continuous rebalancing force.

### Synthetic perpetuals

ProtoPerps is a **synthetic** perp protocol, meaning:

- Positions are **not** matched against a direct counterparty in a traditional sense — the protocol itself holds the math
- The underlying asset (SpaceX, OpenAI stock) **never moves on-chain** — only USDC does
- Price truth comes from an **on-chain oracle**, not the perp market itself
- Any asset with a price feed can be listed — including pre-IPO private companies that have no public market

This is distinct from a CEX perp (where Binance is the counterparty and the order book holds real positions against each other) or a synthetic debt-pool protocol like Synthetix (where a shared liquidity pool absorbs all risk). ProtoPerps uses a **direct counterparty orderbook model** — every long is matched against a short — but the settlement is fully on-chain, non-custodial, and the underlying asset is synthetic.

### Why private companies?

Private company shares can't be traded publicly. Prestocks tokenizes implied valuations of companies like SpaceX and OpenAI as Solana SPL tokens, creating a discoverable market price. ProtoPerps uses those token prices as the oracle feed, enabling leveraged speculation on private valuations — something impossible on any traditional exchange.

---

## Markets

| Market | Symbol | Prestocks Token Mint | Fallback Price |
|--------|--------|----------------------|----------------|
| SpaceX | SPACEX-PERP | `PreANxuXjsy...` | $732 |
| OpenAI | OPENAI-PERP | `PreweJYECqt...` | $1,761 |
| Anthropic | ANTHRP-PERP | `Pren1FvFX6J...` | $1,300 |
| Anduril | ANDURL-PERP | `PresTj4Yc2b...` | $166 |
| Polymarket | POLMKT-PERP | `Pre8AREmFPt...` | $180 |
| Neuralink | NRLNK-PERP | `PrekqLJvJ3q...` | $358 |
| Kalshi | KALSHI-PERP | `PreLWGkkeqG...` | $554 |

Prices are fetched every 30 seconds from Dexscreener mainnet pools (highest-liquidity Solana pair per mint), clamped to ±9% of the on-chain `previous_price`, and pushed to devnet via the keeper bot.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Next.js Frontend                           │
│  Markets page · Trade page · Portfolio page                     │
│  SWR polling · Solana Wallet Adapter · Anchor client            │
│  /api/dex/[mint] → Dexscreener proxy (24h%, volume, liquidity)  │
└────────────────────────┬────────────────────────────────────────┘
                         │ reads / writes
┌────────────────────────▼────────────────────────────────────────┐
│               protoperps program  (Anchor / Rust)               │
│                                                                 │
│  Market accounts  — zero-copy orderbook (64 bids × 64 asks)     │
│  Position accounts — per (trader × market), isolated margin     │
│  MarginAccount PDAs — USDC collateral per wallet                │
│                                                                 │
│  place_order   cancel_order   liquidate                         │
│  deposit_collateral   withdraw_collateral                       │
│  update_funding   settle_funding                                │
└───────────────┬───────────────────────┬─────────────────────────┘
                │ reads oracle          │ invoked by keeper
┌───────────────▼──────────┐   ┌────────▼──────────────────────────┐
│  oracle program           │   │  keeper bot  (TypeScript)         │
│  (Anchor / Rust)          │   │                                   │
│                           │   │  oracle.ts    Dexscreener mainnet │
│  OraclePrice PDAs         │   │               → on-chain push     │
│  update_price             │   │               every 30 s          │
│  admin_pause              │   │                                   │
│  EMA TWAP                 │   │  funding.ts   hourly funding      │
│  ±10% deviation guard     │   │               rate update         │
│  5 min → reduce-only      │   │                                   │
│  15 min → paused          │   │  liquidator.ts scan positions,    │
│                           │   │               execute liquidations │
└───────────────────────────┘   └───────────────────────────────────┘
```

---

## Program design

### Crankless matching

Every `place_order` call resolves the full trade atomically in the same transaction. There is no external crank, no async settlement queue, no off-chain order routing. The matching engine fills against the on-chain orderbook in price-time priority and settles all positions within the same instruction.

Maker Position PDAs are passed as `remaining_accounts` (up to 5 per transaction), the same pattern used by Phoenix v1.

### Zero-copy orderbook

The `Market` account stores `[Order; 64]` bids and `[Order; 64]` asks — 9,424 bytes of orderbook data — using Anchor's `zero_copy` / `bytemuck::Pod` approach. The account is memory-mapped directly; no heap or stack copy is created on deserialisation. This avoids SBF's 4 KB stack frame limit.

```
Order layout (#[repr(C)], 72 bytes, no implicit padding):
  price           u64     (8)
  size            u64     (8)
  sequence_number u64     (8)   — tiebreaker: lower = earlier = higher priority
  timestamp       i64     (8)
  trader          [u8;32] (32)
  active          u8      (1)   — 0 = slot free, 1 = live
  side            u8      (1)   — 0 = Long, 1 = Short
  order_type      u8      (1)   — 0 = Limit, 1 = Market, 2 = PostOnly
  _pad            [u8;5]  (5)   — explicit, total = 72 = 9 × 8 ✓
```

Bids are kept sorted descending by price; asks ascending. Self-trade prevention is enforced in the matching loop.

### Isolated margin

Each position carries its own USDC collateral — no cross-margin in v1. `usdc_locked` in the `MarginAccount` tracks total collateral committed across all open positions.

```
free_collateral = usdc_deposited − usdc_locked
```

### Position math

| Event | Behaviour |
|-------|-----------|
| New position | Entry price = fill price |
| Add to position | Entry price recalculated as VWAP |
| Partial close | PnL = `(close − entry) × size / LOT_PRECISION` (sign-flipped for shorts) |
| Full close | PnL settled, position zeroed |
| Flip | Close all → realise PnL → open opposite |

### Risk parameters

| Parameter | Value |
|-----------|-------|
| Max leverage | 50× |
| Initial margin ratio | 2% |
| Maintenance margin ratio | 1% |
| Liquidation reward | 5% of remaining collateral |
| Collateral | USDC only |
| Orderbook depth | 64 bids / 64 asks per market |
| Max fills per tx | 5 (remaining_accounts) |

---

## Oracle design

### Price feed lifecycle

```
Dexscreener mainnet REST API
  (highest-liquidity Solana pair per Prestocks mint)
        │ every 30 s
        ▼
  keeper / oracle.ts
  · reads on-chain previous_price (byte offset 104 in OraclePrice)
  · clamps new price to ±9% of previous_price
  · calls oracle::update_price { price, confidence, source }
        │
        ▼
  OraclePrice PDA  (oracle program, devnet)
  · rejects if |new − previous_price| / previous_price > 10%
  · updates EMA TWAP
  · records slot + timestamp for staleness tracking
        │
        ▼
  protoperps::place_order
  · reads OraclePrice via cross-program zero-copy load
  · checks effective status (Active / ReduceOnly / Paused)
  · rejects new orders if not Active
```

### Staleness escalation

| Age since last update | Oracle status |
|-----------------------|---------------|
| < 5 minutes | Active — normal trading |
| 5 – 15 minutes | Reduce-only — close/reduce positions only |
| > 15 minutes | Paused — all orders rejected |

### TWAP (EMA)

```
twap_new = twap_old + (new_price − twap_old) / min(sample_count, 100)
```

Alpha floors at 1% (100 samples). A single outlier price can move the TWAP by at most 1%.

### Deviation guard

```rust
// Reject if: |new − previous| / previous > 10%
diff.saturating_mul(10_000) > previous.saturating_mul(1_000)
```

When real prices diverge significantly from the last on-chain price (e.g. after the keeper restarts), the ±9% clamp in the keeper allows gradual convergence over 3–5 ticks (~2.5 minutes) without triggering the on-chain rejection.

---

## Funding rate

ProtoPerps uses funding to keep the perp mark price aligned with the oracle (Prestocks token price):

```
funding_rate = (mark_price − oracle_price) / oracle_price × (1/24)
```

- Computed and pushed on-chain by the keeper every hour
- **Longs pay shorts** when mark > oracle (perp is at a premium)
- **Shorts pay longs** when mark < oracle (perp is at a discount)
- `Market::cumulative_funding_rate` (i64) accumulates the running sum
- Each position stores `last_funding_rate`; unsettled funding is applied lazily on any position interaction

This is the same mechanism used by BitMEX-style CEX perps, translated entirely on-chain.

---

## How a trade works end-to-end

```
1. Deposit USDC
   trader → deposit_collateral → USDC vault
   MarginAccount.usdc_deposited += amount

2. Place order  (e.g. Long SpaceX 0.1 @ market)
   · oracle checked: must be Active
   · matching engine walks asks, fills up to 5 makers
   · maker Position PDAs in remaining_accounts updated atomically
   · required initial margin (2%) locked from free_collateral
   · if unfilled: resting limit order inserted into orderbook

3. Position lives
   · mark price tracked via oracle keeper pushes every 30 s
   · funding accrues hourly in cumulative_funding_rate
   · unrealized PnL = (mark − entry) × size / LOT_PRECISION

4. Close position
   · trader places opposing market order
   · fills against resting bids, PnL released, collateral unlocked

5. Withdraw
   · withdraw_collateral for any free_collateral
```

---

## Liquidation

When `equity / notional < 1%` (maintenance margin breached):

```
equity   = collateral + unrealized_pnl
notional = mark_price × size / LOT_PRECISION
```

Any wallet can call `liquidate`. The liquidator:
- Closes the position at mark price
- Receives 5% of the remaining collateral as a reward
- Remaining collateral is returned to the trader

---

## Precision constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `PRICE_PRECISION` | 1,000,000 | $1.00 = 1,000,000 |
| `LOT_PRECISION` | 1,000,000 | 1 unit = 1,000,000 |
| `BPS_PRECISION` | 10,000 | 100% = 10,000 bps |
| `FUNDING_PRECISION` | 1,000,000,000 | 1.0 funding = 1e9 |

All on-chain arithmetic uses `u64` / `i64` fixed-point with explicit checked operations. No floating point anywhere in the programs.

---

## Account layout (key fields)

### OraclePrice (136 bytes on-chain)

```
offset   field                  type
     0   bump / source / status u8 × 3
     3   _pad0                  [u8;5]
     8   authority              Pubkey (32)
    40   market                 Pubkey (32)
    72   price                  u64        ← current price
    80   confidence             u64
    88   twap                   u64        ← EMA
    96   previous_price         u64        ← deviation guard uses this
   104   twap_samples           u64
   112   last_update_slot       u64
   120   last_update_timestamp  i64        ← staleness computed from this
```

### Market (9,432 bytes on-chain)

Zero-copy. Holds the full sorted orderbook, market parameters, cumulative funding rate, open interest, and volume.

### Position

```
market          Pubkey
trader          Pubkey
side            Long | Short
size            u64   (LOT_PRECISION)
entry_price     u64   (PRICE_PRECISION, VWAP)
collateral      u64   (USDC)
last_funding_rate i64
realized_pnl    i64
```

---

## PDA seeds

| Account | Seeds | Program |
|---------|-------|---------|
| Market | `["market", base_symbol_bytes_16]` | protoperps |
| Position | `["position", market_pubkey, trader_pubkey]` | protoperps |
| MarginAccount | `["margin", owner_pubkey]` | protoperps |
| Vault authority | `["vault"]` | protoperps |
| OraclePrice | `["oracle", market_pubkey]` | oracle |

---

## Devnet program IDs

| Program | Address |
|---------|---------|
| protoperps | `J65U84LyKvCtv76ynd4MBCfjQqTXLjHvFbpieVqRUjbW` |
| oracle | `Bk1ao9hgiYxubch1XtrtaWTsYFscMqbH5QnahB6WLMZV` |

---

## Repo structure

```
protoperps/
├── programs/
│   ├── protoperps/
│   │   └── src/
│   │       ├── instructions/
│   │       │   ├── place_order.rs        # Matching engine + position math
│   │       │   ├── cancel_order.rs
│   │       │   ├── deposit_collateral.rs
│   │       │   ├── withdraw_collateral.rs
│   │       │   ├── liquidate.rs
│   │       │   ├── update_funding.rs
│   │       │   └── settle_funding.rs
│   │       ├── state/
│   │       │   ├── market.rs             # Zero-copy Market + Order structs
│   │       │   ├── position.rs
│   │       │   └── margin.rs
│   │       ├── oracle_client.rs          # Cross-program oracle reader
│   │       └── errors.rs
│   └── oracle/
│       └── src/
│           ├── instructions/
│           │   ├── initialize_feed.rs
│           │   ├── update_price.rs       # EMA TWAP + deviation guard
│           │   └── admin_pause.rs
│           └── state/oracle_price.rs    # Zero-copy OraclePrice (with unit tests)
├── keeper/
│   └── src/
│       ├── oracle.ts     # Dexscreener → on-chain price push, ±9% clamp
│       ├── funding.ts    # Hourly funding rate trigger
│       ├── liquidator.ts # Scan + execute liquidations
│       ├── mm.ts         # Market-maker bot (devnet liquidity seeding)
│       └── config.ts     # Markets, mint addresses, PDA helpers
├── app/
│   └── src/
│       ├── app/
│       │   ├── page.tsx                    # Markets overview
│       │   ├── trade/[symbol]/page.tsx     # Trade page
│       │   ├── portfolio/page.tsx
│       │   └── api/
│       │       ├── dex/[mint]/route.ts     # Dexscreener proxy (30s edge cache)
│       │       └── faucet/route.ts         # Devnet USDC faucet
│       ├── components/
│       │   ├── markets/MarketsTable.tsx
│       │   └── trade/
│       │       ├── OrderBook.tsx
│       │       ├── OrderEntry.tsx
│       │       ├── PositionsTable.tsx      # Partial close + remainingAccounts
│       │       └── PriceChart.tsx
│       ├── hooks/
│       │   ├── useOracle.ts      # On-chain oracle (5 s SWR)
│       │   ├── useMarket.ts      # On-chain market
│       │   ├── usePosition.ts
│       │   ├── useDexStats.ts    # 24h %, volume, liquidity (30 s SWR)
│       │   └── usePrograms.ts
│       └── lib/
│           ├── constants.ts      # Program IDs, PDAs, market configs + mints
│           └── math.ts           # formatPrice, formatCompact, PnL helpers
├── tests/                        # Integration tests (bankrun)
└── Anchor.toml
```

---

## Tech stack

| Layer | Technology |
|-------|------------|
| On-chain programs | Anchor 0.32, Rust, Solana SBF |
| Zero-copy accounts | `bytemuck::Pod`, `AccountLoader` |
| Keeper bot | TypeScript, `@coral-xyz/anchor`, `@solana/web3.js` |
| Price source | Dexscreener REST API (mainnet Solana pairs) |
| Frontend | Next.js 16, Tailwind CSS v4, SWR, Framer Motion |
| Wallet | `@solana/wallet-adapter-react` (Phantom, Backpack, Solflare) |
| Deployment | Vercel (frontend), Solana devnet (programs) |
| Tests | Anchor test framework + bankrun |

---

## Running locally

### Prerequisites

```bash
rustup update stable
sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"   # Solana CLI
cargo install --git https://github.com/coral-xyz/anchor avm && avm install latest
node --version   # 18+
```

### 1. Build and deploy programs

```bash
anchor build
anchor deploy --provider.cluster devnet
```

After deploy, update program IDs in `Anchor.toml`, `programs/*/src/lib.rs`, and `app/src/lib/constants.ts`.

### 2. Start the keeper

```bash
cp keeper/.env.example keeper/.env
# Set KEYPAIR_PATH, RPC_URL, PRICE_PUSH_INTERVAL_MS (default 30000)

cd keeper && npm install && npm run start
```

The keeper runs three services concurrently:
- **oracle** — fetches Dexscreener prices every 30 s, clamps to ±9%, pushes via `update_price`
- **funding** — calls `update_funding` once per hour per market
- **liquidator** — polls positions, calls `liquidate` when maintenance margin is breached

### 3. Start the frontend

```bash
cd app && npm install && npm run dev
# → http://localhost:3000
```

Optional `app/.env.local`:
```
NEXT_PUBLIC_RPC_URL=https://api.devnet.solana.com
NEXT_PUBLIC_USDC_MINT=EKdgVqQVivDRiXeQfK2k2Yx1W2BZZdYJ8D1KEaouriEM
```

### 4. Run integration tests

```bash
anchor test   # uses bankrun for fast local execution, no validator needed
```

---

## Frontend data sources

| Data | Source | Refresh |
|------|--------|---------|
| Mark price | On-chain OraclePrice PDA | 5 s (SWR) |
| Orderbook | On-chain Market PDA | 5 s (SWR) |
| Position | On-chain Position PDA | 5 s (SWR) |
| 24h %, Volume, Liquidity | `/api/dex/[mint]` → Dexscreener | 30 s (SWR + edge cache) |
| Funding rate | On-chain Market PDA | 5 s (SWR) |

---

## What ProtoPerps is not

- **Not a prediction market** — no outcome resolution, no expiry date
- **Not a spot exchange** — no real token or share transfers; positions are purely synthetic
- **Not a debt-pool synthetic** (like Synthetix) — every long is matched against a short in a direct counterparty orderbook, not absorbed by a shared liquidity pool
- **Not cross-margined** — each position is isolated; one bad trade cannot cascade into others
- **Not production-ready** — devnet only, not audited

---

## Acknowledgements

Architecture inspired by:
- **[Phoenix v1](https://github.com/Ellipsis-Labs/phoenix-v1)** — crankless on-chain orderbook, zero-copy account design, remaining_accounts fill pattern
- **[Drift Protocol v2](https://github.com/drift-labs/protocol-v2)** — funding rate mechanics, oracle staleness escalation, margin system design
- **[Prestocks](https://prestocks.com)** — pre-IPO token infrastructure providing the underlying price signal
