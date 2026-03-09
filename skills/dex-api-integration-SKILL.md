# Dragonswap DEX API - Integration Reference

## Quick Reference

| Property | Value |
|----------|-------|
| Base URL | `https://sei-api.dragonswap.app` |
| API Prefix | `/api/v1/` (data) · `/api/v3/` (routing) |
| Authentication | None (public API) |
| Chain | Sei Network (pacific-1) |
| Chain ID | `1329` |
| SplitRouter | `0xCaCbF14f07AF3Be93C6fc8bF0a43f9279F792199` |
| Response Format | `{ "status": "ok", ... }` |

## Overview

The Dragonswap DEX API provides read-only access to pools, tokens, farms, staking, incentives, airdrops, user data, and swap quoting on the Sei Network. The quoting flow is two-step: fetch an optimal route via the API, then call the on-chain SplitRouter for exact output amounts and price impact.

---

## Quoting Flow (Two-Step)

### Step 1: Route Discovery

```
GET /api/v3/quote?amount={wei}&tokenInAddress={addr}&tokenOutAddress={addr}&type=exactIn
```

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | string | yes | Amount in wei |
| `tokenInAddress` | string | yes | Input token address |
| `tokenOutAddress` | string | yes | Output token address |
| `type` | string | yes | `exactIn` or `exactOut` |

Response: `{ status: "success", quote: { route: SwapHop[], version }, linearSubroutes }`

Each route hop: `{ pool, poolType (0=V1, 1=V2), inputToken, outputToken, feePips, inputPercentage, outputPercentage }`

### Step 2: On-chain Quote

Call `DragonswapSplitRouter` at `0xCaCbF14f07AF3Be93C6fc8bF0a43f9279F792199` via `simulateContract` (read-only `eth_call`).

**Functions:**
- `quoteExactInput(ExactInputParams)` → `(uint256 inputAmount, uint256 outputAmount, uint256 impactPips)`
- `quoteExactOutput(ExactOutputParams)` → `(uint256 inputAmount, uint256 outputAmount, uint256 impactPips)`

**ExactInputParams:** `{ inputAmount: uint256, minOutputAmount: 0n, deadline: 0n, unwrap: false, route: SwapParams[] }`
**ExactOutputParams:** `{ outputAmount: uint256, maxInputAmount: 0n, deadline: 0n, unwrap: false, route: SwapParams[] }`
**SwapParams:** `{ poolType: uint8, pool: address, inputToken: address, outputToken: address, feePips: uint24, inputPercentage: uint8, outputPercentage: uint8 }`

### Full Quote Example (viem/TypeScript)

```typescript
import { createPublicClient, http, parseAbi } from "viem";
import { sei } from "viem/chains";

const publicClient = createPublicClient({ chain: sei, transport: http() });
const SPLIT_ROUTER = "0xCaCbF14f07AF3Be93C6fc8bF0a43f9279F792199";

const splitRouterAbi = parseAbi([
  "struct SwapParams { uint8 poolType; address pool; address inputToken; address outputToken; uint24 feePips; uint8 inputPercentage; uint8 outputPercentage }",
  "struct ExactInputParams { uint256 inputAmount; uint256 minOutputAmount; uint256 deadline; bool unwrap; SwapParams[] route }",
  "function quoteExactInput(ExactInputParams params) external returns (uint256 inputAmount, uint256 outputAmount, uint256 impactPips)",
]);

// Step 1: Get route
const params = new URLSearchParams({
  amount: "1000000000000000000",
  tokenInAddress: "0xE30feDd158A2e3b13e9badaeABaFc5516e95e8C7",
  tokenOutAddress: "0x3894085Ef7Ff0f0aEDf52E2A2704928d1Ec074F2",
  type: "exactIn",
});
const data = await fetch(`https://sei-api.dragonswap.app/api/v3/quote?${params}`).then(r => r.json());

// Step 2: Map route and call on-chain
const route = data.quote.route.map((hop: any) => ({
  poolType: hop.poolType,
  pool: hop.pool,
  inputToken: hop.inputToken,
  outputToken: hop.outputToken,
  feePips: hop.feePips,
  inputPercentage: hop.inputPercentage,
  outputPercentage: hop.outputPercentage,
}));

const { result } = await publicClient.simulateContract({
  address: SPLIT_ROUTER,
  abi: splitRouterAbi,
  functionName: "quoteExactInput",
  args: [{ inputAmount: BigInt("1000000000000000000"), minOutputAmount: 0n, deadline: 0n, unwrap: false, route }],
});

