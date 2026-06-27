# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [1.1.0] - Unreleased

**Alignment from preferences — DPO, IPO, KTO + the RLHF KL-to-reference-policy penalty.** Four
charter-owned preference-optimization mechanisms (tarka "owns ALL preference optimization + ALL RL"),
all hand-derived and finite-difference grad-checked, all **additive** to the frozen 1.x API.
Surfaced by the 2026-06-25 ifran/secureyeoman product-mining (the products carry DPO / RLHF /
preference losses only as Python wrappers — demand evidence; a grep confirmed none was built in
tarka). DPO + KL landed first; IPO and KTO complete the standard preference-loss set on the same
frozen-reference machinery.

### Added
- **DPO (Direct Preference Optimization)** — `src/dpo.cyr`. The implicit-reward
  reparameterization of the Bradley-Terry loss onto the **policy** against a FROZEN reference:
  `Δ = β·[(log π_θ − log π_ref)_w − (log π_θ − log π_ref)_l]`, `L = −ln σ(Δ)` (Rafailov 2023,
  β = 0.1). Reuses `reward.cyr`'s stable `bt_loss`/`sigma_stable` and `rl.cyr`'s softmax-CE seed
  — the per-step policy gradient is `scale·(probs − onehot)` with `scale = +β(1−σ(Δ))` on the
  winner, `−β(1−σ(Δ))` on the loser. No reward model, no value critic, no sampler-in-the-loop reward.
- **RLHF KL-to-reference-policy penalty** — `src/dpo.cyr`. `β·KL(π_θ ‖ π_ref)` with `q` frozen;
  hand-derived gradient `dKL/dlogit_k = p_k·(f_k − KL)`, `f_k = ln p_k − ln q_k` (Ouyang 2022 /
  TRL `init_kl_coef`). Distinct from PPO's `π_old` importance-ratio clip and from the frozen
  reward model — neither is a reference *policy*.
- **IPO (Identity Preference Optimization)** — `src/preference_ext.cyr`. The ΨPO identity-map loss
  (Azar et al. 2024): regress the **bare** implicit-reward margin `h = (log π_θ − log π_ref)_w −
  (…)_l` toward a FINITE target `1/(2β)` via `L = (h − 1/(2β))²`. Unlike DPO's `−ln σ`, the gradient
  does not vanish as the margin grows — it pulls `h` back if it overshoots. `dL/dh = 2(h − 1/(2β))`;
  winner seed `−2(h−m)`, loser `+2(h−m)` over the same `pol_accum_scaled` path (reuses `dpo_delta`
  at β=1 for `h`). Here β is a regularization strength (β=0.5 → target margin 1.0), distinct from
  DPO's reward-scale β.
