---
name: solver-integration-nima-rfq
description: Use when building a solver or market maker that provides liquidity to the Nima RFQ Protocol - implementing HTTP or WebSocket endpoints, reversing intents, signing EIP-712 maker intents, handling quote requests, or debugging solver-side RFQ integration
---

# Nima RFQ Protocol - Solver Integration Reference

## Quick Reference

| Property | Value |
|----------|-------|
| Production URL | `https://dev-rfq.saphyre.xyz` |
| Testnet URL | `https://testnet-dev-rfq.saphyre.xyz` |
| WSS Endpoint | `wss://dev-rfq.saphyre.xyz/api/v1/solver-stream` |
| Sei Mainnet Chain ID | `1329` |
| Permit2 (Sei) | `0xC6b7aC7Bbd8b456b67e8440694503cAC2Afb1d98` |
| Signing Standard | EIP-712 via Permit2 (`PermitWitnessTransferFrom` + `MakerSwapIntent` witness) |
| Nonce Strategy | Random 256-bit (unordered bitmap) |
| WSS Encoding | MessagePack (binary frames) |
| WSS Auth Timeout | 5 seconds |
| WSS Heartbeat | ping every 30s, markets refresh every 60s |
| Max WSS Connections/IP | 3 |

## Overview

Solvers provide liquidity to the Nima RFQ Protocol by responding to quote requests and signing counter-intents for atomic on-chain settlement. A solver receives quote requests from the RFQ Coordinator, returns competitive prices, and signs reversed swap intents using EIP-712 via Permit2. Two transport options are available: HTTP (REST) and WebSocket (WSS).

## Integration Flow

### Choose Transport

| | HTTP (REST) | WebSocket (WSS) |
|---|---|---|
| **Direction** | Coordinator calls your API | You connect to coordinator |
| **Serialization** | JSON | MessagePack (binary) |
| **Infrastructure** | Requires public HTTPS endpoint | No public endpoint needed |
| **Best for** | Simple integrations | High-performance, Rust/Go/Python |

### Steps (both transports)

1. **Register** — Contact the RFQ Protocol team with your solver name, endpoint/transport, supported markets, and auth key.
2. **Fetch config** — `GET /api/v1/config?chain_id=1329` to get Permit2 and RFQSettlement addresses. Never hardcode.
3. **Approve Permit2** — For every token you trade: `ERC20.approve(permit2_address, maxUint256)`.
4. **Implement markets** — Return your supported pairs with `max_size` derived from on-chain balances.
5. **Implement quotes** — Price the post-fee `amount` from the coordinator. Echo back `quote_id` and numeric `expiry`.
6. **Implement intent signing** — Reverse the taker intent (swap tokens/amounts, set counterparty), sign with EIP-712.
7. **Handle trade results** — Log `tx_hash` notifications for analytics (fire-and-forget, no response needed).
8. **Test end-to-end** — Verify a complete quote -> sign -> settlement cycle on testnet.

## Prerequisites

| Requirement | Description |
|-------------|-------------|
| **Solver Wallet** | EOA with sufficient token balances for trading |
| **API Endpoint** | HTTPS endpoint (HTTP transport) or WebSocket connection (WSS transport) |
| **Price Feed** | Real-time price data for supported markets |
| **Permit2 Approval** | Approve Permit2 to spend your tokens (`ERC20.approve(permit2_address, maxUint256)`) |
| **Auth Key** | Shared secret from the RFQ Protocol team |

## Registration

Contact the RFQ Protocol team to register. Provide:

| Information | Description |
|-------------|-------------|
| Solver Name | Human-readable identifier |
| API Endpoint | HTTPS URL (HTTP) or indicate WSS preference |
| Supported Markets | Token pairs you'll provide liquidity for |
| Auth Key | Shared secret for request authentication |

Registration is currently manual. Contact the protocol team to get whitelisted.

## Base URLs

