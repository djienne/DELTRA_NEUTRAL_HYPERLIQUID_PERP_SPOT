# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an automated delta-neutral trading bot for Hyperliquid that earns funding rate arbitrage by maintaining market-neutral positions (SHORT PERP + LONG SPOT). The bot automatically selects the best opportunities, opens positions with parallel execution, and manages them based on funding rate changes.

## Common Commands

### Running the Bot

```bash
# Set leverage to 1x for all pairs (required first-time setup)
node tests/set-leverage.js

# Start the automated trading bot
node bot.js

# Test rebalancing logic (runs in 2 minutes)
node tests/test-rebalancing.js

# Run with custom hold time for testing
MIN_HOLD_TIME_MS=300000 node bot.js  # 5 minutes
```

The bot runs continuously with hourly check cycles and maintains persistent state in `bot-state.json`.

### Market Analysis Tools

```bash
# Check current funding rates (sorted highest first)
node tests/check-funding-rates.js

# Check funding rates with 7-day historical averages
node tests/check-funding-history.js

# Check your current PERP and SPOT positions
node tests/check-positions.js

# Analyze 24-hour trading volumes
node tests/check-24h-volumes.js

# Check bid-ask spreads
node tests/check-spreads.js

# Check PERP-SPOT price spreads
node tests/check-perp-spot-spreads.js

# Verify wallet balance distribution
node tests/check-spot-perp-balances.js
```

### Emergency Operations

```bash
# Close all positions immediately (parallel execution)
node emergency-close.js

# Analyze and hedge unbalanced positions
node tests/hedge-positions.js --analyze

# Execute hedges for unbalanced positions
node tests/hedge-positions.js --execute

# View current positions
node tests/get-hl-positions.js
```

## Architecture

### Core Components

**`bot.js`** - Main trading bot orchestrator
- Loads state from `bot-state.json` to recover position on restart
- Displays comprehensive statistics report at startup
- Runs hourly check cycles indefinitely
- Displays status updates every 2 minutes (position details, funding info, age, market summary)
- Manages position lifecycle (open → hold → close/switch)
- Implements exponential backoff for rate limit (429) errors
- Validates orderbook data before displaying tables (no NaN values)
- Uses timestamps throughout logs for better tracking
- Color-coded output for improved readability

**`hyperliquid.js`** - Exchange connector
- WebSocket connection for real-time orderbook streaming
- REST API fallback with automatic failover
- EIP-712 signature generation for authenticated actions
- Rate limiting (1800 WS msgs/min, 600 REST requests/min)
- Price/size rounding per Hyperliquid's rules (5 sig figs, szDecimals)
- Symbol mapping between PERP (BTC) and SPOT (UBTC) formats

### Utility Modules (`utils/`)

**Market Data Utilities:**
- `funding.js` - Fetch current and historical (7-day) funding rates
- `volume.js` - Fetch 24h volumes, convert coin units → USDC
- `spread.js` - Calculate bid-ask spreads for PERP and SPOT
- `arbitrage.js` - Calculate PERP-SPOT price spreads
- `positions.js` - Fetch PERP/SPOT positions, analyze delta-neutral pairs

**Trading Utilities:**
- `opportunity.js` - Filter and rank opportunities by multiple criteria
- `trade.js` - Open/close positions with parallel order execution (sets leverage before opening, validates capital against minOrderSizeUSD)
- `balance.js` - Check PERP/SPOT balance distribution, suggest transfers
- `leverage.js` - Set and manage leverage (1x isolated, set per-pair when opening position)
- `hedge.js` - Automatic position rebalancing and hedge creation
- `state.js` - Persistent state management for position recovery
- `statistics.js` - Generate comprehensive market statistics reports with proper table formatting

**Supporting Utilities:**
- `symbols.js` - PERP ↔ SPOT symbol conversion
- `rate-limiter.js` - Sliding window rate limiter

### Bot Decision Flow