- **KTO (Kahneman-Tversky Optimization)** — `src/preference_ext.cyr`. The UNPAIRED, prospect-theoretic
  loss (Ethayarajh et al. 2024): each example is labeled desirable/undesirable on its own against a
  **detached** KL reference point `z`. `L = λ·(1 − σ(u))`, `u = β(ρ − z)` desirable / `β(z − ρ)`
  undesirable, `ρ = log π_θ − log π_ref`. The gradient coefficient is the prospect-value magnitude
  `σ(u)(1−σ(u))` (NOT DPO's `1−σ`); descent raises ρ for desirable, lowers it for undesirable. `z`
  is a constant scalar threaded through forward + backward — its detachment is gated explicitly.
- **Frozen reference policy** — `dpo_snapshot()` copies the current policy params;
  `ref_logits`/`ref_softmax`/`ref_logp` evaluate under it.
- **Alignment demo + gates** — `src/main.cyr`: DPO from preferences alone raises target frequency
  **0.94 → 24.00 / 24**; the KL-to-reference penalty pulls mean KL **3.33 → 2.46**; IPO regresses the
  implicit-reward margin **h 0.00 → 0.59** toward its target 1.0; KTO raises desirable ρ **0.00 →
  80.31** and lowers undesirable ρ **0.00 → −138.27**.
- **Grad-checks** — `tests/tarka.tcyr`: DPO pairwise-loss FD grad-check (dW/dE/db; maxrel ≤ 9e-9)
  + a descent-direction falsifier, the KL-penalty FD grad-check (maxrel ≤ 4.0e-6) + a pull-to-reference
  falsifier, the IPO squared-margin FD grad-check + a `|h−m|`-decreases (pull-to-margin) falsifier,
  and the KTO FD grad-check on **both** labels + a two-sided **z-detachment** falsifier (the z-detached
  FD matches the analytic grad at ~14e-9 while the z-*live* FD diverges to ~9.7) + a direction
  falsifier (raises desirable ρ, lowers undesirable ρ). Suite **24/24 → 34/34 → 50/50**.

Additive only — the v1.0 public surface is unchanged; all prior gates byte-identical, all prior
grad-checks intact. Toolchain pin: cyrius 6.2.37.

## [1.0.0] - 2026-06-22

**v1.0 — the RL → reasoning reference is complete and frozen.** From the assembly up, with no
RL framework, no autodiff, no BLAS: REINFORCE → actor-critic (PPO/GRPO + value critic + GAE) →
learned reward / process-reward models → verifier-guided reasoning (self-consistency,
best-of-N, PRM-guided beam search). Every hand-derived backward is finite-difference
grad-checked; every method clears a falsifiable acceptance gate. tarka is the **agency
counterpoint to attn11**: attn11 proves gradient-based *representation learning* is expressible
in everything-is-i64 Cyrius; tarka proves the same for *deliberate, reward-driven
problem-solving*.

### Added
- **Public API freeze** — [`docs/api.md`](docs/api.md): the stable 1.x surface (policy +
  REINFORCE, PPO/GRPO, reward/process-reward models, reasoning, search), every symbol
  documented with where it is tested; the internal mechanism is explicitly out of the freeze.
- **Downstream consumer** — [`examples/quickstart.cyr`](examples/quickstart.cyr): a standalone
  program that consumes the public API (REINFORCE in ~20 lines), builds and runs green
  (reward 1.58 → 23.94).

### v1.0 criteria — all met
- [x] Public RL/reasoning API frozen — documented (`docs/api.md`) and tested.
- [x] Every hand-derived backward finite-difference grad-checked — **24/24** (`cyrius test`).
- [x] Benchmarks captured — [`docs/benchmarks.md`](docs/benchmarks.md).
- [x] At least one downstream consumer green — `examples/quickstart.cyr`.
- [x] CHANGELOG complete from v0.1.0 onward.
- [x] Security audit pass — [`docs/audit/2026-06-22-audit.md`](docs/audit/2026-06-22-audit.md)
  (0 reachable bugs; fail-loud precondition guards).

No code change from 0.9.0 — 1.0.0 is the clean cut: the freeze + the consumer that closes the
last criteria. Toolchain pin: cyrius 6.2.37.

## [0.9.0] - 2026-06-22

**Optimization + documentation.** Captures the benchmarks (a v1.0 criterion), polishes the
docs to reflect the now-built RL→reasoning arc, and lands a clarity-improving optimizer hoist.
No behavior change — 24/24 grad-checks + all gates byte-identical.

### Added
- **`docs/benchmarks.md`** — every demo number in one place (REINFORCE; PPO/GRPO
  sample-efficiency; ORM/PRM reward transmission; self-consistency; verifier best-of-N;
  PRM-guided beam search at matched compute) with methodology + exact-reproduction notes.
  Satisfies the v1.0 *"benchmarks captured"* criterion.
- **Module map** in `docs/guides/getting-started.md` — what each `src/*.cyr` contains, in
  include/dependency order.

### Changed
- **`README.md`** — reflects the built arc (REINFORCE → actor-critic → reward models →
  reasoning incl. beam search), a headline-results table, the audit/benchmarks links, and the
  akshara tokenizer (was the stale "attn11 checkpoint format" row).
- **Optimization — loop-invariant hoist in the Adam inner loops** (`adam_one`, `rm_adam_one`):
  `(1−β1)`/`(1−β2)` are precomputed once instead of every element. Numerics byte-identical
  (24/24 grad-checks). No wall-clock change on the demo (≈1.8 s) — it is rollout/matmul-bound,
  not Adam-bound; kept for code leanness/clarity, **not** claimed as a speedup.

### Notes
- Verified the allocation discipline: every working buffer is allocated once in an `*_init`
  function and reused — **no per-step allocation churn** in any training/rollout/search loop.
  Documented in `benchmarks.md` § Performance notes.

## [0.8.0] - 2026-06-22

**Security / hardening audit (the v1.0 audit gate) + toolchain bump.** A 6-dimension audit
workflow (find → adversarially verify reachability) found **zero reachable bugs** — tarka is
memory-safe as called today — and a set of unchecked *preconditions* on public functions that
a future caller could trip into silent heap corruption (Cyrius has no bounds checks). Those
are now guarded fail-loud. Additive — the 24 grad-checks and all demo gates are byte-identical.
Full report: [`docs/audit/2026-06-22-audit.md`](docs/audit/2026-06-22-audit.md).

### Changed
- **Toolchain pin** `cyrius.cyml [package].cyrius` 6.2.36 → **6.2.37** (matches the installed
  toolchain; build clean, no drift warning, 24/24 grad-checks on the new pin).

### Added — hardening guards (`guard()` in `rl.cyr`: fail-loud, prints + `SYS_EXIT`)
- `srch_beam` validates `1 ≤ B ≤ BMAX` and `1 ≤ K, B*K ≤ BMAX²` before indexing the fixed
  beam/candidate buffers (prevents heap overrun + the `K==0` `-1`-sentinel underflow).
- GRPO group size validated `1 ≤ G ≤ GMAX` in `grpo_train` / `par_grpo_step` /
  `rm_orm_grpo_step` (prevents `GR_*` overrun + `G==0` divide-by-zero).
- `par_wrong` / `target_wrong` require `g_V ≥ 2` (the reject-until-different loop would
  otherwise not terminate).
- `reason_accuracy` / `reason_best_of_n` / `reason_self_consistency` / `srch_*` require their
  count args `≥ 1` (`reason_accuracy`'s integer `/nprompts` would otherwise trap).

### Notes
- `assert` (stdlib) only *prints* on failure and does not stop, so it cannot prevent a
  subsequent corrupting write — hence the dedicated fail-loud `guard()`.
- Residual low-risk items (NaN on a `0` count knob in non-search entries; `f64_ln` underflow)
  are documented as accepted in the audit: they yield NaN, not memory unsafety, and are
  unreachable from the reference's own usage.

## [0.7.0] - 2026-06-22

**Reasoning complete — PRM-guided tree (beam) search, where search is genuinely
necessary.** 0.5.0 showed self-consistency + best-of-N, but parity is greedy-optimal so
search couldn't beat greedy there. 0.7.0 adds a task that *requires* search and the tree
search to solve it — closing the verifier-guided-reasoning arc. Design adversarially
reviewed (workflow vs Tree-of-Thoughts / MCTS-PUCT / process-reward search + budget
fairness). Additive (`src/search.cyr`) — the 24 grad-checks stay byte-identical (zero new
backward; pure inference-time search over the frozen policy + frozen PRM verifier).

### Added (`src/search.cyr`)
- **A genuine-search task**: generate a leading run of a `TARGET` token the policy never
  prefers (never its argmax, ~1/V per step) — so greedy ≈ 0 and best-of-N needs ~V⁻ᵀ
  (hopeless), but a PRM trained to prefer TARGET can guide search to it.
- **PRM-guided beam search** (`srch_beam`, width B, branch K): expand each beam by K
  sampled actions, score each child by parent-cumulative-PRM + the new edge (incremental,
  1 PRM forward/child → linear cost), keep top-B. Pruning concentrates the budget on
  verifier-preferred prefixes — *steering* generation, not just filtering whole samples.
- **Matched-compute harness** — a forward-pass counter (wrapper functions, not edits to
  `pol_logits`/`rm_reward`); greedy / best-of-N / beam all measured in total `linear_fwd`.

### Result (at MATCHED compute, 1152 forward passes/prompt; metric = #TARGET of 16)
- **greedy 0.20 · best-of-N 4.05 · PRM-guided beam 13.50** — beam beats best-of-N **3.3×**
  and greedy **67×**. Verifier-guided tree search is genuinely necessary: it generates a
  verifier-preferred sequence that neither greedy decode nor best-of-N sampling can.
- (MCTS/PUCT was scoped out per the design review — deterministic transitions + one good
  action per step make beam the right tool here; MCTS is a post-1.0 lever.)

## [0.6.0] - 2026-06-22

**Refactor — descriptive names replace the `Mx` milestone codes (no behavior change).**
The internal `M1/M2/M3/M4` milestone labels were never real identifiers; every file,
function, and constant now carries a meaningful name. Byte-for-byte behavior preserved —
24/24 grad-checks unchanged, all demo gates pass.

### Changed
- **Files** renamed by what they contain:
  `m2.cyr → actorcritic.cyr` (value critic, GAE, PPO, GRPO),
  `m3.cyr → reward.cyr` (Bradley-Terry reward model),
  `bench_m2.cyr → parity.cyr` (the parity task + per-method RL steps + sample-efficiency
  benchmark), `bench_m3.cyr → preference.cyr` (preference-trained reward + ORM/PRM demo),
  `bench_m4.cyr → reasoning.cyr` (verifier-guided reasoning).
- **Identifiers**: `m2_init → ac_init`, `m2_scale_grads → ac_scale_grads`,
  `m2_bench_run → eff_run`; `m4_setup → reason_setup`, `m4_answer → reason_answer`,
  `m4_oracle → reason_oracle`, `m4_sc_answer → reason_self_consistency`,
  `m4_bon_reward → reason_best_of_n`, `m4_method_acc → reason_accuracy`,
  `m4_corr_gate → reason_verifier_gate`; the `M4_*` task constants → `REASON_*`; the demo
  gate flags `m{2,3,4}_ok → {ac,reward,reason}_ok`. (`rl_*`, `rm_*`, `gae_*`, `ppo_*`,
  `grpo_*`, `critic_*`, `par_*` were already descriptive — untouched.)
- **Comments + demo output**: the `M1/M2/M3/M4` labels read REINFORCE / actor-critic /
  reward-model / reasoning. Versioned-release history in this CHANGELOG is left as-is.

## [0.5.0] - 2026-06-22

**M4 (start) — verifier-guided reasoning, the "thinking" half.** The trained policy + the
M3 learned reward model now produce better answers by **deliberating at inference**, no
extra training: self-consistency scales accuracy **627 → 930** with votes, and a learned
verifier reranks samples to **near-oracle** quality. This opens the v0.5.0 → v1.0 arc
(PRM-guided tree search / MCTS on a sequential task is the remaining piece).

### Added — M4 (start): verifier-guided reasoning at inference (`src/bench_m4.cyr`)
The "thinking" half — the trained policy + the M3 reward model (verifier) now produce
better answers via **deliberation, not more training**. Design adversarially reviewed
(workflow vs Wang 2022 / Cobbe 2021 / Gao 2022 + tarka-fit + measurement fairness). All
in a new file — rl.cyr/m2.cyr/m3.cyr untouched (24 grad-checks byte-identical).
- **Reasoning task** — a checkable answer over a multi-step parity chain (answer =
  majority parity-class of the chain's states; oracle = `par_correct(prompt)`). The
  policy is made imperfect-but-informative by **partial training** (single dial, no
  temperature), giving single-sample accuracy **627/1000** (headroom for deliberation).
- **Self-consistency** (Wang 2022, sample-and-vote, no verifier) — answer accuracy
  **627 → 797 → 930** at vote-budget N = 1 → 16 → 64 (variance reduction; monotone).
- **Verifier best-of-N** (Cobbe 2021) — the PRM reranks N samples by `rm_score`; the
  selected rollout's TRUE reward **892** (×100 of 16) vs **530** mean sample, **matching
  the oracle 894** — the learned verifier is near-perfect (hence no overoptimization dip,
  as the design review predicted). Fair measurement: fixed prompt set + per-prompt reseed
  (nested samples across N); strict verifier/oracle separation.
- **Honest scope**: parity has per-step-independent reward + deterministic transitions, so
  it demonstrates *outcome-level* deliberation (SC + best-of-N) but NOT search — PRM-guided
  **tree search / MCTS** needs a *sequential* task (the next M4 sub-milestone, toward v1.0).

## [0.4.1] - 2026-06-22

**M3 completion — outcome reward model + process-vs-outcome supervision.** 0.4.0 shipped
the reward-model core + the process (PRM) path; this adds the **outcome reward model
(ORM)** — the other half of "reward & process-reward models" — and the head-to-head
comparison (Lightman 2023: process supervision beats outcome supervision).

### Added (`src/bench_m3.cyr`)
- **ORM** — `r_θ` trained from **whole-rollout** preferences (winner ≻ loser by true
  reward), consumed by its natural RL: **GRPO** on the terminal reward `R(τ)=Σ r_θ`.
  GRPO's group-relative (R−mean)/std normalization cancels the Bradley-Terry reward's
  unbounded additive offset — the stability the sparse-terminal PPO+GAE path lacked
  (which the diagnostic showed failing to 0.0). Held-out rollout-pair ranking accuracy.
- **Process-vs-outcome comparison** in the demo: both learned rewards are trained only
  from preference orderings, then a fresh policy is RL'd on each frozen reward and
  scored on TRUE reward — **ORM (outcome→GRPO) 13.64/16, PRM (process→PPO) 15.54/16**
  (both 100% held-out preference accuracy). Process supervision wins at equal budget.

### Note
- Right RL for the reward shape: per-step (PRM) → PPO+GAE; terminal (ORM) → GRPO. No new
  gradient math (the Bradley-Terry backward is 0.4.0's, grad-checked); 24/24 still green,
  lint clean, bench/fuzz compile.

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
