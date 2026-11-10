# tarka — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**1.4.0** — 2026-11-09. **The 1.4.x line: MCTS on a task that warrants it, gated honestly**
([ADR 0003](../adr/0003-prior-guided-mcts.md)). `src/mcts.cyr`: the corridor (three zero-reward
keys before a delayed payoff, a TRAP, a 60-point out-of-order penalty), a behavior-cloned prior,
a policy-evaluation critic, PUCT with subtree reuse, and greedy / best-of-N / four beams given the
same networks. MCTS S512 **54.00 @1,084** forwards per prompt (optimum 54.00) against best-of-64
36.37 and the myopic beam 24.19 — and a prior + value beam **53.42 @543**; the demo prints both
and gates both. Additive: no signature changed, no golden case moved. Demo 706 ms → ≈ 1.3 s (the
new section). Assertions 234 → **266**, abort probes 18, benchmarks 17, 36-mutant run all killed.

**1.3.0** — 2026-11-09. **The 1.3.x line: training throughput and preference fidelity.** The
policy's backward is aggregated per state inside tarka's training loops (one `linear_bwd` per
touched state; [architecture note 002](../architecture/002-aggregated-backward.md)) — a labeled
behavior change: 13 golden cases re-recorded, every gate and every printed demo number unchanged.
Two-length DPO / IPO primitives (`dpo_*2`, `ipo_*2`, FD-checked with unequal sides); `--pref`
scores each side at its own length. `--pref` options `--epochs` / `--lr` / `--dim` /
`--early-stop` (defaults byte-identical). `guard()` aborts through the stdlib's `panic`. Demo
986 → **706 ms**, REINFORCE / GRPO / DPO / KTO steps 2.4× / 1.9× / 2.0× / 1.7× over 1.2.2.
Assertions 190 → **234**, abort probes 14, 28-mutant run all killed.

**1.2.2** — 2026-11-09. **P(-1) sweep: audit, refactor, hardening, optimization, repairs**
([`docs/audit/2026-11-09-audit.md`](../audit/2026-11-09-audit.md)). Toolchain and deps unchanged
(6.7.18; tyche 1.2.2 / rosnet 1.4.1 / akshara 1.1.2). **One HIGH fixed**: `--pref` built its vocab
over the raw file, so an escape-decoded byte (`\t`, `\r`, a lone-line `\n`) had id −1 and indexed
the policy's tensors at −1 (a write into `P_b`); the vocab now covers decoded bytes and a byte
without an id refuses the row. Seven MEDIUM (`--pref`: over-cap truncation, lenient labels,
unchecked field types, wrong `\u`/`\b`/`\f` decoding; the IPO/KTO gate not counted;
`ipo_mean_h` scoring ties — h 0.59 → 0.85; CI ran no gate) and ten LOW, all fixed. Refactors (one
seed / softmax / Adam kernel, `ppo_update`, `grpo_collect`/`grpo_update`) and a per-state
distribution memo keep every bit (golden gate, x86_64 + aarch64): demo **3,495 → 986 ms**, DPO /
KTO steps 4.2× / 4.8×. Tests 73 → **190** assertions in four suites (+ 13 abort probes, a 4,000-round
fuzz harness, 15 benchmarks); 17-mutant run, all killed. CI now runs the demo, `--pref`, the
probes, bench, fuzz, fmt/lint/vet/doc gates and an aarch64 job.

**1.2.1** — 2026-11-09. **Realign to the latest releases, plus the deprecated-getter migration.** Pin `6.7.9` → `6.7.18`;
tyche 1.2.0 → **1.2.2**, rosnet 1.2.0 → **1.4.1**, akshara 1.1.0 → **1.1.2** (each bit-identical on valid
input; akshara still pins rosnet 1.2.2, and the two rosnet functions it calls, `t_alloc` and `t_zero`,
are byte-identical in 1.4.1). `lib/` re-synced to the 6.7.18 snapshot; `cyrius.lock` regenerated,
`deps --verify` 117/117 and `deps --locked` clean. Suite **73/73** unchanged; the demo and test output
are byte-identical to 1.2.0. The binary grows 1.9× (1,253,928 B), almost all of it the Unicode data
tables bayan 1.11.1 brings in. `src/pref_ingest.cyr`'s three calls to the JSON getters bayan 1.11.0
deprecates now use the `_by_str` forms they forwarded to. The build is warning-free, and `--pref`
output is byte-identical.

