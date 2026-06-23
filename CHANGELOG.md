# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.4.0] - 2026-06-22

**M3 — learned reward & process-reward models (RLHF substrate).** The hand-coded
scalar reward is replaced by a reward model `r_θ(s,a)` learned from **preferences**
(Bradley-Terry), in outcome (ORM) and process (PRM) flavors — the substrate M4's
verifier-guided reasoning stands on. Math adversarially design-reviewed (workflow vs
Christiano/Ouyang/Lightman + tarka conventions) then grad-checked.

### Added — reward model (`src/m3.cyr`)
- **Bradley-Terry reward model** — `r_θ(s,a) = (E_r[s]·W_r + b_r)[a]`, a self-contained
  scalar head over its **own** params (separate embedding `E_r`, guaranteeing the demo
  is non-circular), **not** softmaxed. For a preference pair (winner ≻ loser),
  `Δ = R(W) − R(L)`, loss `−ln σ(Δ)`; the load-bearing gradient `dL/dΔ = σ(Δ) − 1`
  gives the per-step seed `g_r·onehot(a)` (single nonzero — distinct from the policy's
  `(probs−onehot)`), descent raising `r_θ` on the winner / lowering it on the loser.
- **Numerically-stable** log-sigmoid / softplus (two-branch, no overflow at large \|Δ\|);
  RM-own Adam (own bias-correction counters, never touches the policy's).
- **Process supervision** (`src/bench_m3.cyr`) — PRM trained from step-level parity
  preferences (parity-correct action ≻ wrong action), `ppo`-RL on the **frozen learned**
  per-step reward.

### Verified
- **+5 grad-checks (19 → 24, all green):** RM `dWr`/`dbr`/`dEr` Bradley-Terry FD-exact,
  plus the **descent-direction falsifier** (one BT step must increase Δ — catches a
  winner/loser sign swap the loss-FD cannot see), with finiteness guards.
- **Acceptance — non-circular signal transmission:** the RM learns the ordering from
  preferences alone (held-out preference accuracy **100%**, random 50%, true reward
  never seen); a **fresh** policy RL-trained against only the frozen learned reward
  reaches **15.52 / 16 true reward** (1.40 → 15.52). Lint clean, bench/fuzz compile.

## [0.3.0] - 2026-06-22

**M2 — GRPO / PPO + value critic.** The advantage-based RL family, the documented
attn11 follow-on, now tarka's: a value critic, GAE, and clipped-surrogate PPO / GRPO
on the minimal rosnet policy — all hand-derived, adversarially design-reviewed, and
finite-difference grad-checked. **Acceptance met:** on a parity control task with
state-dependent optimal actions, PPO and GRPO are measurably more sample-efficient
than bare REINFORCE — median **rollouts-to-threshold** (reward 8 / 16, 5 seeds):
**PPO 32, GRPO 160, REINFORCE 192** (PPO ~6× via per-step GAE credit + multi-epoch
reuse; GRPO via the group baseline).

### Added — M2 core: actor-critic, GAE, PPO, GRPO (`src/m2.cyr`)
- **Value critic** — scalar head `V(s) = w_v·E[s] + b_v` sharing the policy embedding
  (shared-trunk actor-critic), regressed by MSE to the GAE return; built on rosnet
  `linear_fwd`/`linear_bwd` with `N=1` (bit-exact scalar head).
- **GAE** (Schulman 2015) — per-step rewards + critic bootstrap → low-variance per-step
  advantages: `δ_t = r_t + γV(s_{t+1}) − V(s_t)` (V(s_T)=0), `A_t = δ_t + γλ·A_{t+1}`,
  return `Ĝ_t = A_t + V(s_t)`. γ=0.99, λ=0.95.
- **PPO** (Schulman 2017) — frozen `π_old` snapshot, importance ratio `ρ = exp(lnπ_θ −
  lnπ_old)`, clipped surrogate; per-step seed `X = ρ·A` in the trust region, **0** in the
  binding-clip half-space (`(A>0 ∧ ρ>1+ε) ∨ (A<0 ∧ ρ<1−ε)`), reducing **exactly** to M1's
  `A·(probs−onehot)` at ρ=1. Multi-epoch reuse. `ppo_train` (actor-critic, clip 0.2).
- **GRPO** (DeepSeek) — no critic; group-relative normalized advantage
  `Â_g = (R_g − mean)/(popstd + ε)` (std=0 → 0, no NaN), same clipped surrogate.
  `grpo_train` (group-relative).
- **Design** adversarially verified before implementation (8-agent workflow: derive +
  cross-check vs Schulman/DeepSeek and vs tarka's `(probs−onehot)·X` descent convention).

### Verified
- **+15 grad-checks (4 → 19, all green):** critic dW/db/dE (FD, exact); GAE recursion ==
  explicit `Σ(γλ)^l δ` + return identity (exact); PPO ρ=1 == advantage-REINFORCE (exact),
  unclipped surrogate FD (3e-9), binding-clip → zero gradient (exact); GRPO group-norm
  anchors ({2,1}→±1, {0,1,2}→±√1.5, all-equal→0) + ratio=1 == advantage-REINFORCE (exact).
  M1's 4 grad-checks unchanged.
- **Demo:** on the corpus task all three learn — REINFORCE 1.00→24.00, PPO (critic+GAE+clip)
  0.81→23.78, GRPO (group-8) 0.81→24.00. Build + lint + bench/fuzz clean.

### Added — sample-efficiency benchmark (`src/bench_m2.cyr`)
- **Parity control task** — reward depends on the state's parity (TGT on even states,
  DECOY on odd), so the optimal action is state-dependent and per-step credit assignment
  / sample reuse genuinely pay off. Single-step drivers per method (`par_*_step`) reuse
  the grad-checked backward; **rollouts-to-threshold**, median over 5 seeds.
- **Result:** PPO **32** / GRPO **160** / REINFORCE **192** rollouts to reach mean reward
  8 — PPO and GRPO both more sample-efficient (the M2 acceptance gate). `cyrius test`
  19/19, lint clean, bench/fuzz compile.

### Note
- M1 fully closed: attn11 de-featured `--objective rl` at **attn11 1.11.1** (RL lives here now).

## [0.2.1] - 2026-06-22

**akshara wired — tarka consumes the shared tokenizer.** Rollout prompts now come
from a real tokenized corpus instead of a synthetic vocab; "tarka consumes akshara"
(one tokenizer, two consumers with attn11) is real.

### Added
- **`[deps.akshara]` 0.1.0** (git+tag) — the sovereign tokenizer. tarka uses the
  pure path (`corpus_set` builds the byte vocab + packed store; `gd_ld` indexes it);
  the loader/streaming/emit consumer symbols stay unreached (stubbed for a clean build).
- **Corpus-grounded `rl_prompt`** (`src/rl.cyr`): when a corpus is loaded
  (`g_datalen > 0`) prompts are drawn from real tokens via `gd_ld` (attn11's
  rl_prompt); the synthetic-vocab fallback remains for the grad-check tests.
- **Demo** (`src/main.cyr`): tokenizes a 45-byte corpus (vocab 28), rewards the
  space token (id 3); REINFORCE drives its rollout frequency **1.00 → 24.00 / 24**.

### Unchanged
- Grad-checks **4/4 green** (akshara wired); warning-free build; bench/fuzz compile.

## [0.2.0] - 2026-06-22

**M1 core — REINFORCE comes home.** tarka's first real RL surface: a minimal
rosnet-backed policy trained by on-policy REINFORCE, the agency counterpoint to
attn11's supervised learner. No transformer yet — the loop is the point, proven
end-to-end with finite-difference grad-checks.

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
