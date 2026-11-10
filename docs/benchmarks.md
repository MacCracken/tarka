# tarka — Benchmarks

> Satisfies the v1.0 criterion *"Benchmarks captured in `docs/benchmarks.md`"*.
> Every number below is produced by the demo, which **is** the benchmark harness:
> `cyrius build src/main.cyr build/tarka && ./build/tarka`. Runs are deterministic
> (fixed tyche seeds + per-prompt reseed), so the numbers reproduce exactly.

## How to read this

tarka is a *reference* — the goal is that each RL/reasoning idea is demonstrably correct and
that the expected ordering between methods holds, not raw throughput. So the benchmarks are
**method-vs-method acceptance gates**, each falsifiable: if the better method did not actually
beat the baseline the gate would fail. Correctness underneath is gated separately by the
**50 finite-difference grad-checks** (`cyrius test`) — every hand-derived backward is
verified against central differences before any of these numbers are trusted — and by
`tests/golden.tcyr`, which holds every trainer's result to the bit
([ADR 0002](adr/0002-optimizations-preserve-bits.md)).

Measured on the dev host (cyrius 6.7.18, x86_64): full demo wall-clock **1.28 s** (1.4.0, with
the § 9 MCTS section; 706 ms at 1.3.0, 986 ms at 1.2.2, 3,495 ms at 1.2.1 — § 8). The numbers below are the same on aarch64 (qemu), and
1.3.0's reordered gradient sums (§ 8) left every one of them unchanged at the printed precision.

## 1. REINFORCE — the policy learns (count task)

Minimal rosnet-backed neural-bigram policy, on-policy REINFORCE, akshara-tokenized 45-byte
corpus (vocab 28), reward = count of the space token over a length-24 rollout.

| metric | before | after |
|--------|--------|-------|
| mean target-token count / 24 | 1.00 | **24.00** |

## 2. Actor-critic — PPO & GRPO learn, and are more sample-efficient

Same count task (all three learn), then the **acceptance** benchmark on a parity control task:
**rollouts to reach the reward threshold**, median of 5 seeds (lower is better).

| method | count task (before → after) | rollouts-to-threshold (parity, median/5) |
|--------|------------------------------|-------------------------------------------|
| REINFORCE | 1.00 → 24.00 | 192 |
| PPO (critic + GAE, clip 0.2, 4 epochs) | 0.81 → 23.78 | **32** |
| GRPO (group 8, no critic, clip 0.2) | 0.81 → 24.00 | 160 |

**Gate:** PPO and GRPO reach the threshold in fewer rollouts than REINFORCE → both are more
sample-efficient (PPO 6× fewer). ✅

## 3. Reward & process-reward models — learned rewards transmit; process > outcome

A Bradley-Terry reward model trained **only from preference orderings** (never the true
reward), then a fresh policy RL-trained on the *frozen learned* reward; we measure its TRUE
reward. ORM = outcome (whole-rollout prefs → GRPO); PRM = process (step-level prefs → PPO).

| reward model | held-out preference accuracy | policy TRUE reward / 16 |
|--------------|------------------------------|--------------------------|
| ORM (outcome → GRPO) | 100% | 13.64 |
| PRM (process → PPO)  | 100% | **15.54** |

**Gate:** both transmit (acc > 80%, true reward above the random floor) and process
supervision ≥ outcome at equal budget (Lightman 2023). ✅

## 4. Reasoning — deliberation beats single-sample (no extra training)

Inference-time deliberation on the trained policy + the frozen PRM verifier, parity-chain
answer task (random = 500‰).

**Self-consistency** (sample-and-vote), answer accuracy per-mille:

| votes N | 1 (single) | 16 | 64 |
|---------|-----------|----|----|
| accuracy ‰ | 627 | 797 | **930** |

**Verifier best-of-N** (N=16), true reward ×100 / 16:

| selection | mean sample | PRM-verifier-selected | oracle |
|-----------|-------------|------------------------|--------|
| true reward | 530 | **892** | 894 |

**Gate:** self-consistency scales with votes; the verifier reranks to near-oracle. ✅

## 5. Verifier-guided tree search — search is *necessary*

Generate a leading run of a `TARGET` token the policy never favors (never its argmax) — greedy
can't produce it, best-of-N needs ~V⁻ᵀ. PRM-guided beam search *steers* generation toward it.
All three measured at **matched compute** (total policy+PRM forward passes/prompt). Metric =
#TARGET of 16.

| method | #TARGET / 16 | fwd passes / prompt |
|--------|--------------|----------------------|
| greedy decode | 0.20 | 16 |
| best-of-N (sample + verifier-select) | 4.05 | 1152 |
| **PRM-guided beam** (B=8, K=8) | **13.50** | 1152 |

**Gate:** at matched compute, beam beats best-of-N **3.3×** and greedy **67×** — verifier-guided
search generates a verifier-preferred sequence neither greedy nor best-of-N can. ✅

## 6. Alignment — DPO, the KL penalty, IPO and KTO

