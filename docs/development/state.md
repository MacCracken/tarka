# tarka — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.2.1** — 2026-06-22. **akshara wired — tarka consumes the shared tokenizer.**
Rollout prompts now come from a real tokenized corpus (`corpus_set` + `gd_ld`) instead
of a synthetic vocab; demo tokenizes 45 B → vocab 28, rewards the space token, freq
**1.00 → 24.00 / 24**, grad-checks 4/4 green. (0.2.0 = M1 core REINFORCE; 0.1.0 = M0
scaffold.) **Remaining to close M1:** de-feature attn11's `--objective rl` (its own cut).

## Toolchain

- **Cyrius pin**: `6.2.36` (in `cyrius.cyml [package].cyrius`) — matches the
  installed toolchain; same 6.2.x band attn11/rosnet consume.

## Source

**M1 REINFORCE core — shipped at 0.2.0; akshara wired at 0.2.1:**
- `src/rl.cyr` — the minimal rosnet-backed policy (embedding + linear head;
  `π(next|cur) = softmax(E[cur]@W + b)`) + on-policy REINFORCE (rollout → reward →
  advantage `(R − EMA baseline)` → advantage-weighted softmax-CE backward) + compact
  bias-corrected Adam. Dogfoods rosnet `linear_fwd`/`linear_bwd` + tyche sampling.
  `rl_prompt` draws from a real akshara-tokenized corpus when one is loaded.
- `src/main.cyr` — demo: tokenizes a corpus via akshara (`corpus_set`), rewards the
  space token; rollout frequency rises **1.00 → 24.00 / 24** under REINFORCE.

**Remaining in M1:** de-feature attn11's `--objective rl` (its own cut — REINFORCE
now lives here).

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
- **[akshara](https://github.com/MacCracken/akshara) 0.1.0** (wired 0.2.1) — the
  sovereign tokenizer (अक्षर); `corpus_set` builds the byte vocab + packed store,
  `gd_ld` indexes it for rollout prompts. Pure path only — loader/streaming/emit
  consumer symbols (`secure_read_file`/`read_stdin`/`file_seek`) are stubbed unreached.
  The third attn11 extraction after rosnet/tyche; one tokenizer, shared with attn11.

## Consumers

_None yet._ (Long-horizon: a sovereign agent/reasoning runtime — daimon-adjacent —
is the eventual downstream.)

## Next

M1 — migrate attn11's `--objective rl` (REINFORCE) here, re-expressed on rosnet.
See [`roadmap.md`](roadmap.md).
