# tarka — Roadmap

> Milestone plan through v1.0. State lives in [`state.md`](state.md);
> this file is the sequencing — what ships, in what order, against
> what dependency gates.

## v1.0 criteria

- [ ] Public RL/reasoning API frozen — every exported symbol documented and tested
- [x] Every hand-derived backward op finite-difference grad-checked (attn11 discipline) — **24/24**
- [x] Benchmarks captured in [`docs/benchmarks.md`](../benchmarks.md) (0.9.0)
- [ ] At least one downstream consumer green
- [ ] CHANGELOG complete from v0.1.0 onward
- [x] Security audit pass — [`docs/audit/2026-06-22-audit.md`](../audit/2026-06-22-audit.md) (0.8.0: 0 reachable bugs)

## Milestones

### Scaffold (v0.1.0) — ✅ shipped 2026-06-22

- `cyrius init` scaffold landed; identity + attn11 boundary documented.
- Deps wired: rosnet 0.2.0 (CPU) + tyche 0.1.1.
- [ADR 0001](../adr/0001-tarka-scope-and-rl-migration.md) — scope + migration boundary.

### REINFORCE migration (v0.2.0) — core shipped 2026-06-22

Bring attn11's `--objective rl` home and **re-express it on rosnet** (not a literal
copy of attn11's in-model seeding). On-policy REINFORCE: sample rollouts from the
current policy, score with a deterministic reward, weight the log-prob gradient by
the advantage `(R − b)` with an EMA baseline `b`.

- **Dep gate**: tokenizer **extracts to a small shared lib** (decided 2026-06-22 —
  "extraction for reuse"; tarka is the 2nd consumer, parallels rosnet/tyche). **Lib
  name locked: `akshara`** (अक्षर — indivisible text/sound unit). Carve happens in
  attn11's 1.x extract/re-fold window; tarka consumes `akshara`.
- **Acceptance**: the X024 gate reproduces here (target-token frequency rises decisively
  under RL); the REINFORCE backward is finite-difference grad-checked — the same
  discipline that gated it in attn11.
- **Progress (core ✅)**: `src/rl.cyr` — minimal rosnet-backed policy + REINFORCE +
  Adam landed. Demo: rollout target frequency **1.56 → 24.00 / 24** under REINFORCE.
  Grad-checks **4/4 green** (policy dW/dE/db FD-verified, maxrel ≤ 2e-9; RL
  advantage-scaling `grad(A) == A·grad(1)` exact). Uses a synthetic vocab for now.
- **akshara wired ✅ (0.2.1)**: `[deps.akshara]` 0.1.0; `rl_prompt` draws from a real
  akshara-tokenized corpus (`corpus_set` + `gd_ld`). Demo: 45 B → vocab 28, space
  token reward, freq 1.00 → 24.00. Grad-checks 4/4 green, warning-free build.
- **Remaining to close M1**: **attn11 side** (separate, user-confirmed): remove
  `--objective rl` + RL surface; attn11 reverts to a pure SFT/diffusion reference.

### GRPO / PPO + value critic (v0.3.0) — ✅ shipped 2026-06-22

The documented attn11 follow-on, now tarka's. Add a value head (critic), GAE
advantage estimation, and a clipped surrogate (PPO) / group-relative (GRPO) objective.

- **Acceptance**: critic + GAE + clipped objective grad-checked; a control task where
  PPO/GRPO measurably beats bare REINFORCE on sample efficiency.
- **Core ✅ (in-flight)**: `src/actorcritic.cyr` — critic + GAE + PPO + GRPO, math adversarially
  verified (8-agent design workflow vs Schulman/DeepSeek) then **grad-checked: +15 checks,
  19/19 green** (critic FD-exact; GAE recursion==explicit; PPO ρ=1==REINFORCE + clip→0;
  GRPO group-norm + ratio=1==REINFORCE). Demo: all three learn (PPO 0.81→23.78, GRPO →24.00).
- **Acceptance ✅**: parity control task + **rollouts-to-threshold** benchmark
  (`src/parity.cyr`, median of 5 seeds): **PPO 32, GRPO 160, REINFORCE 192** — PPO/GRPO
  measurably more sample-efficient. Cut at **0.3.0**.

### Reward & process-reward models (v0.4.0–0.4.1) — ✅ complete 2026-06-22

