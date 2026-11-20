---
name: dex-api-integration-dragonswap
description: Use when integrating with the Dragonswap DEX API on Sei Network - getting swap quotes (/api/v4/quote plus the on-chain SplitRouter), reading pools, tokens, DEX stats, farms, staking, incentives, airdrops, or a wallet's positions, or debugging DEX API responses
---

# Dragonswap DEX API - Integration Reference

## Quick Reference

| Property | Value |
|----------|-------|
| Base URL | `https://sei-api.dragonswap.app` |
| API Prefix | `/api/v1/` (data) · `/api/v4/` (routing) — always with `/api`; `/v1/...` returns 404 |
| Authentication | None needed for the endpoints documented here |
| Chain | Sei Network (pacific-1) |
| Chain ID | `1329` |
| SplitRouter | `0xCaCbF14f07AF3Be93C6fc8bF0a43f9279F792199` |
| Staking contract | `0xb71886c52d754cf2b1d3d866c7cfe9eec01418a7` |
| Response Format | Data: `{ "status": "ok", ... }` (`/api/v1/vol` has no `status`). Quote: `{ "status": "success", ... }`. Errors: see below |
| CORS | Allowlisted origins only — call the API from a server, not from a browser on your domain |
| Caching | Cloudflare-cached; responses can be up to ~2 min old |

## Overview

The Dragonswap DEX API provides read-only access to pools, tokens, farms, staking, incentives, airdrops, user data, and swap quoting on the Sei Network. The quoting flow is two-step: fetch an optimal route via the API, then call the on-chain SplitRouter for exact amounts and price impact (required for `exactOut`).

## Conventions

- **Pool naming:** API `V2_POOL` / `v2` = DEX V1 (constant-product); `V3_POOL` / `v3` = DEX V2 (concentrated liquidity). Quote/SplitRouter `poolType`: `0` = V1, `1` = V2.
- **Addresses:** path addresses are case-insensitive. Quote token addresses must be EIP-55 checksummed or all-lowercase.
- **Timestamps:** Unix seconds, as numbers — except staking, where they are strings: emissions/rewards `created_at`, staking activity `timestamp`, user stakings payout `blockTimestamp`.
- **Amounts:** token amounts are decimal-adjusted numbers (reserves, swap amounts, farm amounts, incentive `reward`, staking emissions `amount`, `drg_fee`); staking bonus `amount` and airdrop `amounts` are decimal strings. Wei amounts are strings (`*_wei`, staking user-stats values, user stakings `amount`, quote `dex.inputAmount`/`dex.outputAmount`).
- **USD values:** numbers, except decimal strings for incentive `liquidity_usd` and `apr` and staking bonus `amount_usd_value`. `hourly_*` and `*_change_24h` fields can be `null`.
- **24h fields:** `daily_volume`, `daily_fees` = rolling last 24h. `daily_transactions` = cumulative count, not 24h.
- **Ended programs stay listed:** farms (`end_timestamp`), incentives (`end_time`), airdrops (`ends_at`) — compare with now to find active ones.
- **Snapshot series** (oldest first, last entry = latest): pool and token stats `daily` 24h, `weekly` 7d, `monthly` 1 month, `quarterly` 3 months, `all_time` last 12 months (not full history). Staking stats: `daily`, `weekly`, `monthly`, `all_time`.

## Errors

| Case | HTTP | Body |
|------|------|------|
| Unknown route (gateway) | 404 | `404 page not found` (text/plain, not JSON) |
| Wrong HTTP method (gateway) | 405 | `405 method not allowed` (text/plain) |
| Token not found | 404 | `{"status":"fail","error":{"code":404,"type":"token_not_found","message":"Token not found"}}` |
| Pool / farm not found | 404 | `{"status":"error","error":{"code":404,"type":"pool_not_found","message":"Pool does not exist."}}` (farms: `farm_not_found`, `Farm does not exist.`) |
| Staking `{address}` not the staking contract (activity, user stats) | 404 | `{"status":"fail","error":{"code":404,"type":"staker_address_not_found","message":"Staker address not found"}}` |
| Airdrop not found | 404 | `{"status":"fail"}` |
| Quote validation | 400 | `{"status":"error","message":[{ "code", "path", "message", ... }]}` |
| Quote: no route, `amount` `0` or non-integer | 500 | `{"status":"error","message":"..."}` |

