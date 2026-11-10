# 0002 — Refactors and optimizations preserve every bit

**Status**: Accepted
**Date**: 2026-11-09

## Context

tarka's claims are numbers: a grad-check's relative error, a demo gate's before → after,
a matched-compute forward count. They are only trustworthy if the code that produced them
is the code that ships. Every run is deterministic (fixed tyche seeds, per-prompt reseeds),
so a training run is a reproducible function of the source, and any change that reorders a
floating-point sum, skips a step, or draws one random number more or less changes the
result's low bits — and from there, after a few hundred Adam steps, its printed numbers.

The 1.2.2 P(-1) sweep had two kinds of change to make. Refactors folded four copies of the
policy-gradient seed into one kernel, two Adam loops into one, four PPO updates and three
GRPO collect loops into one each. The optimization was a per-state cache of the policy's,
the reference's and the reward model's distributions inside tarka's own loops
([architecture note 001](../architecture/001-distribution-memo.md)), which made the demo 3.5×
faster. Both kinds can silently change what a method computes, and the demo's two-decimal
output is too coarse to show it.

## Decision

**A refactor or an optimization must leave every bit of every result unchanged, on every
target; a change that moves a bit is a behavior change and is labeled as one.**

- `tests/golden.tcyr` runs every trainer and inference path at a reduced size and folds the
  raw bits of what it leaves behind — parameters, Adam moments, the RNG state, metrics,
  forward counts — into one FNV-1a digest per case (14 cases). The digests were recorded
  on 1.2.1 before the first refactor landed; they are the gate.
- x86_64 and aarch64 each have their own digest table. aarch64 fuses rosnet's packed
  multiply-add (rosnet `docs/api.md`, *Numerical reproducibility*), so its bits differ from
  x86_64's — but they must be stable on each.
- A deliberate behavior change re-records only the cases it touches, names them in the
  CHANGELOG, and keeps parameter digests apart from metric digests so that a metric fix
  cannot hide a training change. 1.2.2 re-recorded two: `ipo metric` (`ipo_mean_h` skips
  ties) and `pref ingest` (the vocab covers decoded field bytes).
- Elementwise operations may change implementation when IEEE-754 rounds each element the
  same way regardless (one multiply per element is correctly rounded on any vector width:
  `t_scale` for the seed's scaling loop). Reductions keep their order.

## Consequences

- **Positive** — the 1.2.2 refactor and the cache could be landed and reviewed one step at a
  time with a mechanical answer to "did this change anything?"; the mutation run showed the
  gate catches a cache that forgets one invalidation (`pol_adam_step`, `rm_adam_step`, a hit
  that skips the copy).
- **Negative** — an optimization that reorders a sum (a tree reduction in `linear_bwd`, an
  aggregated per-state backward, a cheaper `exp`) is off the table unless it ships as a
  labeled behavior change with re-recorded digests and re-checked gates.
- **Neutral** — the digest tables are tied to the toolchain and the substrate: a cyrius or
  rosnet bump that changes codegen bits re-records them, the way a dependency's golden dump
  does.

## Alternatives considered

- **Compare the demo's printed output only.** Too coarse: two decimals hide every change
  below 0.005, and a gate can pass on wrong numbers.
- **Tolerance-based comparison.** It would admit reordered sums, but a tolerance that fits
  one seed and step count does not fit another, and the deterministic runs make exact
  comparison free.
- **One digest table for both targets.** Not achievable without giving up rosnet's fused
  aarch64 kernels, which are rosnet's decision, not tarka's.