```
Startup:
├─ Load state from bot-state.json
├─ Verify position on-chain (recover if state out of sync)
├─ Connect to Hyperliquid WebSocket
└─ Auto-rebalance imbalanced positions (hedge.js)
   ├─ Detect weak hedges (>5% mismatch)
   ├─ Detect unhedged positions
   ├─ Create hedge orders (ignores size limits)
   └─ Fallback to close if hedge fails

Hourly Check Cycle:
├─ If position exists:
│  ├─ Verify still on-chain
│  ├─ Check if >= min hold time (default: 2 weeks, configurable)
│  ├─ If can close:
│  │  ├─ Check if position in ranked opportunities
│  │  ├─ If NOT in ranked (filtered out):
│  │  │  ├─ Check raw market data for actual funding
│  │  │  ├─ If negative → close and switch to positive alt (or close only)
│  │  │  └─ If below threshold → switch to better opportunity
│  │  ├─ If in ranked:
│  │  │  ├─ Check if funding turned negative → close and switch (or close only)
│  │  │  └─ Check if 2x better opportunity exists → close and switch
│  │  └─ Otherwise: hold position (same symbol or not better enough)
│  └─ Otherwise: hold position (too young)
│
└─ If no position:
   ├─ Check balance distribution (warn if PERP/SPOT not 50/50 ±10%)
   ├─ Find best opportunities:
   │  ├─ Filter by bid-ask spread (≤ 0.15%)
   │  ├─ Filter by PERP-SPOT spread (≤ 0.5%)
   │  ├─ Filter by 24h volume (≥ $75M USDC)
   │  ├─ Filter by funding rate (≥ 5% APY, filters negative)
   │  └─ Rank by 7-day avg funding (highest first)
   │
   ├─ If no valid opportunities (all filtered):
   │  └─ Skip opening (e.g., all symbols have negative funding)
   │
   └─ Open position:
      ├─ Calculate size (95% of balance, max $20 except BTC $150)
      ├─ Round to proper lot sizes (handle different szDecimals)
      └─ Execute SHORT PERP + LONG SPOT in parallel
```

### State Management

**Persistent State (`bot-state.json`):**
```javascript
{
  version: "1.0",
  position: {
    symbol: "BTC",
    perpSymbol: "BTC",
    spotSymbol: "UBTC",
    perpSize: 0.001,
    spotSize: 0.001,
    perpEntryPrice: 107500,
    spotEntryPrice: 107520,
    positionValue: 107.5,
    fundingRate: 0.0000125,
    annualizedFunding: 0.1095,
    openTime: 1234567890000,
    lastCheckTime: 1234567890000
  },
  lastCheckTime: 1234567890000,
  lastOpportunityCheck: 1234567890000,
  history: [/* closed positions */]
}
```

The bot can be stopped and restarted at any time - it will recover its position from state and verify on-chain.

### Order Execution Critical Details

**Order Response Validation:**
A response with `status: "ok"` does NOT mean the order filled. Always check:
```javascript
if (result.response?.data?.statuses?.[0]?.filled) {
  // Order filled
} else if (result.response?.data?.statuses?.[0]?.error) {
  // Order rejected with error message
}
```

**Parallel Execution:**
PERP and SPOT orders execute simultaneously via `Promise.all()` for minimal execution time gap. If SPOT order fails, the bot automatically closes the PERP position with `reduceOnly: true`.

**Size Rounding:**
PERP and SPOT have different `szDecimals`. The bot calculates the same base size, rounds each separately, and warns if mismatch > 2% (normal for different lot sizes).

**Leverage:**
Bot sets 1x isolated leverage for each pair immediately before opening a position. This minimizes liquidation risk for delta-neutral positions and avoids unnecessary API calls for pairs that won't be traded.

## Configuration (`config.json`)