Many lookups return `200` with empty data instead of 404 (noted per endpoint). Invalid address on `/user/{address}/portfolio` and `0` for a pool-stats query param return HTTP 500.

---

## Quoting Flow (Two-Step)

### Step 1: Route Discovery

```
GET /api/v4/quote?amount={wei}&tokenInAddress={addr}&tokenOutAddress={addr}&type=exactIn
```

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | string | yes | Base-10 integer string in the token's smallest unit. `exactIn`: input amount; `exactOut`: desired output amount. `0` or non-integer → 500 |
| `tokenInAddress` | string | yes | Checksummed or all-lowercase; bad checksum → 400 `Not an address` |
| `tokenOutAddress` | string | yes | Same rules |
| `type` | string | yes | `exactIn` or `exactOut` (case-sensitive; any other value is quoted as `exactIn`) |

Response: `{ status: "success", type: "dex", dex: { inputAmount, outputAmount, impactPips, route, linearSubroutes }, rfq: null, rfqConfig: null, unwrap }`

- `dex.inputAmount`, `dex.outputAmount`: wei strings. `dex.impactPips`: string, millionths (`1000000` = 100%, `10000` = 1%), excluding LP fees.
- `dex.route[]` (maps 1:1 to `SwapParams`): `{ pool (EIP-55), poolType (0=V1, 1=V2), inputToken, outputToken, feePips (3000 = 0.3%), inputPercentage, outputPercentage }`. `inputPercentage` = share (0–100) of the remaining, not-yet-routed balance of `inputToken`, in route order (a 95/5 split shows `95` then `100`). Pass `outputPercentage` through unchanged.
- `dex.linearSubroutes[]`: `{ percentage (share of total swap), swaps[] (hops without the percentage fields) }`.
- `unwrap`: present on `exactIn`, omitted on `exactOut`.
- Error: `{ status: "error", message }` — array (400 validation) or string (500).
- `exactIn`: `dex.outputAmount`/`dex.impactPips` match on-chain `quoteExactInput` at request time, so Step 2 is optional for display. Re-quote on-chain before executing.
- **`exactOut`: do not use `dex.inputAmount`/`dex.outputAmount`** (output can be below `amount`, input below what is required). Use `dex.route` with on-chain `quoteExactOutput`.
- Routes change as liquidity shifts — fetch a fresh route before each quote.

### Step 2: On-chain Quote

Call `DragonswapSplitRouter` at `0xCaCbF14f07AF3Be93C6fc8bF0a43f9279F792199` via `simulateContract` (read-only `eth_call`; the functions are `nonpayable`, not `view`).

- `quoteExactInput(ExactInputParams)` / `quoteExactOutput(ExactOutputParams)` → `(uint256 inputAmount, uint256 outputAmount, uint256 impactPips)`; `impactPips` in millionths (`7744` ≈ 0.77%), excluding LP fees.
- **ExactInputParams:** `{ inputAmount: uint256, minOutputAmount: uint256, deadline: uint256, unwrap: bool, route: SwapParams[] }`
- **ExactOutputParams:** `{ outputAmount: uint256, maxInputAmount: uint256, deadline: uint256, unwrap: bool, route: SwapParams[] }`
- **SwapParams:** `{ poolType: uint8, pool: address, inputToken: address, outputToken: address, feePips: uint24, inputPercentage: uint8, outputPercentage: uint8 }`
- `minOutputAmount`, `maxInputAmount`, `deadline`, `unwrap` are ignored when quoting — pass `0n`, `0n`, `0n`, `false`.

### Full Quote Example (viem/TypeScript)

