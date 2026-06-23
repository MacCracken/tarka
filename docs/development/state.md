# tarka — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.3.0** — 2026-06-22. **M2 — GRPO / PPO + value critic.** The advantage-based RL
family on the minimal rosnet policy: value critic + GAE + PPO clipped surrogate + GRPO,
adversarially design-reviewed then grad-checked (**19/19**). **Acceptance met** — on a
parity control task PPO/GRPO are more sample-efficient than REINFORCE (median rollouts-
to-threshold: **PPO 32, GRPO 160, REINFORCE 192**). (0.2.1 akshara wired; 0.2.0 M1
REINFORCE; 0.1.0 scaffold.) **M1 closed** at attn11 1.11.1 (RL migrated here).

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

**M1 closed:** attn11 de-featured `--objective rl` at **attn11 1.11.1** (RL lives here).

**M2 — actor-critic / PPO / GRPO — shipped at 0.3.0:**
- `src/m2.cyr` — value critic `V(s)=w_v·E[s]+b_v` (shared-E, MSE-regressed), **GAE**
  (γ=0.99 λ=0.95), **PPO** clipped surrogate (frozen π_old, ratio, clip 0.2, multi-epoch
  via `ppo_train`), **GRPO** (group-relative normalized advantage, no critic, `grpo_train`).
  Math adversarially verified (8-agent design workflow) before code; all on rosnet primitives.
- `src/bench_m2.cyr` — parity control task + **rollouts-to-threshold** benchmark (median
  of 5 seeds): **PPO 32, GRPO 160, REINFORCE 192** — PPO/GRPO more sample-efficient (M2
  acceptance gate met).
- Demos: count task all three learn (REINFORCE 1.00→24.00, PPO 0.81→23.78, GRPO 0.81→24.00).
- **Next milestone:** M3 — reward / process-reward models.

## Tests

- `tests/tarka.tcyr` — **grad-check suite, 19/19 green** (M1: 4; M2: +15 — critic dW/db/dE
  FD-exact; GAE recursion==explicit + return identity; PPO ρ=1==REINFORCE / unclipped FD
  3e-9 / binding-clip→0; GRPO group-norm anchors + ratio=1==REINFORCE).
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