| Environment | URL |
|-------------|-----|
| Production | `https://dev-rfq.saphyre.xyz` |
| Testnet | `https://testnet-dev-rfq.saphyre.xyz` |

## Configuration

Fetch contract addresses dynamically from `/api/v1/config`:

```bash
curl "https://dev-rfq.saphyre.xyz/api/v1/config?chain_id=1329"
```

```json
{
  "status": "ok",
  "chain_id": "1329",
  "chain_name": "Sei Mainnet",
  "rfq_settlement_address": "0x...",
  "permit2_address": "0xC6b7aC7Bbd8b456b67e8440694503cAC2Afb1d98",
  "min_quote_ttl_ms": 15000,
  "max_quote_ttl_ms": 30000
}
```

Use `rfq_settlement_address` as the `spender` and `permit2_address` as the `verifyingContract` when signing intents. Do not hardcode these.

---

## HTTP Transport

The coordinator calls YOUR endpoints. Requires a publicly accessible HTTPS endpoint.

### Authentication

All requests from the coordinator include:

```
Authorization: Bearer <your_auth_key>
```

### POST /api/v1/markets

Returns the markets you support. Called by the coordinator periodically.

**Request:**

```json
{ "chain_id": "1329" }
```

**Response:**

```json
{
  "status": "ok",
  "markets": [
    {
      "input_token": "0x3894085ef7ff0f0aedf52e2a2704928d1ec074f1",
      "output_token": "0xe30fedd158a2e3b13e9badaeabafc5516e95e8c7",
      "min_size": 100.00,
      "max_size": 5000.00
    },
    {
      "input_token": "0xe30fedd158a2e3b13e9badaeabafc5516e95e8c7",
      "output_token": "0x3894085ef7ff0f0aedf52e2a2704928d1ec074f1",
      "min_size": 100.00,
      "max_size": 12000.00
    }
  ]
}
```

Return both directions for each pair (A->B and B->A as separate entries). `max_size` should reflect your output token balance converted to input token units. If the coordinator does not receive updated market data within the freshness window, your markets will be considered stale and you will stop receiving quote requests.

### POST /api/v1/quote

Returns a price quote. The `amount` field is the **post-fee** amount (protocol fee already deducted by the coordinator).

**Request:**

```json
{
  "quote_id": "550e8400-e29b-41d4-a716-446655440000",
  "input_token": "0x...",
  "output_token": "0x...",
  "amount": "997000000",
  "chain_id": "1329",
  "expiry_timestamp": 1705612830000
}
```

| Field | Description |
|-------|-------------|
| `quote_id` | Coordinator-generated UUID - echo back in response |
| `amount` | Input amount in wei (**post-fee**, fee already deducted) |
| `expiry_timestamp` | Coordinator-provided expiry - use this as your `expiry` in response |

**Response:**

```json
{
  "status": "ok",
  "quote_id": "550e8400-e29b-41d4-a716-446655440000",
  "input_token": "0x...",
  "output_token": "0x...",
  "amount_in": "997000000",
  "amount_out": "498500000000000000",
  "expiry": 1705612830000,
  "chain_id": "1329"
}
```

Required fields in response: `quote_id`, `amount_out`, `expiry`. Other fields are optional echoes.

### POST /api/v1/intent

Signs the reversed maker intent. Called when a user submits a signed trade intent.

**Request:**

```json
{
  "quote_id": "550e8400-e29b-41d4-a716-446655440000",
  "chain_id": "1329",
  "user_address": "0x1234...",
  "intent": {
    "inputToken": "0xe30fedd158a2e3b13e9badaeabafc5516e95e8c7",
    "outputToken": "0xe15fc38f6d8c56af07bbcbe3baf5708a2bf42392",
    "inputAmount": "975000000000000000",
    "outputAmount": "249250000"
  }
}
```

| Field | Description |
|-------|-------------|
| `intent.inputToken` | Taker's input token |
| `intent.outputToken` | Taker's output token |
| `intent.inputAmount` | Taker's input amount (**post-fee**) - use as maker's `outputAmount` |
| `intent.outputAmount` | Taker's output amount - use as maker's `inputAmount` |