```typescript
import { createPublicClient, http, parseAbi } from "viem";
import { sei } from "viem/chains";

const publicClient = createPublicClient({
  chain: sei,
  transport: http(),
});

const SPLIT_ROUTER = "0xCaCbF14f07AF3Be93C6fc8bF0a43f9279F792199" as const;

const splitRouterAbi = parseAbi([
  "struct SwapParams { uint8 poolType; address pool; address inputToken; address outputToken; uint24 feePips; uint8 inputPercentage; uint8 outputPercentage; }",
  "struct ExactInputParams { uint256 inputAmount; uint256 minOutputAmount; uint256 deadline; bool unwrap; SwapParams[] route; }",
  "struct ExactOutputParams { uint256 outputAmount; uint256 maxInputAmount; uint256 deadline; bool unwrap; SwapParams[] route; }",
  "function quoteExactInput(ExactInputParams params) external returns (uint256 inputAmount, uint256 outputAmount, uint256 impactPips)",
  "function quoteExactOutput(ExactOutputParams params) external returns (uint256 inputAmount, uint256 outputAmount, uint256 impactPips)",
]);

type SwapParams = {
  poolType: number;
  pool: `0x${string}`;
  inputToken: `0x${string}`;
  outputToken: `0x${string}`;
  feePips: number;
  inputPercentage: number;
  outputPercentage: number;
};

type RouteResponse =
  | { status: "success"; dex: { route: SwapParams[] } }
  | { status: "error"; message: unknown };

async function getQuote(
  tokenIn: string,
  tokenOut: string,
  amount: string,
  type: "exactIn" | "exactOut"
) {
  // Step 1: Get optimal route from the API
  const params = new URLSearchParams({
    amount,
    tokenInAddress: tokenIn,
    tokenOutAddress: tokenOut,
    type,
  });

  const res = await fetch(
    `https://sei-api.dragonswap.app/api/v4/quote?${params}`
  );
  const data = (await res.json()) as RouteResponse;

  if (data.status !== "success") {
    throw new Error(`Route discovery failed: ${JSON.stringify(data.message)}`);
  }

  // Map API route to SwapParams structs
  const route = data.dex.route.map((hop) => ({
    poolType: hop.poolType,
    pool: hop.pool,
    inputToken: hop.inputToken,
    outputToken: hop.outputToken,
    feePips: hop.feePips,
    inputPercentage: hop.inputPercentage,
    outputPercentage: hop.outputPercentage,
  }));

  // Step 2: Call the on-chain quote function
  if (type === "exactIn") {
    const { result } = await publicClient.simulateContract({
      address: SPLIT_ROUTER,
      abi: splitRouterAbi,
      functionName: "quoteExactInput",
      args: [
        {
          inputAmount: BigInt(amount),
          minOutputAmount: 0n,
          deadline: 0n,
          unwrap: false,
          route,
        },
      ],
    });

    const [inputAmount, outputAmount, impactPips] = result;
    return { inputAmount, outputAmount, impactPips };
  }

  const { result } = await publicClient.simulateContract({
    address: SPLIT_ROUTER,
    abi: splitRouterAbi,
    functionName: "quoteExactOutput",
    args: [
      {
        outputAmount: BigInt(amount),
        maxInputAmount: 0n,
        deadline: 0n,
        unwrap: false,
        route,
      },
    ],
  });

  const [inputAmount, outputAmount, impactPips] = result;
  return { inputAmount, outputAmount, impactPips };
}

// Usage
const WSEI = "0xE30feDd158A2e3b13e9badaeABaFc5516e95e8C7";
const USDC = "0xe15fC38F6D8c56aF07bbCBe3BAf5708A2Bf42392"; // 6 decimals

// Sell exactly 1 WSEI for USDC
const sell = await getQuote(WSEI, USDC, "1000000000000000000", "exactIn");
console.log("USDC out:", sell.outputAmount.toString());
console.log("Price impact (pips):", sell.impactPips.toString());

