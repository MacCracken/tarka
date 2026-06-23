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
**24/24 finite-difference grad-checks** (`cyrius test`) — every hand-derived backward is
verified against central differences before any of these numbers are trusted.

Measured on the dev host (cyrius 6.2.37): full demo wall-clock **≈ 1.8 s**.

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

## Performance notes

- **Allocation discipline**: every working buffer (policy/critic/reward parameters + gradients
  + Adam moments, rollout/group/beam scratch) is allocated once in an `*_init` function and
  reused across all steps — there is **no per-step allocation churn** in any training, rollout,
  or search loop.
- **Compute profile**: wall-clock is dominated by the rosnet `linear_fwd`/`linear_bwd` matmuls
  in rollouts and backward passes (and, in §5, the beam's `B·K·T` PRM forwards), not by the
  optimizer. A 0.9.0 loop-invariant hoist in the Adam inner loops (`(1−β)` precomputed once)
  leaves the numerics byte-identical and the wall-clock unchanged precisely because the demo is
  rollout/matmul-bound, not Adam-bound — recorded here for honesty, not as a speed claim.
- **Matched-compute methodology (§5)**: the budget unit is *total `linear_fwd` calls* (policy +
  PRM), counted via wrapper functions, so the beam-vs-best-of-N comparison is fair — best-of-N
  is given the same per-prompt forward budget as beam (1152), not merely the same sample count.