const [inputAmount, outputAmount, impactPips] = result;
```

---

## Pools

### GET /api/v1/pools
Returns all pools above minimum TVL with tokens and incentives.

Response: `{ status, tokens: Token[], incentives: Incentive[], pools: Pool[] }`

Pool fields: `pool_address`, `token0_address`, `token1_address`, `first_token_reserve`, `second_token_reserve`, `liquidity` (USD), `daily_volume` (USD), `daily_fees` (USD), `type` ("V3_POOL" | "V2_POOL"), `fee_tier`, `booster_farm_address`, `incentive_id`, `apy_booster`, `apr`

### GET /api/v1/pools/{address}
Single pool with incentives. Response: `{ status, pools: Pool, incentives: Incentive[] }`

### GET /api/v1/pools/{address}/stats
Historical stats. Query params: `weekly` (max 100), `monthly` (max 200), `all_time` (max 500).

Response: `{ status, stats: { daily, weekly, monthly, all_time } }` — each entry: `{ liquidity, daily_volume, created_at }`

### GET /api/v1/pools/{address}/transactions
Swap transactions + aggregate stats.

Response: `{ status, transactions: Transaction[], stats: { tx_count, buy_volume_usd, sell_volume_usd, total_volume_usd, total_buys_count, total_sells_count, buyers_count, sellers_count, makers_count } }`

Transaction: `{ pool_address, first_token_address, second_token_address, first_amount_token, second_amount_token, block_number, tx_hash, from, total_value, type: "Swap", action: "Buy"|"Sell", created_at, pool_type }`

### GET /api/v1/pools/{address}/activity
Paginated pool activity using block cursors. Query: `swaps_from_block`, `mints_from_block`, `burns_from_block`. Max 500 per request.

Response: `{ status, swaps: Transaction[] }`

---

## Tokens

### GET /api/v1/tokens
All tokens above TVL threshold.

Response: `{ status, tokens: Token[] }`

Token: `{ address, name, symbol, usd_price, decimals, liquidity, description, change: { hourly, daily, weekly, monthly } }`

### GET /api/v1/tokens/approved
Curated verified tokens.

Response: `{ status, tokens[] }` — each with `address`, `name`, `symbol`, `usd_price`, `decimals`, `liquidity`, `daily_volume`, `daily_transactions`, `description`

### GET /api/v1/tokens/{address}
Single token. Response: `{ status, token: { address, name, symbol, usd_price, decimals, liquidity, daily_volume, daily_transactions, description } }`

### GET /api/v1/tokens/{address}/stats
Historical snapshots. Response: `{ status, stats: { daily, weekly, monthly, all_time } }` — each entry: `{ usd_price, liquidity, daily_volume, created_at }`

---

## Stats & Transactions

### GET /api/v1/stats
Global DEX statistics.

Response: `{ status, stats: { v2: { total_volume_usd, total_liquidity_usd, total_number_of_swaps, total_users_count, 24_hour_tx_count, 24_hour_fees_usd, 24_hour_volume_usd, hourly_data[] } } }`

Each hourly entry: `{ created_at, volume_usd, total_liquidity_usd, inflows_usd, outflows_usd, fees_usd }`

### GET /api/v1/transactions
Recent transactions with tokens. Response: `{ status, tokens: Token[], transactions: Transaction[] }`

### GET /api/v1/vol?address={wallet}
Daily volume by wallet. Response is date-keyed: `{ "2025-12-10": { volume_usd, swap_fee_usd, count } }`

---

## Farms

### GET /api/v1/farms
All active farms with tokens and pools.

Response: `{ status, tokens: Token[], pools: Pool[], farms: Farm[] }`

Farm: `{ farm_address, pooled_token_address, reward_token_address, booster_token_address, apy, tvl, type: "basic"|"boosted", unlock_rate, booster_unlock_rate, is_lp_pooled_token, end_timestamp, start_timestamp, total_rewards_usd, campaign, is_funded }`

### GET /api/v1/farms/{address}
Single farm. Response: `{ status, tokens: Token[], farm: Farm }`

### GET /api/v1/farms/{address}/history
Farm transaction history. Response: `{ status, tokens: Token[], history[] }`

Each history event: `{ timestamp, amount, user, type: "deposits"|"withdraws"|"payouts", pooled_token_address, block_number, farm_address, tx_hash }`. Payouts add: `reward_token_claim`, `booster_token_claim`.

### GET /api/v1/farms/{address}/stats
Historical farm stats. Response: `{ status, stats: { daily, weekly, monthly, all_time } }` — each: `{ created_at, deposits, deposits_usd }`

---

## Staking

### GET /api/v1/staking/{address}
Emission timestamps. Response: `{ status, emission_timestamps: number[] }`

### GET /api/v1/staking/{address}/emissions
Deposit records. Response: `{ status, deposits[] }` — each: `{ tx_hash, token_address, amount, amount_wei, created_at, apr, reward_per_usd_staked, token_usd_price, buy_tx_hash }`

### GET /api/v1/staking/{address}/rewards
Protocol rewards and fees. Response: `{ status, fees: { total_withdrawal_fees_drg, total_marketplace_fees_drg, total_protocol_rewards_usd }, withdrawl_fees[], marketplace_fees[], protocol_rewards[] }`

### GET /api/v1/staking/{address}/stats
Historical stats with `total_staked`. Response: `{ status, total_staked, stats: { daily, weekly, monthly } }` — each: `{ total_deposits, total_deposits_usd, total_fees, apr, created_at }`

### GET /api/v1/staking/{address}/activity/{user}
User activity. Response: `{ status, data: { reward_activity[], staking_activity[], marketplace_activity[], staking_bonus[] } }`

### GET /api/v1/staking/{address}/stats/{user}
User staking stats. Response: `{ status, stats: { total_DRG_earned, total_USDC_earned, total_staked } | null }`

---

## User Data

### GET /api/v1/user/{address}/pools
User's LP positions. Response: `{ status, tokens: Token[], pools: Pool[], incentives[] }`

### GET /api/v1/user/{address}/portfolio
Portfolio overview. Response: `{ status, meta: { native_token_usd_price, change: { hourly, daily, weekly, monthly } }, portfolio_history: [{ timestamp, liquidity_usd, farm_balance_usd }], tokens: [{ ...Token, change, user_balance }], total_usd_value, native_token: { balance, usd_price } }`

### GET /api/v1/user/{address}/farms
User's farm positions. Response: `{ status, tokens[], pools[], farms[] }`

### GET /api/v1/user/{address}/farms/{farm}
Single farm position with history. Response: `{ status, tokens[], farm: Farm, history[], payouts: { reward_token_claims, total_booster_claims } }`

### GET /api/v1/user/{address}/stakings/{staker}
User staking events. Response: `{ status, events[] }`

---

## Incentives

### GET /api/v1/incentives
All incentive programs. Response: `{ status, incentives[] }`

Incentive: `{ start_time, end_time, reward, reward_token, refundee, pool, incentive_id, apr, title, liquidity_usd, owner, campaign }`

### GET /api/v1/incentives/{incentive_id}/tokens
Token IDs staked in incentive. Response: `{ status, token_ids[] }`

### GET /api/v1/incentives/user/{address}
User incentive participation. Response: `{ status, staked_tokens[], deposited_tokens[], tokens[], incentives[], pools[] }`

---

## Airdrops

### GET /api/v1/airdrops
All campaigns. Response: `{ status, airdrops[] }`

Airdrop: `{ name, address, token_address, number_of_users, timestamps[], vesting_text, total_claimed, total_deposited, starts_at, ends_at }`

### GET /api/v1/airdrops/{airdrop}
Single campaign. Response: `{ status, airdrop: Airdrop }`

### GET /api/v1/airdrops/{airdrop}/user/{user}
User allocation. Response: `{ status, is_eligible: boolean, amounts: string[], amounts_wei: string[] }`

---

## Common Data Types

**Token:** `{ address, name, symbol, usd_price, decimals }`

**Incentive:** `{ start_time, end_time, reward, reward_token, refundee, pool, incentive_id, apr, title, liquidity_usd, owner, campaign }`

## Integration Checklist

1. Use `https://sei-api.dragonswap.app` as base URL
2. For quoting: call `/api/v3/quote` first, then `SplitRouter.quoteExactInput()` via `simulateContract`
3. Map API route response directly to `SwapParams[]` struct
4. Set `minOutputAmount: 0n`, `deadline: 0n`, `unwrap: false` for quote-only calls
5. Use `viem` with `sei` chain for on-chain interactions
6. Handle paginated endpoints (pool activity) using `block_number` cursors
7. Stats endpoints return `{ daily, weekly, monthly, all_time }` arrays
8. All timestamps are Unix timestamps (seconds)
9. All USD values are numbers, all wei values are strings