Preference optimization against a frozen reference policy (the initial policy), on the count
task. DPO and IPO prefer the higher-target-count rollout of each sampled pair; KTO labels each
rollout desirable / undesirable against its batch's mean count.

| method | metric | before | after |
|--------|--------|--------|-------|
| DPO (preferences only, β 0.1) | target frequency / 24 | 0.94 | **24.00** |
| KL-to-reference penalty (60 steps) | mean KL to the reference | 3.33 | **2.46** |
| IPO (β 0.5, target margin 1.00) | mean implicit-reward margin h | 0.00 | **0.85** |
| KTO (β 0.1) | log-ratio ρ, desirable / undesirable held sequence | 0.00 / 0.00 | **80.31 / −138.27** |

**Gates:** DPO raises the target frequency ≥ 1.5× and the penalty lowers the KL ✅; IPO raises h
toward its target and KTO moves the two held sequences apart ✅ (this second gate counts toward
`ALL GATES PASS` since 1.2.2). 1.2.1 printed IPO's h as 0.59: `ipo_mean_h` scored tied draws,
which carry no preference, with an arbitrary winner.

## 7. Preference ingestion — `tarka --pref`

`tests/fixtures/prefs.jsonl`: 3 pairs, 4 thumbs (2 desirable, 2 undesirable), a bad label, a
non-JSON line and an unknown kind (`vocab=33  pairs=3  unary=4  skipped=3 (malformed 1, bad label
1, unknown kind 1)`).

| optimizer | before | after |
|-----------|--------|-------|
| DPO | loss 0.69, ranked 0/3 | loss 0.00, ranked **3/3** |
| IPO | loss 1.00, ranked 0/3 | loss 0.00, ranked **3/3** |
| KTO | ρ desirable 0.00 / undesirable 0.00 | **42.12 / −132.60** |

**Gates:** every pair ranked correctly with the loss down; each present KTO class moved in its
direction ✅. Wall-clock 64 ms. Since 1.3.0 each side of a pair is scored at its own length
(1.2.x truncated both to the shorter); the fixture's printed numbers are unchanged.

Options (1.3.0): `--epochs N` (default 300), `--lr X` (0.05), `--dim D` (8), `--early-stop`
(stop an optimizer once its gate holds, checked every 10 epochs). On the 2026-11-09 review's
103 KB file (256 pairs, 256 thumbs, V 92) the default run takes 14.4 s; with `--early-stop`
each of the three optimizers stops after 10 epochs and the run takes 1.2 s, all gates green.

## 8. Throughput

`scripts/bench-history.sh` runs `CYRIUS_DCE=1 cyrius bench tests/tarka.bcyr` three times and
appends the medians to `bench-history.csv`. The rows below were recorded on the dev host (AMD
Ryzen 7 5800H, x86_64) on 2026-11-09 — 1.2.1 and 1.2.2 in one session, 1.3.0 and 1.4.0 the same day:

| benchmark | 1.2.1 | 1.2.2 | 1.3.0 | 1.4.0 | |
|-----------|------:|------:|------:|------:|---|
| forward V28 D8 | 904 ns | 894 ns | 935 ns | 904 ns | (kernel, uncached) |
| forward V256 D64 | 11.57 µs | 12.83 µs | 13.05 µs | 12.26 µs | (kernel, uncached; run-to-run spread) |
| rollout T24 V28 D8 | 23.06 µs | 24.17 µs | 24.70 µs | 24.16 µs | (outside a memo region) |
| rl_backward T24 V28 D8 | 43.90 µs | 41.04 µs | 41.56 µs | 39.40 µs | (outside a region: per-token) |
| rl_backward T24 V256 D64 | 856.3 µs | 794.1 µs | 850.6 µs | 811.4 µs | |
| adam step V28 D8 | 8.77 µs | 9.02 µs | 9.32 µs | 8.88 µs | |
| adam step V256 D64 | 595.8 µs | 603.6 µs | 635.0 µs | 600.0 µs | |
| **REINFORCE step** (batch 16, T 24, V28 D8) | 1,072 µs | 368 µs | **156 µs** | 154 µs | 6.9× since 1.2.1 |
| **PPO step** (4 epochs) | 285.4 µs | 171.3 µs | **130.2 µs** | 124.0 µs | 2.2× |
| **GRPO step** (group 8) | 703.0 µs | 263.3 µs | **137.8 µs** | 128.5 µs | 5.1× |
| PRM step (batch 16, V12 D8) | 26.7 µs | 24.2 µs | 23.9 µs | 23.0 µs | (the RM backward is per-token) |
| **DPO step** (batch 16) | 2,692 µs | 649 µs | **332 µs** | 309 µs | 8.1× |
| **KTO step** (batch 16) | 2,486 µs | 515 µs | **308 µs** | 297 µs | 8.1× |
| beam search B8 K8 (per prompt) | 313.9 µs | 307.5 µs | 347.4 µs | 327.1 µs | (uncached, no policy backward; spread) |
| MCTS PUCT S512 (per prompt, corridor) | — | — | — | 13.69 ms | (1.4.0; § 9) |
| beam prior + value B4 K4 (per prompt, corridor) | — | — | — | 101.1 µs | (1.4.0; § 9) |
| pref_ingest_file, ~1 MB JSONL | 22.06 ms | 25.83 ms | 27.03 ms | 25.76 ms | the validating second pass (1.2.2) |

