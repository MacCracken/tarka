# tarka — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.1.0** — scaffolded 2026-06-22 via `cyrius init`. M0 (scaffold) only; no RL
surface yet. First real milestone (M1, REINFORCE migration) is v0.2.0.

## Toolchain

- **Cyrius pin**: `6.2.36` (in `cyrius.cyml [package].cyrius`) — matches the
  installed toolchain; same 6.2.x band attn11/rosnet consume.

## Source

**M1 REINFORCE core landed** (in-flight; not yet versioned — a 0.2.0 cut is the
user's call):
- `src/rl.cyr` — the minimal rosnet-backed policy (embedding + linear head;
  `π(next|cur) = softmax(E[cur]@W + b)`) + on-policy REINFORCE (rollout → reward →
  advantage `(R − EMA baseline)` → advantage-weighted softmax-CE backward) + compact
  bias-corrected Adam. Dogfoods rosnet `linear_fwd`/`linear_bwd` + tyche sampling.
- `src/main.cyr` — demo: target-token rollout frequency rises **1.56 → 24.00 / 24**
  under REINFORCE (the X024-style gate).

**Remaining in M1:** wire `[deps.akshara]` (real tokenized corpus for prompts);
de-feature attn11's `--objective rl` (separate cut).

## Tests

- `tests/tarka.tcyr` — **grad-check suite, 4/4 green**: policy backward (dW/dE/db)
  FD-verified (maxrel ≤ 2e-9); RL advantage-scaling identity exact.
- `tests/tarka.bcyr` — benchmark stub (compiles)
- `tests/tarka.fcyr` — fuzz stub (compiles)

Grad-check discipline (every hand-derived backward verified against central finite
differences before it lands, inherited from attn11) is **active as of M1**.

## Dependencies

Direct (declared in `cyrius.cyml`):

- **stdlib** — string, fmt, alloc, io, vec, str, syscalls, assert, bench, math, args
- **[rosnet](https://github.com/MacCracken/rosnet) 0.2.0** — f64 tensor storage,
  BLAS-1, matmul + gradient (CPU bundle). The policy/value networks build on this.
- **[tyche](https://github.com/MacCracken/tyche) 0.1.1** — deterministic PRNG for
  on-policy rollout sampling + weight init. (rosnet's `t_randn` resolves through it.)

**Tokenizer (M1):** the byte/BPE tokenizer is currently attn11-internal. **Decided
(user 2026-06-22): extract to a small shared lib** ("extraction for reuse"), not
vendor — tarka is its concrete second consumer, parallels rosnet/tyche. **Lib name
locked: `akshara`** (अक्षर — the indivisible unit of text/sound; user 2026-06-22) —
the third attn11 extraction after rosnet/tyche. See ADR 0001 § Consequences.

## Consumers

_None yet._ (Long-horizon: a sovereign agent/reasoning runtime — daimon-adjacent —
is the eventual downstream.)

## Next

M1 — migrate attn11's `--objective rl` (REINFORCE) here, re-expressed on rosnet.
See [`roadmap.md`](roadmap.md).
