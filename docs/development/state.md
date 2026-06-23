# tarka — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**1.0.0** — 2026-06-22. **v1.0 — clean cut; the RL→reasoning reference is complete + frozen.**
Public API frozen + documented (`docs/api.md`: stable 1.x surface, internal mechanism
out-of-freeze) and a downstream consumer green (`examples/quickstart.cyr` — REINFORCE via the
public API, 1.58→23.94). All six v1.0 criteria met: ✅ API frozen+documented · ✅ 24/24
grad-checks · ✅ benchmarks · ✅ downstream consumer · ✅ CHANGELOG complete · ✅ security audit.
No code change from 0.9.0 — the freeze + consumer close the last criteria. The full arc
(REINFORCE → actor-critic → reward models → verifier-guided reasoning incl. beam search) is
the agency counterpoint to attn11. Pin 6.2.37.

**0.9.0** — 2026-06-22. **Optimization + documentation.** `docs/benchmarks.md` (v1.0
criterion) captures every demo number with methodology + reproduction; README reflects the
built arc + a headline-results table + akshara; getting-started gains a module map.
Optimization: loop-invariant hoist in the Adam inner loops (`(1−β)` precomputed once) —
byte-identical numerics, no wall-clock change (demo ≈1.8 s is rollout/matmul-bound, not
Adam-bound; kept for leanness). Verified allocation-clean (all buffers `*_init`-allocated +
reused, no per-step churn). 24 grad-checks + all gates byte-identical.

**0.8.0** — 2026-06-22. **Security / hardening audit (v1.0 audit gate) + toolchain bump.**
6-dimension audit workflow (find → adversarially verify reachability): **0 reachable bugs**
(tarka is memory-safe as called today), and unchecked public-API *preconditions* a future
caller could trip into silent heap corruption — now guarded **fail-loud** (`guard()` in
`rl.cyr`: prints + `SYS_EXIT`). Guards: beam `B≤BMAX`/`B*K≤BMAX²`, GRPO `G≤GMAX`, rejection
loops `g_V≥2`, count knobs `≥1`. Toolchain pin **6.2.36→6.2.37**. Additive — 24 grad-checks +
all gates byte-identical. Report: `docs/audit/2026-06-22-audit.md`.

