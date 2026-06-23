# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- **M1 — REINFORCE on a minimal rosnet-backed policy (core).** The first real RL
  surface: a minimal autoregressive softmax policy — token embedding `E(V×D)` +
  linear head `W(D×V),b` (`π(next|cur) = softmax(E[cur]@W + b)`, a "neural
  bigram") — trained by on-policy **REINFORCE** (Williams 1992). Sample rollouts,
  reward = target-token count, weight the softmax-CE gradient by the advantage
  `(R − EMA baseline)`. Uses the same identity attn11's `--objective rl` did
  (`∇log π = −∇CE`), now re-expressed on **rosnet** `linear_fwd`/`linear_bwd`
  (matmul + gradient) with **tyche** sampling/init — no transformer; the loop is
  the point. Compact Adam (bias-corrected). `src/rl.cyr`.
- **Demo** (`src/main.cyr`): the policy learns to raise a target token's rollout
  frequency from random (**1.56**, ≈ T/V) to the max (**24.00 / 24**) — the
  demonstrable RL gate, mirroring attn11's X024.
- **Grad-check suite** (`tests/tarka.tcyr`): every hand-derived backward verified
  against central finite differences (attn11 discipline). Policy backward
  (dW/dE/db) FD-checked — maxrel ≤ 2e-9; the RL advantage-scaling identity
  (`grad(scale=A) == A·grad(scale=1)`) exact. **4/4 green.**

### Remaining in M1
- Wire **`[deps.akshara]`** — draw rollout prompts from a real tokenized corpus
  (replaces the synthetic `rng % V` vocab).
- De-feature attn11's `--objective rl` (its own cut — REINFORCE now lives here).

## [0.1.0] - 2026-06-22

**M0 — Scaffold.** tarka is established as the sovereign reasoning &
reinforcement-learning reference in Cyrius — the agency counterpoint to attn11.
attn11 owns representation / training mechanics (forward, backprop, Adam, SFT
objectives); tarka owns everything reward/verifier-driven and the deliberation it
trains. The two are sibling consumers of `rosnet` (tensors + gradient) and `tyche`
(PRNG), not a chain.

### Added
- Initial project scaffold (`cyrius init`).
- Project identity, goal, and the attn11 boundary in `CLAUDE.md` / `README.md`.
- `cyrius.cyml` deps wired: stdlib + `rosnet` 0.2.0 (CPU) + `tyche` 0.1.1.
- [ADR 0001](docs/adr/0001-tarka-scope-and-rl-migration.md) — tarka scope and the
  attn11 REINFORCE migration boundary (sibling-on-rosnet, not chained-on-attn11).
- Roadmap M1–M4 (REINFORCE migration → GRPO/PPO + critic → reward / process-reward
  models → verifier-guided reasoning).
