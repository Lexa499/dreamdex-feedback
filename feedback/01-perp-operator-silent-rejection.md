# Perp operator path reports silent order rejection as success — closed-position illusion
Severity: major
Area: SDK

**Scope:** dreamdex-bot-kit (`packages/perp-core/src/operator.ts`) + @somnia-chain/markets-sdk 0.33.0 perp surface, Somnia Tokyo testnet.

## Summary
The perp operator placement path verifies an order placement by checking only the **transaction receipt status**, never the pool's boolean return value (`success`) nor the `OrderPlaced` event log. A transaction can mine with `status === "success"` while the pool **silently rejects** the order (`placeOrderFor` returns `success=false`, no log). The bot then believes the order (e.g. a position-closing order) was placed successfully.

## Steps to reproduce
1. Use the perp operator path from dreamdex-bot-kit: `packages/perp-core/src/operator.ts`, `placeFor()` (L166-189).
2. Call `placePerp` / `placeFor` against a Somnia Tokyo testnet perp pool with arguments that pass `simulateContract` but are silently rejected at execution time (e.g. self-matching configuration or a market gate evaluated post-simulation — the pool returns `success=false` without reverting).
3. Observe: the transaction receipt has `status === "success"` (tx mines fine).
4. `decodePlaceOrder(receipt, pool)` finds no `OrderPlaced` log and returns `{ orderId: undefined, fills: [] }`.
5. `placeFor` resolves normally — the caller (`placePerp` in `packages/perp-core/src/index.ts`, used by `perp-starter` / `perp-guard` strategies) reports the order as sent with no error raised.

## Expected
A placement whose pool-level `success` is false (no `OrderPlaced` log) must raise an error, the same way the SDK's own spot path does.

## Actual
`placeFor` resolves successfully. The strategy believes the position was closed / the order was placed. In `perp-guard` this means a protective close is "reported done" while the position stays open and can later be liquidated. Direct fund loss for bot operators.

## Evidence (code)
Contract ABI returned by `operator.ts:48`:

```ts
"function placeOrderFor(address owner, bool isBid, uint64 userData, uint256 price, uint256 quantity, uint64 expireTimestampNs, uint8 orderType, uint8 selfMatchingOption, address builder, uint96 builderFeeBpsTimes1k) payable returns (bool success, uint128 orderId)"
```

Perp path — only receipt status checked (`operator.ts:158-159`):

```ts
const receipt = await key.publicClient.waitForTransactionReceipt({ hash });
if (receipt.status !== "success") { throw new Error(...); }
// then:
const { orderId, fills } = decodePlaceOrder(receipt, p.pool);  // L188 — silent rejection yields {orderId: undefined, fills: []}
```

Spot path in the SAME repo defends correctly (`packages/core/src/execute.ts:134-141`), proving this is an unintended gap, not a design choice:

```ts
const [ok] = sim.result;
if (!ok) throw new GotchaError("SIM_FALSE", "placeOrder simulation returned success=false (would silently reject).");
const placed = receipt.logs.find(l => l.topics[0]?.toLowerCase() === TOPIC.OrderPlaced.toLowerCase());
if (!placed) throw new GotchaError("SILENT_REJECTION", "tx mined but no OrderPlaced log — order was rejected: " + txHash);
```

## Suggested fix
Port the spot-path guards to the perp operator path: (a) decode the returned `success` bool and throw on `false`; (b) require an `OrderPlaced` log in the receipt, throwing `SILENT_REJECTION` otherwise.

## Environment
- dreamdex-bot-kit: main branch (github.com/somnia-chain/dreamdex-bot-kit), packages/perp-core
- @somnia-chain/markets-sdk 0.33.0 (dist/perp)
- Network: Somnia Tokyo testnet
- DoraHacks account email: lyin499@gmail.com