**0.7.0** — 2026-06-22. **Reasoning complete — PRM-guided tree (beam) search where search
is genuinely necessary.** `src/search.cyr` adds a task that *requires* search (generate a
leading run of a `TARGET` token the policy never prefers) + PRM-guided beam search to solve
it. At MATCHED compute (1152 fwd/prompt; metric = #TARGET of 16): **greedy 0.20 · best-of-N
4.05 · beam 13.50** — beam beats best-of-N 3.3× and greedy 67×. Verifier-guided search
generates a verifier-preferred sequence neither greedy nor best-of-N can. Design adversarially
reviewed (ToT/MCTS/budget-fairness). Additive — 24 grad-checks byte-identical. (MCTS scoped
out: deterministic + one-good-action → beam is the right tool; post-1.0 lever.) **The
verifier-guided-reasoning arc is closed.**

**0.6.0** — 2026-06-22. **Refactor — descriptive names replace the `Mx` milestone codes**
(no behavior change; 24/24 grad-checks + all gates unchanged). Files renamed by content
(`actorcritic.cyr`, `reward.cyr`, `parity.cyr`, `preference.cyr`, `reasoning.cyr`); all
`m2_*`/`m4_*`/`M4_*` identifiers → `ac_*`/`reason_*`/`REASON_*`/`eff_run`; comments + demo
output read REINFORCE / actor-critic / reward-model / reasoning. (`rl_*`/`rm_*`/`gae_*`/
`ppo_*`/`grpo_*`/`par_*` were already descriptive.)

**0.5.0** — 2026-06-22. **Reasoning (start) — verifier-guided deliberation.** Inference-time
deliberation on the trained policy + reward-model verifier (no extra training): **self-consistency**
(answer accuracy **627 → 930** over votes N=1→64) and **verifier best-of-N** (PRM reranks
to true reward **892** vs 530 mean, ~oracle **894**). Design adversarially reviewed
(Wang/Cobbe/Gao). Additive — 24 grad-checks byte-identical. Opens the v0.5.0→v1.0 arc
(PRM tree search / MCTS on a sequential task remains). (0.4.x M3; 0.3.0 M2; 0.2.x M1;
0.1.0 scaffold.) **M1 closed** at attn11 1.11.1.

**0.4.1** — 2026-06-22. **M3 complete — learned reward & process-reward models.** A
Bradley-Terry reward model `r_θ(s,a)` from **preferences** (own embedding, not softmaxed,
stable log-sigmoid), in both **ORM** (outcome, whole-rollout prefs → GRPO) and **PRM**
(process, step-level prefs → PPO) flavors; design-reviewed (vs Christiano/Ouyang/Lightman)
+ grad-checked (**24/24**). **Non-circular acceptance + process-vs-outcome:** both RMs hit
**100%** held-out preference accuracy from orderings alone; a fresh policy RL'd on each
frozen learned reward reaches **PRM 15.54 / ORM 13.64 (of 16)** true reward — process
supervision wins (Lightman). (0.4.0 = M3 core+PRM; 0.3.0 M2; 0.2.x M1+akshara; 0.1.0
scaffold.) **M1 closed** at attn11 1.11.1.

## Toolchain

- **Cyrius pin**: `6.2.37` (in `cyrius.cyml [package].cyrius`) — bumped from 6.2.36 at
  0.8.0; matches the installed toolchain (no drift warning); same 6.2.x band attn11/rosnet consume.

## Source

**REINFORCE core — shipped at 0.2.0; akshara wired at 0.2.1:**
- `src/rl.cyr` — the minimal rosnet-backed policy (embedding + linear head;
  `π(next|cur) = softmax(E[cur]@W + b)`) + on-policy REINFORCE (rollout → reward →
  advantage `(R − EMA baseline)` → advantage-weighted softmax-CE backward) + compact
  bias-corrected Adam. Dogfoods rosnet `linear_fwd`/`linear_bwd` + tyche sampling.
  `rl_prompt` draws from a real akshara-tokenized corpus when one is loaded.
- `src/main.cyr` — demo: tokenizes a corpus via akshara (`corpus_set`), rewards the
  space token; rollout frequency rises **1.00 → 24.00 / 24** under REINFORCE.

**M1 closed:** attn11 de-featured `--objective rl` at **attn11 1.11.1** (RL lives here).

**Actor-critic / PPO / GRPO — shipped at 0.3.0:**
- `src/actorcritic.cyr` — value critic `V(s)=w_v·E[s]+b_v` (shared-E, MSE-regressed), **GAE**
  (γ=0.99 λ=0.95), **PPO** clipped surrogate (frozen π_old, ratio, clip 0.2, multi-epoch
  via `ppo_train`), **GRPO** (group-relative normalized advantage, no critic, `grpo_train`).
  Math adversarially verified (8-agent design workflow) before code; all on rosnet primitives.
- `src/parity.cyr` — parity control task + **rollouts-to-threshold** benchmark (median
  of 5 seeds): **PPO 32, GRPO 160, REINFORCE 192** — PPO/GRPO more sample-efficient (M2
  acceptance gate met).
- Demos: count task all three learn (REINFORCE 1.00→24.00, PPO 0.81→23.78, GRPO 0.81→24.00).

**Learned reward / process-reward models — complete (core+PRM 0.4.0, ORM 0.4.1):**
- `src/reward.cyr` — Bradley-Terry reward model `r_θ(s,a)` (own embedding, not softmaxed,
  stable log-sigmoid, RM-own Adam; seed `g_r·onehot(a)`).
- `src/preference.cyr` — **PRM** (step-level prefs → PPO, dense per-step reward) **and ORM**
  (whole-rollout prefs → GRPO, terminal reward; GRPO normalization cancels the BT offset
  the sparse-PPO path couldn't handle).
- Acceptance (non-circular + process-vs-outcome): both RMs **100%** held-out preference
  accuracy from orderings only; fresh policy on each frozen reward reaches **PRM 15.54 /
  ORM 13.64 (of 16)** true reward — process supervision wins.

**Verifier-guided reasoning — complete (deliberation 0.5.0, tree search 0.7.0):**
- `src/reasoning.cyr` — inference-time deliberation on the trained policy + reward-model verifier
  (no extra training). Reasoning task = checkable answer over a parity chain; policy made
  imperfect via partial training (single-sample acc 627/1000). **Self-consistency** scales
  627→797→**930** (N=1→16→64); **verifier best-of-N** selects true reward **892** vs 530
  mean, ~matching oracle **894** (near-perfect PRM verifier). Design adversarially reviewed
  (Wang/Cobbe/Gao). Additive — 24 grad-checks byte-identical.
- **Tree search ✅ (0.7.0)**: `src/search.cyr` — PRM-guided **beam search** on a genuine-search
  task (generate a TARGET token the policy never favors). Matched compute: greedy 0.20 /
  best-of-N 4.05 / **beam 13.50** of 16 — beam 3.3× best-of-N, 67× greedy. Search is necessary.

## Tests

- `tests/tarka.tcyr` — **grad-check suite, 24/24 green** (M1: 4; M2: +15 — critic dW/db/dE
  FD-exact; GAE recursion==explicit + return identity; PPO ρ=1==REINFORCE / unclipped FD
  3e-9 / binding-clip→0; GRPO group-norm anchors + ratio=1==REINFORCE; M3: +5 — RM
  dWr/dbr/dEr Bradley-Terry FD-exact + descent-direction falsifier).
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

## Next (version plan)

- **0.7.0 — ✅ done**: PRM-guided beam search (the reasoning arc is closed).
- **0.8.0 — ✅ done**: security/hardening audit (0 reachable bugs; precondition guards) + pin 6.2.37.
- **0.9.0 — ✅ done**: benchmarks.md (v1.0 criterion) + doc polish + Adam loop-invariant hoist.
- **1.0.0 — ✅ SHIPPED**: clean cut — API freeze (`docs/api.md`) + downstream consumer
  (`examples/quickstart.cyr`); all six v1.0 criteria met. **tarka is v1.0.**

## Post-1.0

The reference is complete. Natural next levers (post-1.0, user-driven): a `--gpu` backend
(rosnet-gpu + mabda), MCTS on a task that warrants it, or extracting reusable primitives to a
shared lib (the rosnet/tyche/akshara pattern) once a second consumer appears.

See [`roadmap.md`](roadmap.md).
