# SkyVerdict

**SkyVerdict** is a GenLayer Intelligent Contract that automatically adjudicates flight-delay compensation claims by cross-checking passenger claims against live flight status data — with an LLM reasoning over the evidence and GenLayer's validator consensus agreeing on the outcome. No airline dispute desk, no manual claims adjuster, no oracle.

Deployed on GenLayer Studio at:

```
0x6f3144c156e546De8f6e562Fe8712B1641490A0F
```

## What it does

A passenger (or anyone submitting on their behalf) opens a claim case tied to a specific flight and booking. The contract:

1. Fetches live flight status data from a flight-status API directly on-chain (no oracle).
2. Validates that the fetched record actually matches the claimed flight and scheduled departure date.
3. Uses an LLM to classify the claim as **eligible**, **not eligible**, or **unresolved**, based on flight status (landed/cancelled/diverted) and delay duration against a configurable threshold.
4. Reaches consensus on that classification through GenLayer's leader/validator model — the validator independently re-fetches evidence and re-runs the classification to confirm the leader's result rather than trusting it blindly.
5. Records the final disposition (`COMPENSATION_APPROVED`, `CLAIM_DENIED`, or `REVIEW_REQUIRED`) on-chain, with a hash binding the decision to the exact evidence it was based on.

## Why GenLayer

Flight delay compensation is a textbook case for [why GenLayer exists](https://docs.genlayer.com/understand-genlayer-protocol/what-is-genlayer): the payout depends on judgment (was this flight really delayed past the threshold, for a reason the passenger didn't cause?), the evidence lives on the open web, and today it's resolved by slow, inconsistent airline claims departments. SkyVerdict turns that judgment call into a verifiable, machine-speed, on-chain decision — validators independently re-derive the same evidence and classification the leader proposed, so no single party (including the contract deployer) unilaterally decides who gets paid.

## Design highlights

- **Replay protection** — a case can only be opened once per `(flight_number, subject_hash)` pair, preventing duplicate claims on the same booking.
- **Evidence freshness window** — flight status records older than 3 days relative to the observation date are rejected as stale.
- **Strict identity binding** — the fetched flight record must match the claimed flight number *and* scheduled departure date exactly; a mismatch aborts classification.
- **Bounded retries** — unresolved cases can be retried up to `MAX_ATTEMPTS` times with a cooldown between attempts, and expire automatically if left pending too long.
- **Evidence-bound decisions** — every resolved case stores a SHA-256 hash tying the on-chain decision to the specific evidence payload (flight status id, delay minutes, carrier, etc.) it was derived from, plus a per-contract/attempt/chain binding so evidence can't be replayed across cases or chains.
- **Structured, auditable LLM output** — the LLM never gets to output free text; it must return a strict JSON decision (`applicability`, `match_mask`, `exclusion_mask`) validated against explicit consistency rules (e.g. `ELIGIBLE` requires a non-zero match mask and zero exclusion mask).

## Contract interface

### Write methods

| Method | Description |
|---|---|
| `open_case(case_id, flight_number, passenger_name, booking_reference, scheduled_departure_date, origin_airport, destination_airport)` | Opens a new claim case in `PENDING` status. |
| `assess_case(case_id, expected_attempt)` | Triggers evidence retrieval + LLM classification + validator consensus for a pending case. |
| `expire_pending(case_id, expected_attempt)` | Marks a case `UNRESOLVED` if it has been pending past `PENDING_EXPIRY_SECONDS` without resolution. |
| `retry_unresolved(case_id, expected_attempt)` | Re-opens an `UNRESOLVED` case for another attempt, subject to cooldown and `MAX_ATTEMPTS`. |

### View methods

| Method | Description |
|---|---|
| `read_case(case_id)` | Returns the full case record — flight number, status, disposition, subject fields, and evidence identifiers. |
| `read_attempt(case_id, attempt)` | Returns the recorded outcome of a specific attempt. |
| `read_case_by_subject(flight_number, subject_hash)` | Looks up an existing `case_id` for a given flight + subject hash, for replay-check purposes. |
| `read_contract_metadata()` | Returns contract name, version, evidence schema version, and classification. |

## Case lifecycle

```
PENDING ──assess_case──▶ ELIGIBLE / NOT_ELIGIBLE / UNRESOLVED
   │                                        │
   └──expire_pending (timeout)──▶ UNRESOLVED ┘
                                        │
                              retry_unresolved (cooldown + attempt limit)
                                        │
                                        ▼
                                    PENDING (next attempt)
```

## Compensation decision model

The LLM evaluates three independent dimensions when classifying a claim:

- **IDENTITY** — the flight status record matches the claimed flight and passenger booking context.
- **DELAY_THRESHOLD** — the flight's delay meets or exceeds the compensation threshold (120 minutes by default).
- **NOT_CANCELLED_BY_PASSENGER** — the disruption reflects an airline-side status (landed late / diverted), not a passenger no-show or self-cancellation.

`ELIGIBLE` requires at least one affirmative match dimension and zero exclusions; `NOT_ELIGIBLE` requires at least one affirmative exclusion; anything ambiguous or contradictory must resolve to `UNRESOLVED` rather than guess.

## Trying it out

This contract is deployed on GenLayer Studio. You can interact with it directly through the [Studio UI](https://studio.genlayer.com) by importing the contract address above, or via [`genlayer-js`](https://docs.genlayer.com/api-references/genlayer-js) / [`genlayer-py`](https://docs.genlayer.com/api-references/genlayer-py):

```typescript
import { createClient } from "genlayer-js";
import { studionet } from "genlayer-js/chains";

const client = createClient({ chain: studionet });

const caseState = await client.readContract({
  address: "0x6f3144c156e546De8f6e562Fe8712B1641490A0F",
  functionName: "read_contract_metadata",
  args: [],
});
```

## Disclaimer

SkyVerdict is a demonstration of GenLayer's Intelligent Contract capabilities and is not a licensed insurance or claims-adjudication product. It does not constitute legal or financial advice, and using it does not create any binding compensation obligation between real airlines and passengers unless explicitly agreed to outside the contract. See GenLayer's note on [dispute resolution not being a court](https://docs.genlayer.com/developers/intelligent-contracts/when-to-use-genlayer#dispute-resolution-is-not-a-court).
