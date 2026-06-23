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

Scaffold stub only (`src/main.cyr` prints identity + dep-wiring banner). No RL or
reasoning code yet — that lands at M1.

## Tests

- `tests/tarka.tcyr` — primary suite (smoke; passes on `cyrius test`)
- `tests/tarka.bcyr` — benchmark stub
- `tests/tarka.fcyr` — fuzz stub

Grad-check discipline (every hand-derived backward verified against central finite
differences before it lands, inherited from attn11) begins at M1.

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