**Response:**

```json
{
  "status": "ok",
  "quote_id": "550e8400-e29b-41d4-a716-446655440000",
  "solver_address": "0xsolveraddress...",
  "signed_swap_intent": {
    "swap_intent": {
      "counterparty": "0x1234...",
      "inputToken": "0xe15fc38f6d8c56af07bbcbe3baf5708a2bf42392",
      "outputToken": "0xe30fedd158a2e3b13e9badaeabafc5516e95e8c7",
      "inputAmount": "249250000",
      "outputAmount": "975000000000000000",
      "unwrap": false
    },
    "signature_params": {
      "deadline": "1705613400",
      "nonce": "98765432109876543210",
      "signer": "0xsolveraddress...",
      "signature": "0x..."
    }
  }
}
```

### POST /api/v1/trade-result

Receives the settlement transaction hash. **Fire-and-forget** - the coordinator does not wait for or use the response.

**Request:**

```json
{
  "quote_id": "550e8400-e29b-41d4-a716-446655440000",
  "tx_hash": "0x1234567890abcdef..."
}
```

Use the `tx_hash` to correlate on-chain settlements with local trade records. The transaction may still be pending confirmation when received.

---

## WebSocket Transport

Solvers connect **to** the coordinator via persistent WebSocket. No public endpoint needed. Recommended for Rust, Go, Python, and high-performance stacks.

### Connection

```
wss://dev-rfq.saphyre.xyz/api/v1/solver-stream
```

All messages are **MessagePack-encoded binary** WebSocket frames (RFC 6455).

### Message Envelope

Every message follows this structure:

```typescript
{
  action: string;         // Message type
  request_id?: string;    // UUID for request/response correlation
  data?: unknown;         // Payload
  success?: boolean;      // Response: true/false
  error?: {               // Error details (when success=false)
    code: string;
    message: string;
  };
}
```

### Connection Lifecycle

1. **Connect** to WSS URL
2. **Authenticate** - send `auth` message within 5 seconds or connection closes
3. **Markets sync** - coordinator sends `get_markets` immediately after auth, then every 60 seconds
4. **Heartbeat** - coordinator sends native WebSocket ping every 30s (pong is automatic)
5. **Quotes** - coordinator sends `get_quote` when users request prices
6. **Intents** - coordinator sends `sign_intent` when users submit swaps
7. **Trade results** - coordinator sends `trade_result` after settlement (fire-and-forget)

### Authentication

First message must be auth within 5 seconds:

**Send:**
```json
{
  "action": "auth",
  "data": { "token": "your-plain-text-auth-key" }
}
```

**Success:**
```json
{
  "action": "auth_success",
  "data": { "solver_id": 9, "solver_name": "my-solver" }
}
```

**Failure:** Connection closed with code `4003`.

### get_markets / markets_response

**Receive (Coordinator -> Solver):**
```json
{
  "action": "get_markets",
  "request_id": "2198702c-bcad-4110-baae-e62e7b34ea71",
  "data": { "chain_id": "1329" }
}
```

**Send (Solver -> Coordinator):**
```json
{
  "action": "markets_response",
  "request_id": "2198702c-bcad-4110-baae-e62e7b34ea71",
  "success": true,
  "data": [
    {
      "chain_id": "1329",
      "input_token": "0xe30fedd158a2e3b13e9badaeabafc5516e95e8c7",
      "output_token": "0xe15fc38f6d8c56af07bbcbe3baf5708a2bf42392",
      "min_size": 100,
      "max_size": 5000
    }
  ]
}
```

Fields per market: `chain_id` (optional, inferred from request), `input_token` (required), `output_token` (required), `min_size` (optional), `max_size` (optional, derived from on-chain balances).

### get_quote / quote_response