1.2.2's gain is the distribution memo, 1.3.0's the aggregated backward. End to end: the demo
3,495 → 986 → **706 ms** (median of three; 4.9× since 1.2.1); the review's 103 KB, V 92
preference file 219 → 41 → **14.4 s**; the demo under qemu-aarch64 36 → 10 → **6 s**. 1.4.0
changes no kernel (its column is run-to-run spread) and adds the § 9 section, which takes the
demo to ≈ 1.3 s (≈ 11 s under qemu).

## 9. Prior-guided MCTS — and the beam given the same networks

The corridor (`src/mcts.cyr`, [ADR 0003](adr/0003-prior-guided-mcts.md)): three zero-reward keys
K1 → K2 → K3 open a GOAL worth +4 per step; a TRAP pays +1 per step; out-of-order entries cost 60.
The evaluator is the exact reward. The prior is behavior-cloned from noisy demonstrations (30%
solver, 70% TRAP, 20% of actions randomized); the value is its critic (policy evaluation). True
return, mean of 40 prompts (prompt seed 2000; optimum 54.00, TRAP path 16.00), at the 0.7.0
accounting (forwards per prompt):

| method | prior seed 7 (the demo) | seed 11 | seed 13 |
|--------|-------------------------|---------|---------|
| policy greedy | 25.19 @16 | 25.19 @16 | 25.19 @16 |
| best-of-64 | 36.37 @2048 | 32.22 @2048 | 34.00 @2048 |
| beam, myopic (B 16, all children) | 24.19 @2844 | 24.19 @2844 | 24.19 @2844 |
| beam, prior (B 16, K 4 from π) | 27.89 @1118 | 27.75 @1128 | 27.89 @1105 |
| **MCTS, PUCT prior + value (S 512, c 1)** | **54.00 @1084** | **51.75 @1109** | **48.60 @1215** |
| **beam, prior + value (B 4, K 4)** | **53.42 @543** | **53.47 @545** | **53.12 @540** |
| beam, value (B 4, all children + V) | 13.19 @1464 | 54.00 @1464 | 54.00 @1464 |

**Gate:** MCTS reaches 85% of the optimum (45.90) and beats greedy, best-of-64 and the two
beams without lookahead at matched-or-higher compute for them ✅ — and the beam given the same
prior and value also reaches 85%, at no more than MCTS's compute ✅. Lookahead from the learned
value is what solves the corridor; on a deterministic task the tree adds nothing a value-ranked
beam lacks. In wall time the beam is cheaper still: 0.10 ms per prompt against MCTS's 13.7 ms
(the § 8 rows), because with cheap model calls PUCT's per-simulation bookkeeping dominates.
The ADR records the three formulations measured before this one (a learned verifier blurred the
corridor or was over-optimized; plain UCT could not expand it; on-policy PPO gave a prior
captured by the TRAP).

## Performance notes

- **The distribution memo** (1.2.2, [architecture note 001](architecture/001-distribution-memo.md)):
  the policy is a bigram, so inside a training step or an evaluation loop at most V distinct
  π(·|x) exist. tarka's loops compute each once per parameter version and copy it; so do the
  frozen reference (every DPO / IPO / KTO / KL forward) and the reward model. Bits unchanged —
  `tests/golden.tcyr` checks it on both targets. The kernel rows in § 8 do not move because
  they run outside a memo region.
- **The aggregated backward** (1.3.0, [architecture note 002](architecture/002-aggregated-backward.md)):
  inside a training step every token of one state shares its embedding row, so the policy's
  backward runs once per touched state on the sum of its seeds (≤ 28 `linear_bwd` per REINFORCE
  step instead of 384). It reorders the gradient's sums, so it shipped as a labeled behavior
  change (ADR 0002): 13 golden cases re-recorded, every gate and every printed demo number
  unchanged. The reward model's backward is still per token (a 1.x item).
- **Allocation discipline**: every working buffer (policy/critic/reward parameters + gradients
  + Adam moments, the memo tables, rollout/group/beam scratch) is allocated in an `*_init`
  function or once on first use (re-sized when the rollout length changes) and reused —
  there is **no per-step allocation churn** in any training, rollout, or search loop.
- **Adam**: a 0.9.0 loop-invariant hoist in the Adam inner loop (`(1−β)` precomputed once)
  leaves the numerics byte-identical; Adam is ~1% of a training step at the demo's shape.
- **Matched-compute methodology (§5)**: the budget unit is *total `linear_fwd` calls* (policy +
  PRM), counted via wrapper functions, so the beam-vs-best-of-N comparison is fair — best-of-N
  is given the same per-prompt forward budget as beam (1152), not merely the same sample count.