```javascript
{
  "trading": {
    "pairs": ["BTC", "ETH", "SOL", ...],  // PERP symbols
    "maxSlippagePercent": 5.0,             // 5% max for market orders
    "balanceUtilizationPercent": 95,       // Use 95% of available balance
    "minOrderSizeUSD": {                   // Minimum order sizes (prevents opening if insufficient capital)
      "BTC": 150,
      "*": 20                              // Default for all other pairs
    }
  },
  "bot": {
    "minHoldTimeDays": 14,                 // Minimum days to hold position before rebalancing
    "improvementFactor": 2                 // Require 2x better funding to switch positions
  },
  "thresholds": {
    "minVolumeUSDC": 75000000,             // $75M min 24h volume
    "maxSpreadPercent": 0.15,              // 0.15% max bid-ask spread
    "maxPerpSpotSpreadPercent": 0.5,       // 0.5% max PERP-SPOT spread
    "minFundingRatePercent": 5             // 5% minimum funding APY (filters negative and low funding)
  },
  "rateLimit": {
    "maxConcurrentRequests": 5,            // Reduced to prevent rate limiting
    "delayBetweenBatches": 500,            // 500ms between batches
    "delayBetweenRequests": 250,           // 250ms between individual requests
    "maxRequestsPerSecond": 10             // Conservative limit
  }
}
```

## Environment Setup

Required environment variables in `.env`:
```
HL_WALLET=0x...           # Your Hyperliquid wallet address
HL_PRIVATE_KEY=0x...      # Private key for signing transactions
```

## Key Hyperliquid API Concepts

**Volume Units:** API returns volumes in coin units (e.g., BTC, not USDC). Always convert using `convertVolumesToUSDC()`.

**Symbol Formats:**
- PERP: Direct symbol name (e.g., "BTC")
- SPOT: U-prefix for non-canonical (e.g., "UBTC"), direct for canonical (e.g., "PURR")
- Orderbook: SPOT uses "@{index}" format (e.g., "@142" for UBTC)

**Funding Rates:**
- Paid every hour (24x per day)
- Positive = longs pay shorts (SHORT PERP earns funding)
- Annualized = hourly × 24 × 365
- Use 7-day averages for strategy selection (current rates are volatile)

**Price/Size Rounding:**
- Price: Max 5 sig figs, max decimals = 8 - szDecimals (spot) or 6 - szDecimals (perp)
- Size: Rounded to szDecimals
- Both implemented in `hyperliquid.roundPrice()` and `hyperliquid.roundSize()`

## Bot Parameters

- **Check Interval:** 1 hour (configurable in `bot.js`)
- **Status Display Interval:** 2 minutes (shows position, funding info, age, market summary)
- **Min Hold Time:** 14 days (configurable via `config.bot.minHoldTimeDays` or env var `MIN_HOLD_TIME_MS`)
- **Improvement Factor:** 2x (configurable via `config.bot.improvementFactor`, requires Nx better funding to switch)
- **Min Funding Threshold:** 5% APY (configurable via `config.thresholds.minFundingRatePercent`, filters negative/low funding)
- **Leverage:** 1x isolated (set per-pair when opening position, not globally at startup)
- **Position Sizing:** 95% of available balance, validated against minOrderSizeUSD ($20 default, $150 for BTC)
- **Display Precision:** 4 decimal places for prices and values
- **Timestamps:** Full date/time format (day/month/year hour:minute:second)

## Negative Funding Protection

The bot implements comprehensive 4-layer protection against negative funding scenarios:

**Layer 1: Opportunity Filter** (`utils/opportunity.js:195`)
- Filters out symbols with funding < `minFundingRatePercent` (default 5% APY)
- Prevents opening new positions in negative or low-funding symbols
- Applied when searching for new opportunities

**Layer 2: Active Position Check** (`bot.js:334-350`)
- For positions that appear in ranked opportunities
- Detects if current funding turned negative
- Switches to positive alternative or closes without reopening
- Checks if same symbol is best (no switch even if funding improved)