**Receive:**
```json
{
  "action": "get_quote",
  "request_id": "a1b2c3d4-...",
  "data": {
    "quote_id": "550e8400-...",
    "chain_id": "1329",
    "input_token": "0x...",
    "output_token": "0x...",
    "amount": "1000000000000000000",
    "expiry_timestamp": 1738780830000
  }
}
```

The `amount` is the **post-fee** input amount. `expiry_timestamp` is coordinator-provided.

**Send:**
```json
{
  "action": "quote_response",
  "request_id": "a1b2c3d4-...",
  "success": true,
  "data": {
    "quote_id": "550e8400-...",
    "amount_out": "249250000",
    "expiry": 1738780830000,
    "chain_id": "1329"
  }
}
```

Required response fields: `quote_id`, `amount_out`, `expiry`. The `expiry` field **must be a number** (not a string).

**Error response:**
```json
{
  "action": "quote_response",
  "request_id": "a1b2c3d4-...",
  "success": false,
  "error": { "code": "MARKET_NOT_FOUND", "message": "No market for this pair" }
}
```

### sign_intent / intent_response

**Receive:**
```json
{
  "action": "sign_intent",
  "request_id": "b2c3d4e5-...",
  "data": {
    "quote_id": "550e8400-...",
    "chain_id": "1329",
    "user_address": "0x1234...",
    "intent": {
      "inputToken": "0xe30fedd158a2e3b13e9badaeabafc5516e95e8c7",
      "outputToken": "0xe15fc38f6d8c56af07bbcbe3baf5708a2bf42392",
      "inputAmount": "975000000000000000",
      "outputAmount": "249250000"
    }
  }
}
```

`intent.inputAmount` is post-fee. Use as maker's `outputAmount`.

**Send:**
```json
{
  "action": "intent_response",
  "request_id": "b2c3d4e5-...",
  "success": true,
  "data": {
    "quote_id": "550e8400-...",
    "solver_address": "0x7d34...",
    "signed_swap_intent": {
      "swap_intent": {
        "counterparty": "0x1234...",
        "inputToken": "0xe15fc38f6d8c56af07bbcbe3baf5708a2bf42392",
        "outputToken": "0xe30fedd158a2e3b13e9badaeabafc5516e95e8c7",
        "inputAmount": "249250000",
        "outputAmount": "975000000000000000",
        "unwrap": false
      },
      "signature_params": {
        "deadline": "1705612890",
        "nonce": "98765432109876543210",
        "signer": "0x7d34...",
        "signature": "0x..."
      }
    }
  }
}
```

### trade_result

**Receive (fire-and-forget, no response needed):**
```json
{
  "action": "trade_result",
  "data": {
    "quote_id": "550e8400-...",
    "tx_hash": "0x1234567890abcdef..."
  }
}
```

### Close Codes

| Code | Name | Description |
|------|------|-------------|
| 4001 | AUTH_TIMEOUT | Auth not received within timeout |
| 4002 | STALE_CONNECTION | No pong received within heartbeat timeout |
| 4003 | UNAUTHORIZED | Invalid auth token, wrong protocol, or malformed auth |
| 4005 | TRY_AGAIN_LATER | Too many connections from same IP |
| 4006 | UNKNOWN_ACTION | Unrecognized message action |
| 4008 | SUPERSEDED | Same solver connected again, old connection replaced |

### Rate Limiting

- **Max connections per IP:** 3
- If a solver connects with a solver_id that already has an active connection, the old connection is closed with code `4008`

### Reconnection

Implement exponential backoff with jitter:

```
delay = min(1000 * 2^attempt, 30000) + random jitter (25%)
```

---

## Intent Reversal

When the coordinator forwards a user's signed intent, the solver must create a **reversed** maker intent:

### Reversal Rules

| Taker Field | Maker Field | Transformation |
|-------------|-------------|----------------|
| `inputToken` | `outputToken` | Swapped |
| `outputToken` | `inputToken` | Swapped |
| `inputAmount` | `outputAmount` | Swapped (this is the post-fee amount) |
| `outputAmount` | `inputAmount` | Swapped |
| (none) | `counterparty` | Set to taker's address |
| `unwrap` | `unwrap` | **Independent** - solver decides |

