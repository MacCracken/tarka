# tarka

> तर्क — *logic / reasoning / inference*. The sovereign **reasoning &
> reinforcement-learning** reference in [Cyrius](https://github.com/MacCracken/cyrius),
> the AGNOS "assembly-up" systems language where *everything is i64* and floating
> point is IEEE-754 `f64` bit-patterns driven by `f64_*` builtins.

tarka is the **agency counterpoint to [attn11](https://github.com/MacCracken/attn11)**.
attn11 proves that *gradient-based representation learning* is expressible from the
assembly up — forward pass, hand-derived backprop, Adam, all on raw `f64` arrays
with no BLAS, no libc, no autodiff. **tarka proves the same for *deliberate,
reward-driven problem-solving*** — the policy reasons in multiple steps, and
learning optimizes a *reward / verifier*, not next-token likelihood.

No RL framework. No autodiff. The policy gradient, the advantage estimator, the
value critic, and the search are all written by hand.

## What it owns

Everything past attn11's one basic REINFORCE objective — the full RL → reasoning
arc is **built** (the policy is a deliberately minimal rosnet-backed neural bigram;
the *methods* are the point):

- **RL training** — REINFORCE → PPO (value critic + GAE, clipped surrogate) & GRPO
  (group-relative) → learned **reward / process-reward models** (Bradley-Terry).
- **Reasoning** — inference-time deliberation: **self-consistency** (sample-and-vote),
  **verifier best-of-N**, and **PRM-guided beam search** that steers generation to a
  verifier objective greedy and best-of-N cannot reach.

Every hand-derived backward is finite-difference grad-checked (**24/24**), and each
method is held to a falsifiable **acceptance gate** — see
[`docs/benchmarks.md`](docs/benchmarks.md). A 0.8.0 security/hardening audit
([`docs/audit/`](docs/audit/2026-06-22-audit.md)) found **0 reachable bugs**.

## Results (headline)

| demonstration | result |
|---------------|--------|
| PPO / GRPO vs REINFORCE (rollouts-to-threshold) | **32 / 160** vs 192 — more sample-efficient |
| learned reward transmits (process vs outcome) | PRM **15.54** / ORM 13.64 of 16 true reward, 100% held-out |
| self-consistency (answer accuracy, votes 1→64) | 627 → **930** ‰ |
| verifier best-of-N (true reward, N=16) | **892** vs 530 mean ≈ oracle 894 |
| PRM-guided beam vs best-of-N vs greedy (matched compute) | **13.50** vs 4.05 vs 0.20 of 16 |

Full methodology + reproduction: [`docs/benchmarks.md`](docs/benchmarks.md).

## How it fits the stack

tarka is a **sibling consumer** of the substrate attn11 stands on, not a layer on
top of it:

| Need | Provider |
|------|----------|
| f64 tensors, matmul + gradient | [rosnet](https://github.com/MacCracken/rosnet) |
| deterministic PRNG (rollout sampling, init) | [tyche](https://github.com/MacCracken/tyche) |
| byte / BPE tokenizer | [akshara](https://github.com/MacCracken/akshara) (shared with attn11) |
| stats / epistemology | [pramana](https://github.com/MacCracken/pramana) |

The policy/value networks reassemble from rosnet primitives — the same way attn11's
transformer does — so tarka does **not** depend on attn11-the-binary. See
[ADR 0001](docs/adr/0001-tarka-scope-and-rl-migration.md) for the boundary.

## Build

```sh
cyrius deps                              # resolve stdlib + rosnet/tyche/akshara
cyrius build src/main.cyr build/tarka    # compile to a static ELF
./build/tarka                            # run the demo — every acceptance gate
cyrius test                              # 24/24 finite-difference grad-checks
```

The demo runs each milestone end-to-end and prints `ALL GATES PASS` (≈ 1.8 s).

## License

GPL-3.0-only
