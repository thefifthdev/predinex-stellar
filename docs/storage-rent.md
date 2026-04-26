# Storage Rent Footprint and Lifecycle Policy

<!-- #190 — documents every persistent key the contract writes, its lifetime,
and the retention assumptions frontend and indexer teams can rely on. -->

## Background

Soroban charges **rent** for persistent storage entries. Each entry must be
periodically extended (bumped) or it is evicted from the ledger. Eviction does
not destroy the data permanently — it moves to the archival layer — but it
blocks contract reads until the entry is restored via `restoreFootprint`.

All persistent entries in Predinex use `env.storage().persistent()`. The
contract does **not** use `instance()` or `temporary()` storage.

---

## Key catalogue

| `DataKey` variant | Written by | Read by | Deleted by | Notes |
|---|---|---|---|---|
| `Token` | `initialize` | all token transfers | — | Set once; never mutated |
| `TreasuryRecipient` | `initialize`, `rotate_treasury_recipient` | `withdraw_treasury`, `set_creation_fee` | — | Single address; updated atomically |
| `Treasury` | `initialize`, `claim_winnings` | `get_treasury_balance`, `withdraw_treasury` | — | Accumulates 2 % fees |
| `PoolCounter` | `initialize` (implicit via `create_pool`), `create_pool` | `get_pool_counter` | — | Monotone counter; never decremented |
| `Pool(pool_id)` | `create_pool`, `place_bet`, `settle_pool`, `freeze_pool`, `unfreeze_pool`, `dispute_pool` | `get_pool`, `claim_winnings`, all read fns | — | Central market record; updated in-place |
| `UserBet(pool_id, user)` | `place_bet` | `claim_winnings`, `get_user_bet` | — | Created on first bet; never deleted |
| `DelegatedSettler(pool_id)` | `assign_settler` | `settle_pool` | — | Optional per-pool; absent means no delegation |
| `FreezeAdmin` | `set_freeze_admin` | `freeze_pool` | — | Single optional address |
| `CreationFee` | `set_creation_fee` | `create_pool`, `get_creation_fee` | — | Defaults to 0 when absent |

---

## Lifecycle walkthrough

### 1. Initialization

`initialize` writes `Token`, `TreasuryRecipient`, and `Treasury` (set to 0).
These three entries live for the lifetime of the contract deployment and should
be bumped by the operator as part of routine maintenance.

### 2. Pool creation

`create_pool` writes `Pool(pool_id)` and increments `PoolCounter`. If a
creation fee is set, no new storage key is written — the fee is a token
transfer only.

**Bump obligation**: `Pool` entries must be kept alive at least until the pool
is settled and all winnings are claimed. For long-duration markets (days to
weeks) the operator should bump the entry before it expires. A safe default is
`max_ttl` at creation time.

### 3. Betting

`place_bet` writes or updates `UserBet(pool_id, user)` and mutates
`Pool(pool_id)` (updates `total_a`, `total_b`, `participant_count`).

**Bump obligation**: `UserBet` entries must survive until the user claims
winnings. Bump alongside the corresponding `Pool` entry.

### 4. Settlement

`settle_pool` mutates `Pool(pool_id)` in-place (sets `settled`, `status`,
`winning_outcome`). No new keys are written.

The enriched settlement event emits `(caller, winning_outcome,
winning_side_total, total_pool_volume, fee_amount)` — indexers and the
frontend can reconstruct payout context from the event alone.

### 5. Claim

`claim_winnings` reads `Pool(pool_id)` and `UserBet(pool_id, user)`, performs
the token transfer, and increments `Treasury`. Neither entry is deleted after
the claim — the `UserBet` record is retained as an on-chain audit trail.

**Future improvement**: A post-claim tombstone could mark the bet as claimed to
prevent double-claim bugs, and expired entries could be archived after all
participants have claimed.

### 6. Treasury operations

`withdraw_treasury` reads and zeroes `Treasury`. `rotate_treasury_recipient`
updates `TreasuryRecipient`.

---

## Retention assumptions for consumers

| Concern | Safe assumption |
|---|---|
| Is a pool's outcome queryable after settlement? | Yes — `Pool` is never deleted |
| Can an indexer reconstruct payouts from events? | Yes — the `settle_pool` event carries all necessary totals |
| Is a user's bet history available on-chain? | Yes — `UserBet` is never deleted |
| Can a claimed `UserBet` be distinguished from an unclaimed one? | **No** — the current implementation does not mark bets as claimed |
| Do evicted entries block reads? | Yes — evicted `Pool` or `UserBet` entries must be restored before they can be read |

---

## Operational guidance

- Bump `Token`, `TreasuryRecipient`, `Treasury`, and `PoolCounter` periodically
  (e.g. weekly) as part of a maintenance transaction.
- Bump `Pool(pool_id)` and all associated `UserBet(pool_id, *)` entries at
  creation time using `max_ttl`, and again before expiry for long-duration
  markets.
- Monitor for eviction using the Stellar Horizon API or a Soroban event
  indexer; restore via `restoreFootprint` before any contract call that reads
  an evicted key.