```typescript
function reverseIntent(intent: Intent, takerAddress: string): MakerSwapIntent {
  return {
    counterparty: takerAddress,       // ALWAYS set to prevent front-running
    inputToken: intent.outputToken,   // Swap tokens
    outputToken: intent.inputToken,
    inputAmount: intent.outputAmount, // Swap amounts
    outputAmount: intent.inputAmount, // This is post-fee amount from coordinator
    unwrap: false,                    // Solver decides independently
  };
}
```

**CRITICAL:** Always set `counterparty` to the taker's address. Setting it to zero exposes the trade to front-running attacks.

The solver's `unwrap` flag is independent of the taker's. Each party decides for themselves whether to receive unwrapped native tokens.

### Fee-Adjusted Amounts

The `intent.inputAmount` received from the coordinator is already **post-fee** (taker's input minus protocol fee). Use it directly as the maker's `outputAmount`. The on-chain settlement validates: `maker.outputAmount == taker.inputAmount - fee`.

---

## EIP-712 Signing Specification (Maker)

The maker signs a `PermitWitnessTransferFrom` message with a `MakerSwapIntent` witness.

### Domain

```json
{
  "name": "Permit2",
  "chainId": 1329,
  "verifyingContract": "<permit2_address from /api/v1/config>"
}
```

### Types

```json
{
  "PermitWitnessTransferFrom": [
    { "name": "permitted", "type": "TokenPermissions" },
    { "name": "spender", "type": "address" },
    { "name": "nonce", "type": "uint256" },
    { "name": "deadline", "type": "uint256" },
    { "name": "witness", "type": "MakerSwapIntent" }
  ],
  "TokenPermissions": [
    { "name": "token", "type": "address" },
    { "name": "amount", "type": "uint256" }
  ],
  "MakerSwapIntent": [
    { "name": "counterparty", "type": "address" },
    { "name": "inputToken", "type": "address" },
    { "name": "outputToken", "type": "address" },
    { "name": "inputAmount", "type": "uint256" },
    { "name": "outputAmount", "type": "uint256" },
    { "name": "unwrap", "type": "bool" }
  ]
}
```

### Message Values

| Field | Value |
|-------|-------|
| `permitted.token` | Maker's input token (= taker's output token) |
| `permitted.amount` | Maker's input amount (= taker's output amount) |
| `spender` | `rfq_settlement_address` from `/api/v1/config` |
| `nonce` | Random 256-bit integer |
| `deadline` | Unix timestamp (seconds) - use short deadline (e.g., 60s) |
| `witness.counterparty` | Taker's address (from `user_address`) |
| `witness.inputToken` | Taker's `outputToken` |
| `witness.outputToken` | Taker's `inputToken` |
| `witness.inputAmount` | Taker's `outputAmount` |
| `witness.outputAmount` | Taker's `inputAmount` (post-fee) |
| `witness.unwrap` | Whether to unwrap maker's output token |

---

## Nonce Strategy

Permit2 uses **unordered nonces** with bitmap tracking. Generate a random 256-bit nonce for each signature. Do NOT fetch a "current nonce".

```typescript
// TypeScript
import crypto from 'crypto';
const nonce = BigInt('0x' + crypto.randomBytes(32).toString('hex'));
```

```python
# Python
import secrets
nonce = secrets.randbits(256)
```

```rust
// Rust
use rand::Rng;
let nonce: [u8; 32] = rand::thread_rng().gen();
let nonce = U256::from_big_endian(&nonce);
```

---

## Fee Mechanics (Solver Perspective)

The coordinator handles fee deduction before sending requests to solvers:

1. User requests quote with `amount` (e.g., 1,000,000)
2. Coordinator calculates fee: `fee = amount * feePips / 1,000,000` (e.g., 3,000 at 0.3%)
3. Coordinator sends `amount - fee` to solver (e.g., 997,000) as the `amount` in quote request
4. Solver quotes output based on this post-fee amount
5. When signing intent: `intent.inputAmount` is the post-fee amount - use as maker's `outputAmount`

On-chain validation: `maker.outputAmount == taker.inputAmount - fee`

---

## Node.js WSS Example

```javascript
import WebSocket from "ws";
import { encode, decode } from "@msgpack/msgpack";

const WS_URL = "wss://dev-rfq.saphyre.xyz/api/v1/solver-stream";
const AUTH_KEY = process.env.MM_AUTH_KEY;

function connect() {
  const ws = new WebSocket(WS_URL);

  ws.on("open", () => {
    // First message must be auth
    ws.send(encode({ action: "auth", data: { token: AUTH_KEY } }));
  });

  ws.on("message", (raw) => {
    const msg = decode(raw);

    switch (msg.action) {
      case "auth_success":
        console.log(`Authenticated as ${msg.data.solver_name}`);
        break;

      case "get_markets":
        ws.send(encode({
          action: "markets_response",
          request_id: msg.request_id,
          success: true,
          data: [
            {
              chain_id: msg.data.chain_id,
              input_token: "0xe30fedd158a2e3b13e9badaeabafc5516e95e8c7",
              output_token: "0xe15fc38f6d8c56af07bbcbe3baf5708a2bf42392",
              min_size: 100,
              max_size: 5000,
            },
            {
              chain_id: msg.data.chain_id,
              input_token: "0xe15fc38f6d8c56af07bbcbe3baf5708a2bf42392",
              output_token: "0xe30fedd158a2e3b13e9badaeabafc5516e95e8c7",
              min_size: 100,
              max_size: 12000,
            },
          ],
        }));
        break;

      case "get_quote": {
        const { quote_id, chain_id, amount, expiry_timestamp } = msg.data;
        const amountOut = calculateOutput(amount); // your pricing logic
        ws.send(encode({
          action: "quote_response",
          request_id: msg.request_id,
          success: true,
          data: {
            quote_id,
            amount_out: amountOut,
            expiry: expiry_timestamp, // must be a number, not string
            chain_id,
          },
        }));
        break;
      }

      case "sign_intent": {
        const signedIntent = signReversedIntent(msg.data); // your signing logic
        ws.send(encode({
          action: "intent_response",
          request_id: msg.request_id,
          success: true,
          data: signedIntent,
        }));
        break;
      }

      case "trade_result": {
        // Fire-and-forget -- no response needed
        const { quote_id, tx_hash } = msg.data;
        saveTradeResult(quote_id, tx_hash);
        break;
      }
    }
  });

  ws.on("close", (code, reason) => {
    console.log(`Disconnected: ${code} ${reason}`);
    // Reconnect with exponential backoff
    setTimeout(connect, 1000 + Math.random() * 500);
  });
}

connect();
```

---

## Rust WSS Example