// Buy exactly 1 USDC with WSEI
const buy = await getQuote(WSEI, USDC, "1000000", "exactOut");
console.log("WSEI in:", buy.inputAmount.toString());
```

---

## Pools

### GET /api/v1/pools
All pools above minimum TVL. No params.

Response: `{ status, tokens: Token[], incentives: Incentive[], pools: Pool[] }` — `incentives` includes ended programs.

Pool: `pool_address`, `token0_address`, `token1_address`, `first_token_reserve`, `second_token_reserve` (token units, not wei), `liquidity` (USD), `daily_volume`, `daily_fees` (USD, last 24h), `hourly_volume`, `hourly_fees` (USD, last 60 min, nullable), `hourly_swaps_count` (nullable), `type` (`"V2_POOL"` | `"V3_POOL"`), `fee_tier` (percent: `0.01`, `0.05`, `0.3` on V3; `0.3` on V2), `booster_farm_address` (nullable), `incentive_id` (nullable), `apy_booster`, `apr` (percent, `daily_fees × 365 / liquidity × 100`), `base_apr`, `boosted_apr`, `pool_created_at`, `liquidity_change_24h`, `volume_change_24h` (percent, nullable)

### GET /api/v1/pools/{address}
Response: `{ status, pools: Pool, incentives: Incentive[] }` — `pools` is a single object (plural key); `incentives` = this pool's programs, incl. ended. Unknown pool → 404 `pool_not_found`.

### GET /api/v1/pools/{address}/stats
Query: `weekly`, `monthly`, `quarterly`, `all_time` — approximate downsampling targets (≈ that many points; omitted or ≥ stored count → full series). Each must be ≥ 1 (`0` → 500). `daily` is always full.

Response: `{ status, liquidity_24h: { low, high } | null, stats: { daily, weekly, monthly, quarterly, all_time } }` — each entry: `{ liquidity, daily_volume (rolling 24h at snapshot time), hourly_volume, hourly_fees, hourly_swaps_count (null in older snapshots), created_at }`. Unknown pool → 200 with empty series and `liquidity_24h: null`.

### GET /api/v1/pools/{address}/transactions
The pool's 5,000 most recent swaps, newest first. No pagination (use `/activity`). Unknown pool → 404 `pool_not_found`.

Response: `{ status, transactions: Transaction[], stats }` — `stats` covers the last 24h, not the returned array: `tx_count`, `buy_volume_usd`, `sell_volume_usd`, `total_volume_usd` (= pool `daily_volume`), `total_buys_count`, `total_sells_count`, `buyers_count`, `sellers_count`, `makers_count` (unique buyers + sellers)

Transaction: `{ pool_address, first_token_address, second_token_address, first_amount_token, second_amount_token, block_number, tx_hash, from, total_value (USD), type: "Swap", action: "Buy"|"Sell", created_at, pool_type: "v2"|"v3" }` — amounts in token units; first/second token is not always the pool's `token0` (it flips with `action`).

### GET /api/v1/pools/{address}/activity
Block-cursor pagination. Query: `swaps_from_block`, `mints_from_block`, `burns_from_block` (default `0`; inclusive, `block_number >=`). Unknown pool → 404 `pool_not_found`.

Response: `{ status, swaps: Transaction[], mints: [], burns: [] }` — at most 500 per array, oldest first. Next page: pass the last event's `block_number`; the cursor is inclusive, so skip events you already have (e.g. by `tx_hash`). `mints`/`burns` are currently always empty.

---

## Tokens

### GET /api/v1/tokens
Tokens above the TVL threshold.

Response: `{ status, tokens[] }` — each: `address`, `name`, `symbol`, `usd_price`, `decimals`, `liquidity` (USD), `description` (nullable), `verified` (on the approved list), `category` (nullable: `wrapped`, `stablecoin`, `defi`, `lst`, `meme`, `native`, `ai`), `change: { hourly, daily, weekly, monthly }` (price %), `liquidity_change_24h`, `volume_change_24h` (nullable), `sparkline_7d: [{ day: "YYYY-MM-DD", price }]` (7 entries, oldest first)

### GET /api/v1/tokens/approved
Official token list, included regardless of liquidity (some have `liquidity: 0`).

Response: `{ status, tokens[] }` — each: `address`, `name`, `symbol`, `usd_price`, `decimals`, `liquidity`, `daily_volume` (USD, 24h), `daily_transactions` (cumulative), `verified` (always `true`), `category`, `description`, `website` (nullable), `socials: { x?, telegram?, discord?, github? } | null`, `liquidity_change_24h`, `volume_change_24h` (always `null` here)

### GET /api/v1/tokens/{address}
Response: `{ status, token }` — same fields as `/tokens/approved`, with `verified` = on the approved list and `liquidity_change_24h`/`volume_change_24h` populated (nullable). Unknown → 404 `token_not_found`.

### GET /api/v1/tokens/{address}/stats
No query params; every series is returned in full. Unknown → 404 `token_not_found`.

Response: `{ status, price_24h: { low, high }, stats: { daily, weekly, monthly, quarterly, all_time } }` — each entry: `{ usd_price, liquidity, daily_volume (rolling 24h at snapshot time), created_at }`

---

## Stats & Transactions

### GET /api/v1/stats
DEX-wide statistics. No params.

Response: `{ status, stats: { v2, v3, rfq, all } }` — buckets: `v2` = DEX V1 pools, `v3` = DEX V2 pools, `rfq` = RFQ trades, `all` = sum of the three (use for DEX-wide totals). Each bucket: `total_volume_usd`, `total_liquidity_usd`, `total_number_of_swaps`, `total_users_count` (same in every bucket), `24_hour_tx_count`, `24_hour_fees_usd`, `24_hour_volume_usd`, `hourly_data[]` (last 6 months), `daily_data[]` (one per UTC day; last = current, incomplete day). Entries, oldest first: `{ created_at (start of hour/day), volume_usd, total_liquidity_usd, inflows_usd, outflows_usd, fees_usd }`. In `rfq`, liquidity/inflows/outflows are `0`.

### GET /api/v1/transactions
Most recent swaps across all pools (currently 3,001), newest first. No params.

Response: `{ status, tokens: Token[], transactions: Transaction[] }`

### GET /api/v1/vol?address={wallet}
Daily volume for a wallet. Object keyed by `YYYY-MM-DD`, newest first, one key per day with swaps: `{ "2025-12-10": { volume_usd, swap_fee_usd, count } }`. No `status` field. Missing/invalid address or no swaps → 200 `[]` (empty array, not object).

---

## Farms

### GET /api/v1/farms
All farms, including ended ones (currently all have ended; keep `end_timestamp > now` for active; `apy`/`tvl` are still reported).

Response: `{ status, tokens[], pools: Pool[], farms: Farm[] }` — `tokens` = reward, booster and pool tokens; `pools` = one per distinct `pooled_token_address`.

Farm: `{ farm_address, pooled_token_address (LP token), reward_token_address, booster_token_address (null for basic), apy (percent), tvl (USD), type: "basic"|"boosted", unlock_rate (reward tokens/s), booster_unlock_rate (null for basic), is_lp_pooled_token, start_timestamp, end_timestamp, total_rewards_usd, campaign (nullable, e.g. "DS Epoch 2"), is_funded (currently always null) }`

### GET /api/v1/farms/{address}
Response: `{ status, tokens[], farm }` — `tokens` = reward token (+ booster for boosted); `farm` = Farm without `is_funded`, plus `info` (HTML string or null). Unknown → 404 `farm_not_found`.

### GET /api/v1/farms/{address}/history
All event types in one array, newest first. No pagination; at most 1,000 records per type (`deposits`, `withdraws`, `payouts`), so busy farms are truncated. Unknown → 404 `farm_not_found`.

Response: `{ status, tokens[], history[] }` — each: `{ type: "deposits"|"withdraws"|"payouts", timestamp, user, pooled_token_address, block_number, farm_address, tx_hash }` plus `amount` (deposits/withdraws, token units) or `reward_token_claim`, `booster_token_claim` (payouts, token units; booster `0` for basic farms).

### GET /api/v1/farms/{address}/stats
Historical farm stats. Response: `{ status, stats: { daily, weekly, monthly, all_time } }` — each: `{ created_at, deposits, deposits_usd }`

---

## Staking

Use the staking contract `0xb71886c52d754cf2b1d3d866c7cfe9eec01418a7` as `{address}` / `{staker}`.

### GET /api/v1/staking/{address}
Response: `{ status, emission_timestamps: number[] }` — upcoming reward emission times. `{address}` is ignored (any value returns the same data).

### GET /api/v1/staking/{address}/emissions
Reward emissions (DRG or USDC) deposited for stakers; same records as `protocol_rewards` in `/rewards`. Newest first, no pagination. Any other address → `deposits: []`.

Response: `{ status, deposits[] }` — each: `{ tx_hash, token_address (DRG 0x0a526e425809aea71eb279d24ae22dee6c92a4fe or USDC 0xe15fc38f6d8c56af07bbcbe3baf5708a2bf42392), amount (number, token units), amount_wei (string), created_at (string, Unix s), apr (percent), reward_per_usd_staked, token_usd_price, buy_tx_hash (nullable) }`

### GET /api/v1/staking/{address}/rewards
`{address}` is ignored. Arrays newest first, not paginated.

Response: `{ status, fees: { total_withdrawal_fees_drg, total_marketplace_fees_drg, total_protocol_rewards_usd }, withdrawl_fees[], marketplace_fees[], protocol_rewards[] }`
- `total_*_fees_drg`: despite the suffix, ≈ the USD value of the fees (not the sum of `drg_fee`). `total_protocol_rewards_usd` = Σ `amount` × `token_usd_price` of `protocol_rewards`.
- `withdrawl_fees[]` (API spelling), `marketplace_fees[]`: `{ tx_hash, created_at (string), drg_fee (number, DRG), reward_per_usd_staked (decimal string, may use exponent notation, e.g. "8.1500662470371e-05") }`
- `protocol_rewards[]`: same records as `/emissions` `deposits`.

### GET /api/v1/staking/{address}/stats
Any other address → `total_staked: 0` and empty arrays.

Response: `{ status, total_staked (DRG, token units; = latest total_deposits), stats: { daily, weekly, monthly, all_time } }` — each entry: `{ total_deposits (DRG), total_deposits_usd, total_fees, apr (percent), created_at }`. `daily`: about every 10 min (~144 entries), oldest first. `weekly`/`monthly`: currently a single entry (the latest). `all_time`: irregular intervals, oldest first, ending with the latest.

### GET /api/v1/staking/{address}/activity/{user}
Any other `{address}` → 404 `staker_address_not_found`. User with no history → 200 with four empty arrays. Arrays newest first; amounts are wei strings, timestamps are strings.

Response: `{ status, data: { reward_activity[], staking_activity[], marketplace_activity[], staking_bonus[] } }`
- `reward_activity`: `{ type: "Claim"|"Compound"|"Autocompound"|"Autocompound Update", user, token ("1"=DRG, "2"=USDC; null on Autocompound Update), token_operation (set on Autocompound Update), claimed_amount_wei (Claim), drg_amount_wei, usdc_amount_wei (Compound, Autocompound), tx_hash, timestamp }`
- `staking_activity`: `{ type: "Deposit"|"Withdraw Start"|"Instant Withdraw", user, amount_wei, fees (Instant Withdraw only), tx_hash, timestamp }`
- `marketplace_activity`: `{ type: "Listed"|"Price updated"|"Removed"|"Bought"|"Sold", user, amount_wei, price_in_USDC_wei (both null on Removed), bought_from (Bought), sold_to (Sold), tx_hash, timestamp }`
- `staking_bonus`: `{ type: "Bonus", user, token (currently the staking contract address), amount (decimal string), amount_wei, amount_usd_value (decimal string), tx_hash, timestamp }`

### GET /api/v1/staking/{address}/stats/{user}
Any other `{address}` → 404 `staker_address_not_found`.

Response: `{ status, stats: { total_DRG_earned, total_USDC_earned, total_staked } | null }` — wei strings (DRG 18 decimals, USDC 6); `null` when the user has no staking history.

---

## User Data

### GET /api/v1/user/{address}/pools
User's liquidity positions. Invalid address → 200 with empty arrays.

Response: `{ status, tokens[], pools[], incentives[] }` — `tokens` = both sides of every pool. Each `V3_POOL` position is its own entry (the same `pool_address` can repeat; no token ID or tick range). Entry = Pool market fields (`pool_created_at` nullable) plus `lp_balance` (`V2_POOL` only; absent on `V3_POOL`), `first_token_amount`, `second_token_amount` (token units), `total_liquidity_usd` (USD value of the position).

### GET /api/v1/user/{address}/portfolio
Invalid address → HTTP 500 with an HTML page.

Response: `{ status, meta: { native_token_usd_price, change: { hourly, daily, weekly, monthly } }, portfolio_history: [{ timestamp, liquidity_usd, farm_balance_usd }], tokens: [{ address, name, symbol, usd_price, decimals, change, user_balance }], total_usd_value, native_token: { balance, usd_price } }`
- `meta` = SEI market data, same for every wallet (`meta.change` is SEI's price change, not the portfolio's).
- `portfolio_history`: oldest first, irregular intervals (can be one entry); the last entry is close to now.
- `tokens` excludes native SEI; `user_balance` in token units; `change` = token price %.
- `total_usd_value` = `tokens` + native SEI + latest `liquidity_usd`; excludes `farm_balance_usd`.

### GET /api/v1/user/{address}/farms
Farms where the user has a non-zero balance (the balance itself is not returned — use `/farms/{farm}` history or portfolio `farm_balance_usd`). Invalid address → 200 with empty arrays.

Response: `{ status, tokens[] (reward + booster), pools: Pool[] (one per pooled_token_address), farms: Farm[] }`

### GET /api/v1/user/{address}/farms/{farm}
No activity or invalid address → 200 with empty `history` and zero `payouts`. Unknown farm → 404 `farm_not_found`.

Response: `{ status, tokens[], farm: Farm, history[], payouts: { reward_token_claims, total_booster_claims } }` — `history` = the user's events in this farm, newest first (same shape as farm history; `user` lowercase); `payouts` = sums of the user's claims (`0` if none).

### GET /api/v1/user/{address}/stakings/{staker}
`{staker}` is ignored (any value returns the same events). No stakes or invalid address → 200 `events: []`.

Response: `{ status, events[] }` — each: `{ amount (wei string), deposited_at, transaction_hash, stake_index (from 0), unlock_timestamp, stake_locked_tx_hash, withdraw_tx_hash, emergency_withdraw_tx_hash (nullable), payouts: [{ amount (wei string), blockTimestamp (string, Unix s), rewardToken, transactionHash }] }` — payout keys are camelCase.

---

## Incentives

### GET /api/v1/incentives
All programs, including ended ones (compare `end_time` with now).

Response: `{ status, incentives: Incentive[] }`

Incentive: `{ start_time, end_time, reward (number, token units), reward_token, refundee, pool, incentive_id, apr (decimal string, percent; computed per request and non-zero even after the program ends), title (nullable free text), liquidity_usd (decimal string), owner (nullable), campaign (nullable) }`

### GET /api/v1/incentives/{incentive_id}/tokens
Response: `{ status, token_ids: number[] }` — DEX V2 position NFT IDs staked in the program. Unknown/malformed ID → 200 `token_ids: []`.

### GET /api/v1/incentives/user/{address}
No positions or invalid address → 200 with empty arrays.

Response: `{ status, staked_tokens: number[] (position NFT IDs staked in incentives), deposited_tokens: number[] (position NFT IDs deposited), tokens[], incentives: Incentive[], pools: Pool[] }`

---

## Airdrops

### GET /api/v1/airdrops
All campaigns, including ended ones (after `ends_at`). Response: `{ status, airdrops[] }`

Airdrop: `{ name, address, token_address, number_of_users, timestamps[] (vesting milestones, Unix s; one per period, matching `amounts`), vesting_text, total_claimed, total_deposited, starts_at, ends_at }`

### GET /api/v1/airdrops/{airdrop}
Response: `{ status, airdrop: Airdrop }`. Unknown/malformed → 404 `{"status":"fail"}`.

### GET /api/v1/airdrops/{airdrop}/user/{user}
Response: `{ status, is_eligible: boolean, amounts: string[] (decimal, token units, one per `timestamps` entry), amounts_wei: string[] }`. Not eligible, invalid user, or unknown airdrop → 200 with `is_eligible: false` and empty arrays.

---

## Common Data Types

**Token:** `{ address, name, symbol, usd_price, decimals }`

**Pool:** see `GET /api/v1/pools`. **Transaction:** see `GET /api/v1/pools/{address}/transactions`. **Farm:** see `GET /api/v1/farms`. **Incentive:** see `GET /api/v1/incentives`.

## Integration Checklist

1. Use `https://sei-api.dragonswap.app`; data under `/api/v1/`, quotes under `/api/v4/`
2. For quoting: call `/api/v4/quote` first, then `SplitRouter.quoteExactInput()` / `quoteExactOutput()` via `simulateContract`
3. Map `data.dex.route` directly to `SwapParams[]`
4. Quote-only calls: `minOutputAmount`/`maxInputAmount` `0n`, `deadline` `0n`, `unwrap` `false`
5. For `exactOut`, take amounts from on-chain `quoteExactOutput`, not from the API
6. Use `viem` with the `sei` chain for on-chain interactions
7. Call the API from a server (CORS allowlist); data can be up to ~2 min old
8. Check HTTP status and `status`; gateway 404/405 bodies are plain text, not JSON
9. Paginate pool activity with inclusive `block_number` cursors and dedupe by `tx_hash`
10. Filter farms, incentives and airdrops by end time — lists include ended programs
11. Pool/token stats return `{ daily, weekly, monthly, quarterly, all_time }`; staking stats `{ daily, weekly, monthly, all_time }`
12. Timestamps are Unix seconds (strings in staking endpoints)
13. Token amounts are decimal-adjusted numbers; wei amounts are strings
14. USD values are numbers, except incentive `liquidity_usd`/`apr` and staking `amount_usd_value` (strings); some fields are nullable