**Layer 3: Filtered Position Check** (`bot.js:289-332`) ⭐
- For positions filtered out due to low/negative funding
- Checks raw market data to detect actual funding rate
- Handles positions that wouldn't appear in ranked opportunities
- Critical for detecting funding that drops below threshold

**Layer 4: Opening Prevention** (`bot.js:376-383`)
- Won't open position if `analysis.best` is null
- Occurs when all symbols are filtered (e.g., all negative funding)
- Logs clear message explaining why no position opened

**Decision Matrix:**

| Current Funding | Best Alternative | Action |
|----------------|------------------|---------|
| Negative | Positive (different symbol) | SWITCH (close + reopen) |
| Negative | Positive (same symbol) | HOLD (already in best) |
| Negative | None/Negative | CLOSE (no reopen) |
| Below threshold (0-5%) | Above threshold | SWITCH |
| Below threshold | Below threshold | HOLD |
| Above threshold | 2x better | SWITCH |
| Above threshold | < 2x better | HOLD |

**Test Coverage:**
Run `node tests/test-rebalancing.js` to verify all 11 scenarios pass, including:
- Negative funding with positive alternative (switch)
- Negative funding without alternatives (close only)
- All symbols negative (skip opening)
- Filtered position with negative funding (detected and handled)
- Low funding below threshold (switch if better available)

## Hedge Utility and Position Rebalancing

### Automatic Hedge Creation (`utils/hedge.js`)

The bot includes a comprehensive hedge utility that automatically detects and corrects imbalanced positions:

**Key Functions:**
- `analyzeHedgeNeeds(hyperliquid, options)` - Analyzes positions and returns hedge recommendations
- `createHedge(hyperliquid, hedgeNeed, config, options)` - Creates a single hedge order
- `autoHedgeAll(hyperliquid, config, options)` - Automatically hedges all unbalanced positions

**Detection Logic:**
1. **Weak Hedges**: Delta-neutral pairs with >5% size mismatch are strengthened
2. **Unhedged SPOT**: Creates matching PERP SHORT for unhedged LONG SPOT
3. **Unhedged PERP**: Creates matching SPOT position for unhedged PERP

**Important Characteristics:**
- **Ignores Size Limits**: Hedge orders bypass the `maxOrderSizeUSD` constraint to ensure proper delta-neutral ratios
- **Lot Size Handling**: Properly rounds PERP and SPOT sizes separately (different szDecimals), warns if mismatch >2%
- **Fallback to Close**: If hedge creation fails, automatically closes the original position to prevent unhedged exposure
- **Minimum Value**: Defaults to $1 minimum value for hedges (can be configured)

**Hedge Quality Levels:**
- **PERFECT**: <5% size mismatch between PERP and SPOT
- **GOOD**: 5-15% mismatch
- **PARTIAL**: 15-30% mismatch
- **WEAK**: >30% mismatch (automatically strengthened on startup)

**Integration with Bot:**
The bot automatically runs `autoHedgeAll()` at startup to rebalance any imbalanced positions from previous failed orders or partial fills.

### Emergency Close Script (`emergency-close.js`)

Root-level script for fast parallel closure of all positions:

**Key Features:**
- Fetches all PERP and SPOT positions in parallel
- Gets current prices via REST API (`getAllMids()`)
- Pre-filters positions below $9.9 minimum notional (displays skipped positions)
- Closes remaining positions simultaneously using `Promise.all()`
- Properly handles `reduceOnly` flag (PERP only, not SPOT)
- Shows detailed summary of closed/skipped/failed positions

**Usage:**
```bash
node emergency-close.js
```