```rust
use tokio_tungstenite::{connect_async, tungstenite::Message};
use futures_util::{StreamExt, SinkExt};
use rmp_serde::{to_vec, from_slice};
use serde_json::{json, Value};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let url = "wss://dev-rfq.saphyre.xyz/api/v1/solver-stream";
    let (ws, _) = connect_async(url).await?;
    let (mut tx, mut rx) = ws.split();

    // Authenticate
    let auth = json!({ "action": "auth", "data": { "token": "your-key" } });
    tx.send(Message::Binary(to_vec(&auth)?.into())).await?;

    while let Some(Ok(Message::Binary(data))) = rx.next().await {
        let msg: Value = from_slice(&data)?;
        let request_id = msg["request_id"].clone();

        match msg["action"].as_str().unwrap_or("") {
            "auth_success" => {
                println!("Authenticated as {}", msg["data"]["solver_name"]);
            }
            "get_markets" => {
                let resp = json!({
                    "action": "markets_response",
                    "request_id": request_id,
                    "success": true,
                    "data": [{
                        "chain_id": msg["data"]["chain_id"],
                        "input_token": "0xe30fedd158a2e3b13e9badaeabafc5516e95e8c7",
                        "output_token": "0xe15fc38f6d8c56af07bbcbe3baf5708a2bf42392",
                        "min_size": 100,
                        "max_size": 5000
                    }, {
                        "chain_id": msg["data"]["chain_id"],
                        "input_token": "0xe15fc38f6d8c56af07bbcbe3baf5708a2bf42392",
                        "output_token": "0xe30fedd158a2e3b13e9badaeabafc5516e95e8c7",
                        "min_size": 100,
                        "max_size": 12000
                    }]
                });
                tx.send(Message::Binary(to_vec(&resp)?.into())).await?;
            }
            "get_quote" => {
                let d = &msg["data"];
                let amount_out = calculate_output(&d["amount"]); // your pricing
                let resp = json!({
                    "action": "quote_response",
                    "request_id": request_id,
                    "success": true,
                    "data": {
                        "quote_id": d["quote_id"],
                        "amount_out": amount_out,
                        "expiry": d["expiry_timestamp"],  // must be a number
                        "chain_id": d["chain_id"]
                    }
                });
                tx.send(Message::Binary(to_vec(&resp)?.into())).await?;
            }
            "sign_intent" => {
                let signed = sign_reversed_intent(&msg["data"]); // your signing
                let resp = json!({
                    "action": "intent_response",
                    "request_id": request_id,
                    "success": true,
                    "data": signed
                });
                tx.send(Message::Binary(to_vec(&resp)?.into())).await?;
            }
            "trade_result" => {
                // Fire-and-forget -- no response needed
                let quote_id = msg["data"]["quote_id"].as_str().unwrap_or("");
                let tx_hash = msg["data"]["tx_hash"].as_str().unwrap_or("");
                save_trade_result(quote_id, tx_hash);
            }
            _ => {}
        }
    }

    Ok(())
}
```

---

## Complete Intent Signing Example (TypeScript)

```typescript
import { createWalletClient, http } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';
import { sei } from 'viem/chains';
import crypto from 'crypto';

// Get addresses from /api/v1/config
const PERMIT2_ADDRESS = '0xC6b7aC7Bbd8b456b67e8440694503cAC2Afb1d98';
const RFQ_SETTLEMENT_ADDRESS = '0x...'; // from config
const CHAIN_ID = 1329;

const account = privateKeyToAccount(process.env.SOLVER_PRIVATE_KEY as `0x${string}`);
const walletClient = createWalletClient({
  account,
  chain: sei,
  transport: http(process.env.CHAIN_RPC_URL),
});

interface Intent {
  inputToken: string;
  outputToken: string;
  inputAmount: string;
  outputAmount: string;
}

async function signReversedIntent(data: {
  quote_id: string;
  chain_id: string;
  user_address: string;
  intent: Intent;
}) {
  // 1. Reverse the intent
  const makerIntent = {
    counterparty: data.user_address,
    inputToken: data.intent.outputToken,
    outputToken: data.intent.inputToken,
    inputAmount: data.intent.outputAmount,
    outputAmount: data.intent.inputAmount,
    unwrap: false,
  };

  // 2. Generate nonce and deadline
  const nonce = BigInt('0x' + crypto.randomBytes(32).toString('hex'));
  const deadline = BigInt(Math.floor(Date.now() / 1000) + 60); // 60 seconds

  // 3. Sign EIP-712
  const signature = await walletClient.signTypedData({
    domain: {
      name: 'Permit2',
      chainId: CHAIN_ID,
      verifyingContract: PERMIT2_ADDRESS as `0x${string}`,
    },
    types: {
      PermitWitnessTransferFrom: [
        { name: 'permitted', type: 'TokenPermissions' },
        { name: 'spender', type: 'address' },
        { name: 'nonce', type: 'uint256' },
        { name: 'deadline', type: 'uint256' },
        { name: 'witness', type: 'MakerSwapIntent' },
      ],
      TokenPermissions: [
        { name: 'token', type: 'address' },
        { name: 'amount', type: 'uint256' },
      ],
      MakerSwapIntent: [
        { name: 'counterparty', type: 'address' },
        { name: 'inputToken', type: 'address' },
        { name: 'outputToken', type: 'address' },
        { name: 'inputAmount', type: 'uint256' },
        { name: 'outputAmount', type: 'uint256' },
        { name: 'unwrap', type: 'bool' },
      ],
    },
    primaryType: 'PermitWitnessTransferFrom',
    message: {
      permitted: {
        token: makerIntent.inputToken as `0x${string}`,
        amount: BigInt(makerIntent.inputAmount),
      },
      spender: RFQ_SETTLEMENT_ADDRESS as `0x${string}`,
      nonce,
      deadline,
      witness: {
        counterparty: makerIntent.counterparty as `0x${string}`,
        inputToken: makerIntent.inputToken as `0x${string}`,
        outputToken: makerIntent.outputToken as `0x${string}`,
        inputAmount: BigInt(makerIntent.inputAmount),
        outputAmount: BigInt(makerIntent.outputAmount),
        unwrap: makerIntent.unwrap,
      },
    },
  });

  // 4. Return signed intent
  return {
    quote_id: data.quote_id,
    solver_address: account.address,
    signed_swap_intent: {
      swap_intent: {
        counterparty: makerIntent.counterparty,
        inputToken: makerIntent.inputToken,
        outputToken: makerIntent.outputToken,
        inputAmount: makerIntent.inputAmount,
        outputAmount: makerIntent.outputAmount,
        unwrap: makerIntent.unwrap,
      },
      signature_params: {
        deadline: deadline.toString(),
        nonce: nonce.toString(),
        signer: account.address,
        signature,
      },
    },
  };
}
```

