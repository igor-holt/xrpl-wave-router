# Proposal: keep hard filters, replace linear score with squared-norm energy ranking

Upstream: https://github.com/Shellfish011235/xrpl-wave-router

Tracking: https://github.com/igor-holt/genesis-conductor-ecosystem/issues/3

Research mode / quote-only. Not a custody, token, or production-settlement change.

## Why this instead of rewriting the product

The pathfinder already has the right split:

1. Hard constraints reject illegal offers.
2. A ranker picks among survivors.
3. A ledger adapter reserves / posts / voids.
4. Open Payments and XRPL stay behind adapters.
5. `PROJECT-BOUNDARIES.md` keeps this in research mode.

The weak piece is only step 2. `findBestRoute()` uses

```
score = 0.5 * norm(cost) + 0.2 * norm(latency) + 0.3 * (1 - quality)
```

That affine mix lets a zero-price stub win even when quality, privacy residual, or execution cost is bad. The `shellfish-research-bridge` offer at `priceMicrounits: 0` is the concrete example.

## Proposal

Keep the existing filter block exactly. Rank only the eligible set with

```
E(d) = ||phi(d)||^2
p(d) proportional to exp(-E(d))
```

where `d` is a job + provider pair and `phi` includes at least:

- normalized cost
- normalized latency
- quality risk `(1 - quality)`
- privacy gap (0 if rank >= requested)
- optional heuristic thermo term `(1 - eta)`
- optional hardware / active-parameter residual for local or edge providers

Then either `r* = argmin E` or sample at temperature `T`.

Hard constraints stay outside the energy. Empty support returns the existing 422.

## What not to change

- Do not put interior bookkeeping on XRPL.
- Do not enable live Open Payments or TigerBeetle by finishing the ranker.
- Do not grant agents signing authority.
- Keep research-mode / quote-only as the default.
- Treat any thermo ratio as heuristic until a live energy oracle exists.

## Suggested code shape

- `src/services/router.ts` — keep `eligible`; replace `score` with `energy` / `logP`.
- New `src/services/energy.ts` — `phi`, `E = ||phi||^2`, unnormalized `-E`.
- Quote / job responses add `energy`, `unnormalizedLogProb`, `reasons[]`.
- Optional later lanes are more rows in `providerOffers`.

## Acceptance

- Existing README curl `/jobs` still works.
- Zero-price low-quality offers lose to a priced, constrained, higher-quality offer.
- `/quote` and `/jobs` use the same energy for the same request.
- Reserve amount equals selected `priceMicrounits`.
- `PROJECT-BOUNDARIES.md` unchanged.
