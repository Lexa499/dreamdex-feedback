# dreamDEX Perps Builder Bounty — Feedback Track Submission

Bounty page: https://dorahacks.io/hackathon/dreamdex-perps-bounty/detail (Feedback track — Track 1)

**Audited:** `@somnia-chain/markets-sdk` **0.33.0** (`dist/perp/`) and the official **dreamdex-bot-kit** (`packages/perp-core`, `packages/core`, `strategies/perp-maker`) on Somnia Tokyo testnet.

## What was found
Two SDK/bot-kit bugs, both in `dreamdex-bot-kit` (Somnia Tokyo testnet):

1. **`feedback/01-perp-operator-silent-rejection.md` (major, SDK)** — perp operator path reports silent order rejection as success (closed-position illusion → liquidation risk).
   - `packages/perp-core/src/operator.ts` (`send()` L143-159, `placeFor()` L188) checks only the transaction receipt status. The pool's `placeOrderFor` returns `(bool success, uint128 orderId)` and can mine with `success=false` (no `OrderPlaced` log) — the bot then believes a closing order was placed. In `perp-guard` this means a protective close is "reported done" while the position stays open and can be liquidated. The repo's own spot path (`packages/core/src/execute.ts:134-141`) guards against exactly this (`SIM_FALSE` + `SILENT_REJECTION`), proving the omission is unintended.
2. **`feedback/02-cancelquietly-orphan-quotes.md` (major, SDK)** — `cancelQuietly` swallows every cancel failure → orphaned resting quotes + position-cap bypass.
   - `packages/perp-core/src/orders.ts` L211-225 catches **all** cancel exceptions and only logs them. The `perp-maker` strategy (`strategies/perp-maker/src/index.ts:131-135`) then unconditionally resets local state and re-quotes, leaving the old order **still live on-chain** but untracked. The orphaned quote can fill at a stale price → unintended long/short, and `MAX_POSITION` gating (computed from local state) is bypassed.

Each file follows the bounty template: Severity, Area, Steps to reproduce, Expected, Actual, Evidence, Environment.

## Why these are not security reports sent by email
Both bugs live in the **open-source bot kit** (MIT sample code, the target of Track 1 "API and SDK issues"), not in the on-chain contracts or a hosted service. The bounty page explicitly routes *contract/app security vulnerabilities* to security@dreamdex.io; these two are developer-experience/liveness bugs in public sample code — exactly the class Track 1 pays for. No private vulnerability details are exposed.

## Reproduction
Steps use only: node >= 18, `@somnia-chain/markets-sdk@0.33.0`, dreamdex-bot-kit checked out at main, a funded Somnia Tokyo testnet key with a linked perp trading key (Link a Bot guide). Each report lists the exact file:line.