Learned reward models (outcome) and process-reward models (per-step), replacing the
hand-coded scalar reward — the substrate for verifier-guided reasoning.
- **Shipped**: `src/reward.cyr` — Bradley-Terry reward model `r_θ(s,a)` (own embedding,
  stable log-sigmoid), grad-checked (**24/24**, +5 incl. the BT descent-direction
  falsifier), design-reviewed vs Christiano/Ouyang/Lightman.
- **Acceptance (non-circular)**: RM held-out preference accuracy **100%** from orderings
  alone; a fresh policy RL'd on only the frozen learned reward reaches **15.52/16 true
  reward** (`src/preference.cyr`). Cut at **0.4.0**.
- **0.4.1 — completed with the outcome RM (ORM)** + process-vs-outcome comparison: ORM
  (whole-rollout prefs → GRPO) **13.64/16** vs PRM (step-level → PPO) **15.54/16** — both
  transmit; process supervision wins (Lightman 2023). Right RL per reward shape:
  per-step→PPO+GAE, terminal→GRPO (group-norm cancels the BT offset).

### Verifier-guided reasoning (v0.5.0–0.7.0) — ✅ complete 2026-06-22

The "thinking" half: multi-step chain-of-thought rollouts, self-consistency
(sample-and-vote), then verifier-guided search → tree search / MCTS over reasoning
steps, scored by the learned reward/process models.
- **0.5.0 (start) — outcome-level deliberation** (`src/reasoning.cyr`), adversarially
  design-reviewed (Wang/Cobbe/Gao): **self-consistency** answer accuracy 627→797→930
  (votes N=1→16→64); **verifier best-of-N** true reward 892 vs 530 mean, ~oracle 894
  (near-perfect PRM verifier). Reasoning task = parity-chain answer; policy made
  imperfect via partial training. Additive (24 grad-checks intact).
- **0.7.0 — tree search ✅** (`src/search.cyr`): a genuine-search task (generate a TARGET
  token the policy never favors → greedy ≈ 0, best-of-N ≈ V⁻ᵀ) + **PRM-guided beam search**
  that steers generation to it. Matched compute (1152 fwd/prompt, #TARGET of 16): **greedy
  0.20 / best-of-N 4.05 / beam 13.50** — beam 3.3× best-of-N, 67× greedy. Search is necessary.
  (MCTS scoped out: deterministic + one-good-action → beam is the right tool; post-1.0 lever.)

## Path to v1.0 (version plan)

The feature arc above (REINFORCE → actor-critic → reward models → reasoning) is built;
the remaining versions are about finish quality.

- **0.6.0 — ✅ refactor**: descriptive names replace the `Mx` milestone codes throughout
  the code (no behavior change; 24/24 grad-checks + all gates unchanged).
- **0.7.0 — ✅ tree search**: PRM-guided beam search on a genuine-search task (greedy 0.20 /
  best-of-N 4.05 / beam 13.50 of 16 at matched compute). The reasoning arc is closed.
- **0.8.0 — ✅ security/hardening audit**: 6-dimension find→verify workflow — 0 reachable bugs,
  fail-loud precondition guards (beam/GRPO buffer caps, count knobs) + pin 6.2.37.
  [`docs/audit/2026-06-22-audit.md`](../audit/2026-06-22-audit.md).
- **0.9.0 — ✅ optimization + documentation**: [`docs/benchmarks.md`](../benchmarks.md)
  (v1.0 criterion) + README/getting-started polish + Adam loop-invariant hoist (byte-identical
  numerics; the demo is rollout/matmul-bound, so no wall-clock change — kept for leanness).
- **1.0.0 — clean cut**: the v1.0 release. Remaining criteria: public API frozen + documented,
  ≥1 downstream consumer green, CHANGELOG complete.

## Out of scope (for v1.0)

- **The transformer itself** — that's attn11/rosnet. tarka consumes the policy
  network; it does not re-implement attention.
- **GPU execution** — CPU is the reference oracle through v1.0; a `--gpu` backend
  (rosnet-gpu + mabda) is a post-1.0 lever, mirroring attn11's M18 sequencing.
- **A production agent runtime** — tarka is the *reference*; the runtime is
  downstream (daimon-adjacent).
