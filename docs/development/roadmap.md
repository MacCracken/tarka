# tarka — Roadmap

> Milestone plan through v1.0. State lives in [`state.md`](state.md);
> this file is the sequencing — what ships, in what order, against
> what dependency gates.

## v1.0 criteria

- [ ] Public RL/reasoning API frozen — every exported symbol documented and tested
- [ ] Every hand-derived backward op finite-difference grad-checked (attn11 discipline)
- [ ] Benchmarks captured in `docs/benchmarks.md`
- [ ] At least one downstream consumer green
- [ ] CHANGELOG complete from v0.1.0 onward
- [ ] Security audit pass (`docs/audit/YYYY-MM-DD-audit.md`)

## Milestones

### M0 — Scaffold (v0.1.0) — ✅ shipped 2026-06-22

- `cyrius init` scaffold landed; identity + attn11 boundary documented.
- Deps wired: rosnet 0.2.0 (CPU) + tyche 0.1.1.
- [ADR 0001](../adr/0001-tarka-scope-and-rl-migration.md) — scope + migration boundary.

### M1 — REINFORCE migration (v0.2.0) — core shipped 2026-06-22

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
- **Remaining**: (a) wire `[deps.akshara]` so rollout prompts come from a real
  tokenized corpus; (b) **attn11 side** (separate, user-confirmed): remove
  `--objective rl` + RL surface; attn11 reverts to a pure SFT/diffusion reference.

### M2 — GRPO / PPO + value critic (v0.3.0)

The documented attn11 follow-on, now tarka's. Add a value head (critic), GAE
advantage estimation, and a clipped surrogate (PPO) / group-relative (GRPO) objective.

- **Acceptance**: critic + GAE + clipped objective grad-checked; a control task where
  PPO/GRPO measurably beats bare REINFORCE on sample efficiency.

### M3 — Reward & process-reward models (v0.4.0)

Learned reward models (outcome) and process-reward models (per-step), replacing the
hand-coded scalar reward — the substrate for verifier-guided reasoning.

### M4 — Verifier-guided reasoning (v0.5.0 → v1.0)

The "thinking" half: multi-step chain-of-thought rollouts, self-consistency
(sample-and-vote), then verifier-guided search → tree search / MCTS over reasoning
steps, scored by the M3 reward/process models.

## Out of scope (for v1.0)

- **The transformer itself** — that's attn11/rosnet. tarka consumes the policy
  network; it does not re-implement attention.
- **GPU execution** — CPU is the reference oracle through v1.0; a `--gpu` backend
  (rosnet-gpu + mabda) is a post-1.0 lever, mirroring attn11's M18 sequencing.
- **A production agent runtime** — tarka is the *reference*; the runtime is
  downstream (daimon-adjacent).
