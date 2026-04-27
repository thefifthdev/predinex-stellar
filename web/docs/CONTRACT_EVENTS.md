# Contract Event Schemas

This document is the canonical reference for every event emitted by the Predinex Soroban smart contract (`contracts/predinex/src/lib.rs`). Frontend developers and indexer consumers should rely on this document rather than reverse-engineering event payloads from the contract source.

---

## Overview

The contract uses Soroban's `env.events().publish(topics, data)` API. Each event carries:

| Field    | Description                                                   |
|----------|---------------------------------------------------------------|
| `topics` | A tuple of `Symbol` (event name), a schema version `Symbol`, and additional identifiers |
| `data`   | The event payload — a single value or a tuple                 |

All events are emitted on the **Stellar testnet** during development and the **Stellar mainnet** for production. Use a Soroban event filter or a horizon endpoint to subscribe.

---

## Event versioning (issue #175)

Every emitted event carries an explicit **schema version marker** at a fixed topic position so indexers and frontend consumers can route by version and reject events whose schema they do not yet understand instead of silently mis-decoding payloads.

### Topic layout

```
topics[0] = Symbol(event_name)        // e.g. "create_pool"
topics[1] = Symbol(schema_version)    // currently "v1"
topics[2..] = identifiers             // pool_id, user, etc. (event-specific)
```

The version marker is **always at topic position 1**, regardless of which event family is emitted. This lets a Soroban topic filter target a specific schema version positionally:

```ts
// Subscribe only to v1 create_pool events
filters: [{ type: "contract", contractIds: [CONTRACT_ID], topics: [["create_pool", "v1"]] }]
```

The current schema version emitted by the contract is exposed by the public constant `predinex::EVENT_SCHEMA_VERSION` (`"v1"`).

### Upgrade rules for consumers

1. **Pin the version you understand.** Decoders must read `topics[1]` and only proceed if the value matches a version they know how to decode. Unknown versions must be skipped (and logged), never silently coerced.
2. **A version bump means breaking change.** Any change to topic layout, identifier order, or the data shape of an event is a breaking change — the contract bumps the version marker (e.g. `"v2"`) and re-emits under the new version. Two versions are never published for the same event in the same release.
3. **New optional payload fields keep the same version.** Backward-compatible extensions — adding a new field to a struct payload that older decoders can ignore — reuse the existing version marker. The contract will document such additions in the changelog below without bumping the version.
4. **Two versions during migration.** If a future migration ever needs to emit both `vN` and `vN+1` for the same event family during a transition window, this document will explicitly call that out per event. By default, exactly one version is emitted per event family per release.

### Sample decoder

The reference decoder in `web/app/lib/soroban-event-service.ts` reads the version marker from `topics[1]` and dispatches by `(name, version)` pair. A minimal sketch:

```ts
const SUPPORTED_EVENT_SCHEMA_VERSION = 'v1';

function decode(raw: { topic: unknown[]; value: unknown }) {
  const name = scValToNative(raw.topic[0]) as string;
  const version = scValToNative(raw.topic[1]) as string | undefined;

  if (version !== SUPPORTED_EVENT_SCHEMA_VERSION) {
    // Unknown version — skip and let the operator notice via logs
    console.warn(`Skipping ${name} event with unsupported schema version "${version}"`);
    return null;
  }

  // Identifiers shift right by 1 because of the version marker
  const poolId = Number(scValToNative(raw.topic[2]));
  // …name-specific decoding follows
}
```

---

## Events

> All topic tuples below show the schema version `"v1"` at position 1. Update this section whenever the version marker bumps.

### 1. `create_pool`

Emitted when a new prediction market (pool) is created.

**Trigger:** `PredinexContract::create_pool`

**Topics tuple:**
```
(Symbol("create_pool"), Symbol("v1"), pool_id: u32)
```

**Data:**
```
(creator: Address, status: Symbol("Open"))
```

**Full TypeScript shape:**
```ts
interface CreatePoolEvent {
  topics: [eventName: string, schemaVersion: string, poolId: number];
  data: [creator: string, status: string];
}
```

**Example (decoded):**
```json
{
  "topics": ["create_pool", "v1", 42],
  "data": ["GBXXX...CREATOR_ADDRESS", "Open"]
}
```

---

### 2. `place_bet`

Emitted when a user places a bet on an outcome.

**Trigger:** `PredinexContract::place_bet`