**1.2.0** — 2026-11-09. **Onto the cyrius 6.7 series, no source change.** Pin `6.6.20` → `6.7.9`;
tyche 1.1.1 → **1.2.0**, rosnet 1.1.3 → **1.2.0**, akshara 1.0.4 → **1.1.0** (the 6.7.9-built set,
one consistent chain, none with a source or value change). tarka's source was checked against 6.7's
language changes (new reserved words, a real `!`, tighter unary `-`/`~`, typed-pointer and `f32`
semantics): it uses none of them. `lib/` re-synced to the 6.7.9 snapshot; `cyrius.lock` regenerated,
`deps --verify` 117/117 and `deps --locked` clean. Suite **73/73** unchanged; the demo and test output
(every FD `maxrel` included) are byte-identical to 1.1.5.

**1.1.5** — 2026-11-09. **Toolchain + substrate realign, no source change.** Pin `6.6.2` → `6.6.20`;
tyche 1.0.1 → **1.1.1**, rosnet 1.1.1 → **1.1.3**, akshara 1.0.2 → **1.0.4** (the 6.6.20-built set,
one consistent chain). `lib/` re-synced to the 6.6.20 snapshot (113 files; the shadow warning is gone);
`cyrius.lock` regenerated, `deps --verify` 117/117. Suite **73/73** unchanged; the demo's output is
byte-identical to 1.1.4, and quickstart still gives 1.58 → 23.94. (1.1.3, 2026-08-18: pin 6.2.37 → 6.5.27
plus the substrate's 1.0 tags. 1.1.4, 2026-09-11: pin 6.5.27 → 6.6.2.)

**1.1.2** — 2026-07-05. **Preference-file ingestion (`--pref`) — ifran Lane 3 closes** (`src/pref_ingest.cyr`).
`tarka --pref <prefs.jsonl>` trains DPO/IPO/KTO from an ifran `pref export` (curated preferences in,
aligned policy out): bayan flat-JSON parse per line (+ escape handling; embedded-quote rows skip loudly —
bayan's parser doesn't honor `\"`), akshara byte vocab over the whole file, bigram (state, action) spans
over completion-given-prompt, full-batch training via the EXISTING FD-gated primitives (no new gradient
math). Caps 48 pairs/seq · 256 rows/kind · 1 MB. E2E proof on a real ifran-curated set (3 pairs +
4 thumbs): 7/7 ingested, DPO loss 0.69→0.00 ranked **3/3**, IPO 1.00→0.00 **3/3**, KTO gap 0→**237.78** —
all gates green, demo suite regression clean. Suite **50 → 73** (`tests/pref_ingest.tcyr`); stdlib
`bayan` added to the manifest. Additive — the frozen surface unchanged. Pin 6.2.37.

**1.1.1** — 2026-06-27. **IPO + KTO — completing the standard preference-loss set** (`src/preference_ext.cyr`),
on the same frozen-reference machinery as DPO (1.1.0). **IPO** (Azar 2024) regresses the bare implicit-reward
margin `h` toward a FINITE target `1/(2β)` via `L=(h−1/(2β))²` (pulls `h` back if it overshoots, where DPO
pushes to ∞). **KTO** (Ethayarajh 2024) is UNPAIRED — each example labeled desirable/undesirable against a
**detached** KL reference point `z`; gradient coefficient is the prospect-value magnitude `σ(u)(1−σ(u))`
(not DPO's `1−σ`). Demo: IPO margin **h 0.00 → 0.59** toward 1.0; KTO desirable ρ **0.00 → 80.31**,
undesirable ρ **0.00 → −138.27**. Both hand-derived backwards FD-grad-checked + falsifiers (IPO pull-to-margin;
KTO two-sided z-detachment — z-detached FD ~14e-9 vs z-live diverges ~9.7 — + direction); suite
**34/34 → 50/50**. Additive — the frozen v1.0 surface unchanged, all prior gates byte-identical. Pin 6.2.37.

**1.1.0** — 2026-06-25. **Alignment from preferences — DPO + the RLHF KL-to-reference-policy
penalty** (`src/dpo.cyr`), both over a FROZEN reference policy (`dpo_snapshot`). DPO reparametrizes
the Bradley-Terry loss onto the policy (`Δ = β·[(log π_θ − log π_ref)_w − (...)_l]`, `L = −ln σ(Δ)`,
β=0.1; Rafailov 2023) — reuses `reward.cyr`'s `bt_loss` and `rl.cyr`'s softmax-CE seed, no reward
model / critic / sampler. The KL penalty `β·KL(π_θ‖π_ref)` (Ouyang 2022; `dKL/dlogit_k = p_k·(f_k−KL)`)
pulls a diverged policy back to the reference. Demo: DPO raises target freq **0.94 → 24.00 / 24** from
preferences alone; KL penalty pulls mean KL **3.33 → 2.46**. Both hand-derived backwards FD-grad-checked
+ descent-direction falsifiers; suite **24/24 → 34/34**. Additive — the frozen v1.0 surface unchanged,
all prior gates byte-identical. Surfaced by the 2026-06-25 ifran/secureyeoman product-mining (both
shell DPO/RLHF to Python — demand evidence, not ported code). Pin 6.2.37.

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

- **Cyrius pin**: `6.7.18` (in `cyrius.cyml [package].cyrius`), the latest release and the pin
  tyche 1.2.2 / rosnet 1.4.1 / akshara 1.1.2 build against. `lib/` byte-matches the 6.7.18 snapshot,
  so there is no shadow warning.

**Pin history:** 6.2.36 → 6.2.37 (0.8.0) → 6.5.27 (1.1.3, 2026-08-17) → 6.6.2 (1.1.4, 2026-09-11) →
6.6.20 (1.1.5, 2026-11-09) → 6.7.9 (1.2.0, 2026-11-09) → **6.7.18 (1.2.1, 2026-11-09)**. Each bump
re-ran `cyrius lib sync --full` and re-verified the suite at the new pin. From 6.7.12, a moved pin
makes `cyrius deps` / `cyrius build` exit 1 until `lib sync --full` runs, so the pin, `lib/` and
`cyrius.lock` land in one commit. From 6.6.20, bare `cyrius test` also runs the `[build].test`
stub (`src/test.cyr`), so the runner reports 3 files; `cyrius test tests` runs the two suites only.
`cyrius tests` is deprecated at 6.7.6 and removed at 6.7.12; tarka never used it.

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

**Shared kernels + the distribution memo — 1.2.2:** every policy gradient ends in `pol_seed_bwd`
(`src/rl.cyr`), every softmax in `softmax_into`, every Adam step in `adam_kernel`; PPO in
`ppo_update`, GRPO in `grpo_collect` / `grpo_update`. Inside tarka's training and evaluation loops
the policy's, the reference's and the reward model's per-state outputs are computed once per
parameter version (`memo_begin` / `memo_end`;
[architecture note 001](../architecture/001-distribution-memo.md)). Bits unchanged
([ADR 0002](../adr/0002-optimizations-preserve-bits.md)).

**Prior-guided MCTS — 1.4.0** ([ADR 0003](../adr/0003-prior-guided-mcts.md)):
- `src/mcts.cyr` — the corridor task (K1 → K2 → K3 open a GOAL worth +4 per step; a TRAP pays +1;
  an out-of-order entry costs 60; optimum by dynamic programming), a policy behavior-cloned from
  noisy demonstrations, its critic by policy evaluation, **PUCT** (priors from π, the critic as
  leaf value, subtree reuse, most-visited commit) and four beams on the same networks.
- Matched compute (forwards per prompt, 40 prompts): greedy 25.19 @16, best-of-64 36.37 @2,048,
  myopic beam 24.19 @2,844, prior beam 27.89 @1,118, **MCTS 54.00 @1,084**, **prior + value beam
  53.42 @543**. The learned value's lookahead solves it; on a deterministic task a value-ranked
  beam gets it at less compute, and the gate says so.

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

**Alignment — 1.1.x:**
- `src/dpo.cyr` — frozen reference policy (`dpo_snapshot`), **DPO** (target freq 0.94 → 24.00),
  the RLHF **KL-to-reference** penalty (`kl_penalty_step`; mean KL 3.33 → 2.46).
- `src/preference_ext.cyr` — **IPO** (h 0.00 → 0.85 toward 1.00) and **KTO** (ρ +80.31 / −138.27).
- `src/pref_ingest.cyr` — `tarka --pref <file>`: validating two-pass JSONL ingestion of an ifran
  `pref export` (one JSON object per line, strict kinds and labels, RFC 8259 escapes, vocab over
  decoded bytes, 1 MB / 256-row / 48-pair caps, nine counted skip reasons) → DPO / IPO / KTO,
  gated. `src/report.cyr` — the printers it shares with the demo.

## Tests

**266 assertions in four suites** (`cyrius test` also runs the `src/test.cyr` stub):
- `tests/tarka.tcyr` — **grad-check suite, 59** (M1: 4; M2: +15 — critic dW/db/dE
  FD-exact; GAE recursion==explicit + return identity; PPO ρ=1==REINFORCE / unclipped FD
  3e-9 / binding-clip→0; GRPO group-norm anchors + ratio=1==REINFORCE; M3: +5 — RM
  dWr/dbr/dEr Bradley-Terry FD-exact + descent-direction falsifier; 1.1.0: +10 — DPO +
  KL penalty; 1.1.1: +16 — IPO + KTO; 1.3.0: +9 — two-length DPO / IPO with unequal sides).
- `tests/pref_ingest.tcyr` — **preference-file ingestion, 117** (1.1.2: 23; 1.2.2: +65 — escape
  bytes tokenize, raw-vocab refusal, the full escape set, strict labels, field types and line
  structure, identical spans, the row cap and skip counts, KTO class sizes; 1.3.0: +29 — option
  parsers, early stop, each side at its own length).
- `tests/units.tcyr` — **75** (1.2.2: 38 — the memo's invalidation rules, parity / reasoning /
  search helpers, beam(1,1) == best-of-1, GAE normalization, GRPO's equal group, buffer
  re-sizing; 1.3.0: +6 — the aggregated gradient against the per-token one, discard; 1.4.0: +31 —
  the corridor's reward and oracle, the stable sort, the tree's visit books).
- `tests/golden.tcyr` — **15** (1.2.2; 1.4.0: + `mcts`): FNV-1a digests of every trainer's bits,
  per target.
- `tests/abort/` — 18 guard probes (each must exit 1 with the guard's message; run by CI).
- `tests/tarka.fcyr` — seeded `--pref` fuzz harness, 5 properties × 4,000 rounds (~1.2 s).
- `tests/tarka.bcyr` — 17 benchmarks; `scripts/bench-history.sh` → `bench-history.csv`.
- Reference coverage (`cyrius coverage`): 132 / 208 functions; the demo, run by CI, exercises the rest.

Grad-check discipline (every hand-derived backward verified against central finite
differences before it lands, inherited from attn11) is **active as of M1**.

## Dependencies

Direct (declared in `cyrius.cyml`):

- **stdlib** — string, fmt, alloc, io, vec, str, syscalls, assert, bench, math, args, bayan
- **[rosnet](https://github.com/MacCracken/rosnet) 1.4.1** — f64 tensor storage,
  BLAS-1, matmul + gradient (CPU bundle). The policy/value networks build on this.
- **[tyche](https://github.com/MacCracken/tyche) 1.2.2** — deterministic PRNG for
  on-policy rollout sampling + weight init. (rosnet's `t_randn` resolves through it.)
- **[akshara](https://github.com/MacCracken/akshara) 1.1.2** (wired 0.2.1) — the
  sovereign tokenizer (अक्षर); `corpus_set` builds the byte vocab + packed store,
  `gd_ld` indexes it for rollout prompts. Pure path only — loader/streaming/emit
  consumer symbols (`secure_read_file`/`read_stdin`/`file_seek`) are stubbed unreached.
  The third attn11 extraction after rosnet/tyche; one tokenizer, shared with attn11.

## Consumers

_None yet._ (Long-horizon: a sovereign agent/reasoning runtime — daimon-adjacent —
is the eventual downstream.)

## Next

1.5.x: a GPU backend through rosnet's GPU bundle. Unscheduled: MCTS on a stochastic or
adversarial task (where the tree, not only the value, should matter), an aggregated backward for
the reward model, a `PMIN` floor for `f64_ln`. The 2.0.0 backlog holds the changes
that would break the frozen 1.x API. See [`roadmap.md`](roadmap.md).