**Important Implementation Details:**
- Pre-filtering at $9.9 prevents order placement errors (Hyperliquid $10 minimum)
- Uses `overrideMidPrice` to pass REST API prices (doesn't rely on WebSocket cache)
- Sets `reduceOnly: false` for SPOT orders (reduceOnly is invalid for SPOT on Hyperliquid)
- Checks notional value before order placement to avoid minimum notional errors
- Returns success for skipped positions (no error exit)
- Displays skipped positions with actual notional values for transparency

## Key Implementation Patterns

### Exponential Backoff for Rate Limits

The bot implements exponential backoff for handling 429 (rate limit) errors:

```javascript
async function retryWithExponentialBackoff(fn, options = {}) {
  const { maxRetries = 5, initialDelay = 1000, maxDelay = 30000, onRetry = null } = options;
  let delay = initialDelay;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      const is429 = error.message?.includes('429') ||
                    error.message?.includes('Too Many Requests') ||
                    error.message?.includes('rate limit');

      if (!is429 || attempt === maxRetries) throw error;

      const nextDelay = Math.min(delay * 2, maxDelay);
      if (onRetry) onRetry(attempt + 1, maxRetries, delay, error);

      await new Promise(resolve => setTimeout(resolve, delay));
      delay = nextDelay;
    }
  }
}
```

**Key Features:**
- Initial delay: 1000ms (configurable)
- Max delay: 30000ms (configurable)
- Doubles delay on each retry (exponential)
- Detects 429 errors in error message
- Supports retry callbacks for logging
- Applied to market data fetching and statistics logging

### Data Validation Before Display

Before displaying tables with market data, the bot validates orderbook data:

```javascript
// Subscribe to all orderbooks
for (const symbol of config.trading.pairs) {
  await hyperliquid.subscribeOrderbook(symbol);
  const spotSymbol = HyperliquidConnector.perpToSpot(symbol);
  await hyperliquid.subscribeOrderbook(spotSymbol, true);
}

// Wait for valid data (retry loop with validation)
const maxRetries = 5;
const retryDelay = 500;
let validDataCount = 0;

for (let retry = 0; retry < maxRetries; retry++) {
  await new Promise(resolve => setTimeout(resolve, retryDelay));
  validDataCount = 0;

  for (const symbol of config.trading.pairs) {
    const perpBidAsk = hyperliquid.getBidAsk(symbol);
    const spotSymbol = HyperliquidConnector.perpToSpot(symbol);
    const spotBidAsk = hyperliquid.getBidAsk(spotSymbol, true);

    if (perpBidAsk?.bid && perpBidAsk?.ask && perpBidAsk?.mid && perpBidAsk.mid > 0 &&
        spotBidAsk?.bid && spotBidAsk?.ask && spotBidAsk?.mid && spotBidAsk.mid > 0) {
      validDataCount++;
    }
  }

  // Accept when ≥75% of symbols have valid data
  if (validDataCount >= config.trading.pairs.length * 0.75) break;
}

// Validate spreads
let perpSpread = null;
if (perpBidAsk.ask && perpBidAsk.bid && perpBidAsk.mid && perpBidAsk.mid > 0) {
  perpSpread = ((perpBidAsk.ask - perpBidAsk.bid) / perpBidAsk.mid) * 100;
  if (!isFinite(perpSpread)) perpSpread = null;
}
```

**Key Features:**
- WebSocket subscription pattern (subscribe → wait → validate → use → unsubscribe)
- Retry loop up to 5 times with 500ms delays
- Validates ≥75% of symbols have valid data before proceeding
- Uses `isFinite()` to prevent NaN values
- Checks bid, ask, mid all present and mid > 0
- Throws error if no valid data after retries

### Table Alignment with Negative Numbers

Tables reserve space for minus signs even when displaying positive numbers:

```javascript
// Format with space reservation for sign
const fmtSigned = (n, digits = 1) => (n >= 0 ? ` ${n.toFixed(digits)}` : n.toFixed(digits));

// Example usage
const avgFundingNum = funding?.history?.avg?.annualized ? (funding.history.avg.annualized * 100) : null;
const avgFunding = avgFundingNum !== null ? fmtSigned(avgFundingNum) : 'N/A';

// Results in:
//  " 12.5" for positive (space + number)
//  "-3.4"  for negative (minus + number)
```

**Benefits:**
- Prevents column misalignment when mixing positive/negative numbers
- Maintains consistent visual alignment in tables
- Applied to funding rates, PnL, spreads in all tables

### Status Display Architecture

The bot displays comprehensive status every 2 minutes:

**Components:**
1. **Header:** Bot status with full date/time
2. **Position Details:** Symbol, PERP/SPOT sizes, entry prices (4 decimals), individual values, total value
3. **Funding Information:** Current funding rate (hourly and APY), expected earnings (hourly and daily)
4. **Position Age:** Days/hours/minutes held
5. **Rebalancing Status:** Time until rebalancing allowed or available now
6. **Market Summary Table:** All tracked symbols with funding (avg|current), 24h volume, bid-ask spreads (PERP|SPOT), PERP-SPOT spread, quality indicator

**Color Coding:**
- Cyan: Highlights (current position indicator, section headers)
- Green: Positive values (earning funding)
- Yellow: Warnings (moderate values)
- Red: Negative values or errors
- Dim: Secondary information (parenthetical values, borders)
- Bright: Emphasized values (totals)

### Position Sizing with Capital Validation

Position sizing validates available capital against minimum requirements:

```javascript
// Get minimum notional from config
const minNotional = config.trading?.minOrderSizeUSD?.[symbol] || 20;

// Calculate available capital (95% utilization)
const availablePerpNotional = perpBalance * (utilization / 100);
const availableSpotNotional = spotBalance * (utilization / 100);
const availableNotional = Math.min(availablePerpNotional, availableSpotNotional);

// Validate before opening position
if (availableNotional < minNotional) {
  const errorMsg = `Insufficient capital for ${symbol}: $${availableNotional.toFixed(2)} available < $${minNotional.toFixed(2)} minimum required`;
  console.error(`[Trade] ❌ ${errorMsg}`);
  return {
    success: false,
    error: errorMsg,
    symbol: symbol,
    availableCapital: availableNotional,
    minimumRequired: minNotional
  };
}

// Calculate position size from available capital (no division by 2)
const size = availableNotional / perpMid;
```

**Key Points:**
- Uses full balance with 95% utilization (NOT divided by 2)
- Takes minimum of PERP and SPOT available capital
- Validates against minOrderSizeUSD before opening
- Returns detailed error with actual values for debugging

## Important Development Guidelines

### Leverage Management
**IMPORTANT:** Leverage is set to 1x isolated on a **per-pair basis** immediately before opening each position (in `trade.js`), NOT globally for all pairs at startup. This optimization:
- Reduces unnecessary API calls (only sets leverage for pairs actually being traded)
- Minimizes startup time
- Avoids rate limiting issues
- Sets leverage exactly when needed

### No Dry-Run Mode
**IMPORTANT:** Never implement dry-run, simulation, or test modes for trading operations. All trading functions should execute real orders. Analysis and reporting functions (like `analyzeHedgeNeeds`) that don't execute trades are fine, but any function that creates orders should do so for real. This is a production trading bot that requires immediate execution without safety guards that could delay critical operations.

### Order Response Status
**CRITICAL:** When creating orders, `status: "ok"` only means the request was valid. Always check `response.data.statuses[0]` for actual fill status (`filled`, `error`, or `resting`).

### Spot Trading Constraints
- Never use order size > $20 USDC except BTC ($150)
- Minimum order notional: $10
- Max slippage: 5% (from config)

### Volume Data
Volume from API is in coin units, NOT USDC. Always convert with `convertVolumesToUSDC()` for meaningful comparisons.

### Delta-Neutral Position Monitoring
Use `tests/check-positions.js` to analyze existing positions and verify hedge quality (PERFECT < 5% mismatch, GOOD < 15%, PARTIAL < 30%, WEAK > 30%).

### Funding Rate Analysis
- Current rates are volatile (change hourly)
- Use 7-day averages for strategy selection
- Best opportunities = high avg funding (>8%) + low volatility (range <30%)
- Check with `tests/check-funding-history.js`

### Common Pitfalls and Critical Fixes

**Position Sizing:**
- MUST use full balance with utilization percentage, NOT divide by 2
- CORRECT: `perpBalance * (utilization / 100)`
- INCORRECT: `(perpBalance / 2) * (utilization / 100)` ← This halves position size
- Take minimum of PERP and SPOT available to ensure both sides can fill
- Location: `utils/trade.js` openDeltaNeutralPosition function
- Bug symptom: $340 balance produces $170 position instead of $323

**Slippage Calculation:**
- Config stores slippage as percentage (5.0 = 5%)
- MUST convert to decimal: `slippage > 1 ? slippage / 100 : slippage`
- Without conversion: SELL orders get negative prices (e.g., $40 * (1 - 5.0) = -$160)
- Location: `hyperliquid.js` createMarketOrder function

**Symbol Matching:**
- Hyperliquid API returns `asset.coin` as STRING (e.g., "HYPE"), not numeric index
- Always check if `typeof assetIndex === 'string'` before treating as array index
- Use `meta.universe.find(a => a.name === assetIndex)` for string symbols
- Location: `positions.js` getPerpPositions function

**Property Name Consistency:**
- PERP/SPOT spread maps use `perpSymbol` as key, not `symbol`
- Volume objects use `perpVolUSDC` and `spotVolUSDC`, not `perpVolumeUSDC`
- Delta-neutral analysis returns `balance.total`, not direct `.total`
- Location: `opportunity.js`, `hedge.js`

**reduceOnly Flag:**
- Works for PERP markets only
- INVALID for SPOT markets (causes order rejection)
- Always use: `reduceOnly: isSpot ? false : true`
- Location: All order placement code

**Minimum Notional:**
- Hyperliquid requires $10 minimum order size
- Emergency close pre-filters at $9.9 to avoid errors
- Check notional BEFORE placing order to avoid errors
- Skip/close positions below minimum gracefully
- Display skipped positions with actual notional values for transparency
- Location: `emergency-close.js`, `utils/trade.js` (minOrderSizeUSD validation), hedge creation functions

**Data Validation and NaN Values:**
- ALWAYS validate orderbook data before calculating spreads
- Check bid, ask, mid all present AND mid > 0 (prevent division by zero)
- Use `isFinite()` to prevent NaN values: `if (!isFinite(spread)) spread = null;`
- Implement retry loops to wait for valid WebSocket data
- Require ≥75% of symbols to have valid data before displaying tables
- Location: `bot.js` displayStatus function, spread calculation
- Bug symptom: "NaN" or "---" displayed in tables

**Table Formatting with Negative Numbers:**
- Reserve space for minus sign even when displaying positive numbers
- Use: `n >= 0 ? \` \${n.toFixed(1)}\` : \`\${n.toFixed(1)}\``
- Without space reservation: Tables misalign when mixing positive/negative values
- Apply to all numeric columns: funding rates, PnL, spreads
- Location: `bot.js`, `utils/statistics.js`
- Bug symptom: Table headers/footers misaligned with data rows

**Negative Funding Protection:**
- MUST check raw market data if position not in ranked opportunities (filtered out)
- Positions can be filtered due to funding < minFundingRatePercent (default 5%)
- MUST check `analysis.marketData.fundingRates` for actual funding rate
- MUST only reopen if `newOpportunity.avgFundingPercent > 0` (positive funding)
- MUST skip opening if `analysis.best` is null (all symbols filtered)
- Location: `bot.js` lines 289-332 (filtered position check), 295-310 (active position check), 374-383 (opening prevention)
- Bug symptom: Position with negative funding not detected, bot doesn't close or opens position when all symbols negative
- Test coverage: Run `node tests/test-rebalancing.js` to verify all 11 scenarios