---

## Environment Variables Reference

```bash
# Connection
RFQ_SERVER_WS_URL=wss://dev-rfq.saphyre.xyz/api/v1/solver-stream
MM_AUTH_KEY=your-plain-text-auth-key

# Chain
CHAIN_ID=1329
CHAIN_RPC_URL=https://evm-rpc.sei-apis.com
SOLVER_PRIVATE_KEY=0x...

# Quote pricing
SPREAD_BPS=30  # 0.3% spread
```

Fetch `rfq_settlement_address` and `permit2_address` from `/api/v1/config` rather than hardcoding.

---

## Security Best Practices

- **Never hardcode private keys.** Use environment variables or a secure key management service.
- **Always set counterparty** to the taker's address in reversed intents.
- **Validate all inputs** - verify quote_id matches one you issued, amounts are correct.
- **Use short deadlines** (e.g., 60 seconds) for maker signatures.
- **Monitor positions** - track token balances and inventory. On-chain balances determine `max_size`.
- **Keep markets fresh** - respond to market refresh requests promptly. Stale markets stop receiving quotes.
- **Implement rate limiting** to prevent abuse of your endpoints.
- **Verify balance** before signing - ensure sufficient output token balance.

---

## Verification Checklist

1. Solver registered with RFQ Protocol team
2. Permit2 approved for all trading tokens (`ERC20.approve(permit2, maxUint256)`)
3. Wallet has sufficient token balances
4. For HTTP: HTTPS endpoint is publicly accessible
5. For WSS: Can connect and authenticate (`auth_success` received)
6. Markets endpoint returns valid directional pairs with `max_size` based on balances
7. Quote endpoint returns valid `amount_out` with echoed `quote_id` and numeric `expiry`
8. Intent signing correctly reverses tokens/amounts, sets counterparty, uses random nonce
9. EIP-712 signature uses correct domain (Permit2), types (MakerSwapIntent), and message values
10. Trade result notifications are handled for analytics/auditing
11. Reconnection implemented with exponential backoff (WSS)
12. Private keys stored securely, never in code
