# cancelQuietly swallows every cancel failure — orphaned resting quotes and position-cap bypass
Severity: major
Area: SDK

**Scope:** dreamdex-bot-kit (`packages/perp-core/src/orders.ts` + `strategies/perp-maker/src/index.ts`), Somnia Tokyo testnet.

## Summary
`cancelQuietly` catches **all** exceptions from the on-chain cancel and only logs them. The comment says this is for the benign race "order already filled/expired", but the same catch also hides real failures: bot-key permission errors (the operator is no longer allowed to cancel for that market), RPC outages, and gas failures. The `perp-maker` strategy then unconditionally resets its local resting-order state and re-quotes, leaving the old order **still live on-chain** but untracked by the bot.

## Steps to reproduce
1. Run the `perp-maker` strategy (strategies/perp-maker/src/index.ts) with a trading key that has lost cancel permission for the market (or with an RPC endpoint that fails writes).
2. Let the mark move past `REQUOTE_BPS` so the re-quote branch runs (index.ts:131-135): both `resting.bid`/`resting.ask` are passed to `cancelQuietly`.
3. `cancelFor` reverts (permission error) — `cancelQuietly` catches it and only logs `cancel <id> skipped: ...` (orders.ts:211-225).
4. The strategy continues: `resting = { atMark: 0 }` and fresh bid+ask are placed.
5. Query the pool: the OLD order is still resting on-chain and can fill at the stale price, while the bot's local inventory model no longer knows it exists.

## Expected
A cancel that fails for a reason other than "already filled/expired" must abort re-quoting and trigger reconciliation of on-chain open orders before the bot continues.

## Actual
Every failure is swallowed; the bot builds a second (third, ...) untracked live quote. Consequences:
- Orphaned quote can fill at a stale price → unintended long or short.
- `MAX_POSITION` gating is computed from local state, so the bot can exceed its own position cap.
- On a perp market, double exposure = direct loss exposure for the operator.

## Evidence (code)
`packages/perp-core/src/orders.ts:211-225`:

```ts
export async function cancelQuietly(ctx: WriteCtx, market: PerpMarket, orderId: bigint): Promise<void> {
  ...
  try {
    if (ctx.tradingKey) {
      await cancelFor(ctx.tradingKey, market.info.poolAddress, orderId);
    } else {
      await ctx.exchange.trader.cancelOrder({ pool: market.info.poolAddress, orderId });
    }
  } catch (err) {
    // An order that filled or expired between the read and the cancel is not a
    // failure of the cancel; a cascade of these means the quote is too slow.
    console.log(`cancel ${orderId} skipped: ${(err as Error).message.slice(0, 120)}`);
  }
}
```

`strategies/perp-maker/src/index.ts:131-135` — unconditional reset after the swallow:

```ts
for (const id of [resting.bid, resting.ask]) {
  if (id !== undefined) await cancelQuietly(ctx, market, id);
}
resting = { atMark: 0 };
```

Note: the operator module itself (`operator.ts`) writes a dedicated help message when a key cannot cancel a market — proof that permission failures are an expected, distinguishable case, yet they are swallowed here.

## Suggested fix
Decode the revert/error: tolerate only explicit "already filled / already expired / order not found" codes; rethrow everything else. On rethrow, stop quoting and reconcile by enumerating open on-chain orders for the owner before resuming.

## Environment
- dreamdex-bot-kit: main branch, packages/perp-core + strategies/perp-maker
- Network: Somnia Tokyo testnet
- DoraHacks account email: lyin499@gmail.com