**Topics tuple:**
```
(Symbol("place_bet"), Symbol("v1"), pool_id: u32, user: Address)
```

**Data — `BetEvent` struct:**
```
{ outcome: u32, amount: i128, amount_a: i128, amount_b: i128, total_bet: i128 }
```

| Field       | Type    | Values             | Description                                |
|-------------|---------|--------------------|--------------------------------------------|
| `outcome`   | `u32`   | `0` = A, `1` = B   | Which outcome was bet on                   |
| `amount`    | `i128`  | positive integer   | Token amount staked in this single bet     |
| `amount_a`  | `i128`  | non-negative       | User's cumulative stake on outcome A       |
| `amount_b`  | `i128`  | non-negative       | User's cumulative stake on outcome B       |
| `total_bet` | `i128`  | non-negative       | User's total stake in this pool after bet  |

**Full TypeScript shape:**
```ts
interface PlaceBetEvent {
  topics: [eventName: string, schemaVersion: string, poolId: number, user: string];
  data: {
    outcome: 0 | 1;
    amount: bigint;
    amount_a: bigint;
    amount_b: bigint;
    total_bet: bigint;
  };
}
```

**Example (decoded):**
```json
{
  "topics": ["place_bet", "v1", 42, "GBXXX...USER_ADDRESS"],
  "data": { "outcome": 0, "amount": 5000000, "amount_a": 5000000, "amount_b": 0, "total_bet": 5000000 }
}
```

---

### 3. `settle_pool`

Emitted when an authorized settler declares the winning outcome of a market.

**Trigger:** `PredinexContract::settle_pool`

**Topics tuple:**
```
(Symbol("settle_pool"), Symbol("v1"), pool_id: u32)
```

**Data tuple:**
```
(caller: Address, winning_outcome: u32, winning_side_total: i128, total_pool_volume: i128, fee_amount: i128)
```

| Field                | Type       | Description                                              |
|----------------------|------------|----------------------------------------------------------|
| `caller`             | `Address`  | The account that submitted settlement                    |
| `winning_outcome`    | `u32`      | `0` = A, `1` = B                                         |
| `winning_side_total` | `i128`     | Total tokens staked on the winning side                  |
| `total_pool_volume`  | `i128`     | `total_a + total_b` at settlement time                   |
| `fee_amount`         | `i128`     | Protocol fee skimmed from the pool (2 % of total volume) |

**Example (decoded):**
```json
{
  "topics": ["settle_pool", "v1", 42],
  "data": ["GBXXX...SETTLER", 1, 600000000, 1000000000, 20000000]
}
```

---

### 4. `claim_winnings`

Emitted when a winner claims their share of the pool.

**Trigger:** `PredinexContract::claim_winnings`

**Topics tuple:**
```
(Symbol("claim_winnings"), Symbol("v1"), pool_id: u32, user: Address)
```

**Data:**
```
winnings: i128
```

| Field      | Type    | Description                                                  |
|------------|---------|--------------------------------------------------------------|
| `winnings` | `i128`  | Net payout transferred to the user (after the 2 % protocol fee) |

**Example (decoded):**
```json
{
  "topics": ["claim_winnings", "v1", 42, "GBXXX...USER_ADDRESS"],
  "data": 9800000
}
```

---

### 5. `claim_refund`

Emitted when a user claims their original stake back from a voided pool.

**Trigger:** `PredinexContract::claim_refund`

**Topics tuple:**
```
(Symbol("claim_refund"), Symbol("v1"), pool_id: u32, user: Address)
```

**Data:**
```
refund: i128
```

---

### 6. `cancel_pool`

Emitted when the creator cancels a pool that has not yet received any bets.

**Trigger:** `PredinexContract::cancel_pool`

**Topics tuple:**
```
(Symbol("cancel_pool"), Symbol("v1"), pool_id: u32)
```

**Data:**
```
creator: Address
```

---

### 7. `void_pool`

Emitted when the creator voids an open pool, opening the way for refund claims.

**Trigger:** `PredinexContract::void_pool`

**Topics tuple:**
```
(Symbol("void_pool"), Symbol("v1"), pool_id: u32)
```

**Data:**
```
caller: Address
```

---

### 8. `assign_settler`

Emitted when the creator delegates settlement authority for a pool.

**Trigger:** `PredinexContract::assign_settler`

**Topics tuple:**
```
(Symbol("assign_settler"), Symbol("v1"), pool_id: u32)
```

**Data:**
```
(creator: Address, settler: Address)
```

---

### 9. `pool_frozen` / `pool_disputed` / `pool_unfrozen`

Emitted by the freeze admin when a pool transitions into or out of `Frozen` / `Disputed`.

**Topics tuple:**
```
(Symbol(<event_name>), Symbol("v1"), pool_id: u32)
```

**Data:**
```
caller: Address
```

---

### 10. `fee_collected`

Emitted alongside `claim_winnings` to surface the per-claim protocol fee.

**Trigger:** `PredinexContract::claim_winnings`

**Topics tuple:**
```
(Symbol("fee_collected"), Symbol("v1"), pool_id: u32)
```

**Data:**
```
fee: i128
```

---

### 11. `treasury_recipient_rotated`

Emitted when the treasury recipient address is rotated.

**Trigger:** `PredinexContract::rotate_treasury_recipient`

**Topics tuple:**
```
(Symbol("treasury_recipient_rotated"), Symbol("v1"))
```

**Data:**
```
(old_recipient: Address, new_recipient: Address)
```

---

### 12. `treasury_withdrawn`

Emitted on a successful treasury withdrawal.

**Trigger:** `PredinexContract::withdraw_treasury`

**Topics tuple:**
```
(Symbol("treasury_withdrawn"), Symbol("v1"))
```

**Data:**
```
(caller: Address, recipient: Address, amount: i128)
```

---

### 13. `freeze_admin_set`

Emitted when the freeze admin address is configured.

**Trigger:** `PredinexContract::set_freeze_admin`

**Topics tuple:**
```
(Symbol("freeze_admin_set"), Symbol("v1"))
```

**Data:**
```
freeze_admin: Address
```

---

## Parsing guide for frontend / indexers

### Topic structure

Soroban publishes topics as a `Vec<Val>`. The first element is always a `Symbol` carrying the event name, and the second element is always a `Symbol` carrying the schema version (`"v1"` today). Subsequent elements carry typed identifiers.

```ts
// Minimal helper — adapt to your Soroban SDK version
function parseEventTopic(raw: SorobanEvent): {
  name: string;
  schemaVersion: string;
  poolId?: number;
  user?: string;
} {
  const [nameVal, versionVal, poolIdVal, userVal] = raw.topic;
  return {
    name: scValToNative(nameVal) as string,
    schemaVersion: scValToNative(versionVal) as string,
    poolId: poolIdVal !== undefined ? Number(scValToNative(poolIdVal)) : undefined,
    user: userVal !== undefined ? String(scValToNative(userVal)) : undefined,
  };
}
```

### Amount units

`amount` and `winnings` are raw token units in `i128`. To convert to the human-readable token amount divide by the token's decimal factor (typically `10^7` for XLM-derived tokens):

```ts
const DECIMAL_FACTOR = 10_000_000n; // 7 decimals

function toHuman(raw: bigint): string {
  return (Number(raw) / Number(DECIMAL_FACTOR)).toFixed(7);
}
```

### Outcome mapping

| Value | Meaning   |
|-------|-----------|
| `0`   | Outcome A (`pool.outcome_a_name`) |
| `1`   | Outcome B (`pool.outcome_b_name`) |

Outcome labels are stored on the `Pool` struct, not in the event. Always look up the `Pool` to display a human-readable name.

---

## Horizon event subscription

```ts
import { Server } from "@stellar/stellar-sdk/rpc";

const server = new Server("https://soroban-testnet.stellar.org");

// Subscribe only to v1 user-relevant events for this contract
const events = await server.getEvents({
  startLedger: 0,
  filters: [
    {
      type: "contract",
      contractIds: [CONTRACT_ID],
      // Pin to schema version "v1" so a future v2 rollout doesn't silently
      // mis-decode in older clients.
      topics: [["create_pool", "v1"], ["place_bet", "v1"], ["settle_pool", "v1"], ["claim_winnings", "v1"]],
    },
  ],
});
```

---

## Changelog

| Version | Change                                                                                  |
|---------|-----------------------------------------------------------------------------------------|
| v1      | Schema version marker added at topic position 1 on every event (issue #175).            |
| v0.1    | Initial event schema documented (`create_pool`, `place_bet`, `settle_pool`, `claim_winnings`). |

> This document must be updated whenever a new event is added to the contract or an existing event's topics/data structure changes. See [CONTRACT_VERSIONING.md](./CONTRACT_VERSIONING.md) for the full migration process.
