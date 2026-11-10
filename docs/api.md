# tarka — Public API (frozen at v1.0.0; additively extended at v1.1.0 + v1.1.1 + v1.1.2 + v1.2.2 + v1.3.0 + v1.4.0 + v1.5.0)

> Satisfies the v1.0 criterion *"Public RL/reasoning API frozen — every exported symbol
> documented and tested."*

## Freeze policy

The **stable public API** below is frozen for the 1.x series: signatures and semantics will not
change without a major version bump. Minor/patch versions may **add** symbols (v1.1.0 added the
alignment surface — DPO + the RLHF KL penalty in `src/dpo.cyr`; v1.1.1 added IPO + KTO in
`src/preference_ext.cyr` — without touching any existing signature). v1.2.2 added
`pref_corpus` / `pref_skip_count` / `PI_R_*`, `kl_penalty_step` and `src/report.cyr`, changed no
signature, and tightened three behaviors, each noted in its row: `ipo_mean_h` skips tied draws,
`reason_verifier_gate` no longer passes when a class is empty, and `--pref` refuses input it used
to mis-parse. v1.3.0 added the two-length preference primitives (`dpo_*2`, `ipo_*2`),
`tarka_pref_ingest_opts` and the `--pref` options, again without changing a signature; its
aggregated backward moves training results by floating-point rounding (a labeled behavior change,
[ADR 0002](adr/0002-optimizations-preserve-bits.md)). v1.4.0 added `src/mcts.cyr`; v1.5.0 added the
device path (`dev_table`, its counters, `--gpu`), again changing no signature and no CPU result. Each symbol is exercised by the demo (`src/main.cyr`), the suites (`tests/tarka.tcyr`
grad-checks, `tests/pref_ingest.tcyr`, `tests/units.tcyr`, the bit-for-bit `tests/golden.tcyr`),
and/or the quickstart consumer (`examples/quickstart.cyr`) — noted per row.

**Not frozen (internal):** the per-element optimizer internals (`adam_one`, `*_adam_step`),
gradient accumulators / zeroers (`*_zero_grads`, `*_backward`, `gae_*`, `ppo_snapshot`,
`grpo_group_adv`/`grpo_ratio`/`grpo_coeff`), reward-model internals (`rm_logits`,
`sigma_stable`, `bt_loss`, `rm_accum_seed`, `*_fill`, `orm_bufs`, `rm_sample_to`), the
matched-compute counting wrappers (`cnt_*`), and module-private helpers (`par_rw`, `RUNI`,
`reason_oracle`/`reason_answer`, `target_wrong`, `pol_argmax`). Since v1.2.2 also the shared
kernels (`softmax_into`, `pol_seed_bwd`, `adam_kernel`, `pol_scale_grads` / `rm_scale_grads`,
`ppo_update`, `grpo_collect` / `grpo_update`), the distribution memo (`memo_begin` / `memo_end`,
`pol_dist`, `ref_dist`, the `PM_*` / `RF_*` / `RC_*` tables), the dims checks (`ac_ready`,
`dpo_ready`), and since v1.3.0 the aggregated backward (`pol_bwd_seeded`, `pol_bwd_flush`,
`pol_bwd_discard`, the `PB_*` tables). These are the reference's *mechanism*; they are readable and grad-checked but may
change between minor versions.

Convention: dimensions/seed are set by `pol_init`; call each module's `*_init` (`ac_init`,
`rm_init`, `dpo_init`, `search_init`) after every `pol_init` — their public entries check that the
dims they were sized for are the current ones and abort (fail loud) when not. `g_datalen` selects
the corpus (`0` = synthetic vocab). Working buffers are allocated in the `*_init` functions, and a
few training-loop buffers on first use, re-sized when the rollout length changes; none is
allocated per step. Inside tarka's own training and evaluation loops the policy's, the frozen
reference's and the reward model's per-state outputs are cached
([architecture note 001](architecture/001-distribution-memo.md)); writing `P_*` directly between
calls (as the grad-checks do) is safe — the next call recomputes. Inside those loops the policy's
backward is also aggregated per state and applied when the gradient is next needed
([architecture note 002](architecture/002-aggregated-backward.md)); outside them every backward
writes `G_*` at once, as before. Token arguments outside
`0..V-1`, a batch or count of 0, and `pol_init` dims below 1 abort with `tarka: precondition
violated: …` (since v1.3.0 through the stdlib's `panic`, so the stderr line reads `panic: tarka:
precondition violated: …`; exit status 1).

## Policy + REINFORCE — `src/rl.cyr`

| symbol | purpose | tested by |
|--------|---------|-----------|
| `pol_init(V, D, T, target, seed)` | allocate + seed the policy (vocab, embed dim, rollout len, reward token) | demo, suite, quickstart |
| `rl_train(steps, batch, lr, log_every)` | on-policy REINFORCE training | demo, quickstart |
| `rl_eval(nroll)` → f64 | mean reward over `nroll` rollouts | demo, quickstart |
| `rl_rollout(prompt)` / `rl_prompt()` → i64 / `rl_reward()` → f64 | sample a rollout / draw a prompt / score the current rollout | demo, suite |
| `pol_logits(x)` / `pol_softmax()` / `pol_sample()` → i64 | per-step distribution / normalize / sample (custom decoding) | demo, suite |
| `guard(cond, msg)` | fail-loud precondition guard (stdlib `panic`: `panic: tarka: precondition violated: <msg>`, exit 1) — the hardening primitive | demo, `tests/abort/` (18 probes) |
| `ADAM_B1()` `ADAM_B2()` `ADAM_EPS()` `RL_BETA()` | optimizer / baseline hyperparameters | suite |

## Actor-critic: PPO + GRPO — `src/actorcritic.cyr`

| symbol | purpose | tested by |
|--------|---------|-----------|
| `ac_init()` | allocate the critic + GAE/PPO/GRPO buffers (call after `pol_init`) | demo, suite |
| `ppo_train(steps, epochs, lr, log_every)` | PPO (value critic + GAE + clipped surrogate) | demo |
| `grpo_train(steps, G, lr, log_every)` | GRPO (group-relative, no critic); `1 ≤ G ≤ GMAX()` | demo |
| `GAMMA()` `LAM()` `GLAM()` `CLIP_EPS()` `ADV_EPS()` `VAL_COEF()` `GMAX()` | discount / GAE-λ / clip / coefficients / max group size | suite |
| `f64_min2(a,b)` `f64_max2(a,b)` `f64_clip(v,lo,hi)` | f64 numeric utilities | demo, suite |

## Learned reward model — `src/reward.cyr` (+ training in `src/preference.cyr`)

| symbol | purpose | tested by |
|--------|---------|-----------|
| `rm_init()` | allocate the Bradley-Terry reward model (own embedding) | demo, suite |
| `rm_reward(x, a)` → f64 / `rm_score(states, actions, n)` → f64 | query the learned reward / sum over an `n`-step prefix | demo, suite |
| `rm_train_prm(steps, batch, lr)` / `rm_train_orm(steps, batch, lr)` | train from step-level (PRM) / whole-rollout (ORM) preferences | demo |
| `rm_prm_accuracy(n)` / `rm_orm_accuracy(n)` → i64 | held-out preference accuracy | demo |
| `rm_ppo_step(epochs, lr)` / `rm_orm_grpo_step(G, lr)` | one RL step against the frozen learned reward (PRM→PPO / ORM→GRPO) | demo |
| `par_correct(x)` / `par_true_reward(st, ac)` | the parity-task oracle (true reward, never shown to the RM) | demo, suite |

## Alignment: DPO + RLHF KL penalty — `src/dpo.cyr` (added v1.1.0, additive)

Direct preference optimization and the RLHF KL-to-reference-policy penalty, both evaluated
against a **frozen reference policy** (`dpo_snapshot`). Both hand-derived backwards are
finite-difference grad-checked (`tests/tarka.tcyr`).

| symbol | purpose | tested by |
|--------|---------|-----------|
| `dpo_init()` | allocate the frozen reference-policy buffers (call after `pol_init`) | demo, suite |
| `dpo_snapshot()` | freeze the current policy as the reference π_ref | demo, suite |
| `dpo_train(steps, batch, lr)` | DPO training from sampled preferences (higher-reward rollout = winner) | demo |
| `dpo_backward_pair(stW, acW, stL, acL, n, beta)` → f64 | one preference pair's DPO gradient (returns the loss) | demo, suite |
| `dpo_pair_loss(stW, acW, stL, acL, n, beta)` → f64 / `dpo_delta(...)` → f64 | forward-only DPO loss −ln σ(Δ) / the implicit-reward margin Δ | suite |
| `dpo_backward_pair2(stW, acW, nW, stL, acL, nL, beta)` → f64 / `dpo_pair_loss2(...)` / `dpo_delta2(...)` → f64 | the same with each side at its own length — the full-sequence DPO objective (added v1.3.0); the single-n forms are these at `nW = nL = n` | suite (FD, unequal sides), `--pref` |
| `kl_backward_state(x, beta)` | accumulate the β·KL(π_θ‖π_ref) penalty gradient at state `x` | demo, suite |
| `kl_loss(x, beta)` → f64 / `kl_div_at(x)` → f64 | forward KL penalty / raw KL at a state | demo, suite |
| `dpo_mean_kl(n)` → f64 | mean KL to the reference over `n` random states | demo |
| `kl_penalty_step(beta, lr)` | one optimizer step of the KL penalty alone: its gradient at every state, meaned over V, one Adam step (added v1.2.2) | demo, golden |
| `pol_logp(x, a)` / `ref_logp(x, a)` → f64 | policy / reference log-prob of an action | suite |
| `DPO_BETA()` | the DPO / penalty temperature β (0.1) | demo, suite |

**Not frozen (internal mechanism):** `pol_accum_scaled`, `dpo_seqlogp_theta`/`dpo_seqlogp_ref`,
`ref_logits`/`ref_softmax`, `dpo_bufs`/`dpo_sample_to`/`dpo_count_target`.

## Preference-optimization extensions: IPO + KTO — `src/preference_ext.cyr` (added v1.1.1, additive)

Two more preference losses on the same frozen-reference machinery. IPO regresses the implicit-reward
margin toward a finite target (it does not push to ∞ like DPO); KTO is unpaired (per-example
desirable/undesirable against a detached reference point). Both hand-derived backwards are
finite-difference grad-checked (`tests/tarka.tcyr`), including an IPO pull-to-margin falsifier and a
KTO two-sided z-detachment falsifier.

| symbol | purpose | tested by |
|--------|---------|-----------|
| `ipo_backward_pair(stW, acW, stL, acL, n, beta)` → f64 | one preference pair's IPO gradient `L=(h−1/(2β))²` (returns the loss) | demo, suite |
| `ipo_pair_loss(stW, acW, stL, acL, n, beta)` → f64 / `ipo_h(...)` → f64 / `ipo_target(beta)` → f64 | forward IPO loss / the bare margin `h` / the target margin `1/(2β)` | demo, suite |
| `ipo_backward_pair2(stW, acW, nW, stL, acL, nL, beta)` → f64 / `ipo_pair_loss2(...)` / `ipo_h2(...)` → f64 | the same with each side at its own length (added v1.3.0) | suite (FD, unequal sides), `--pref` |
| `ipo_train(steps, batch, lr)` / `ipo_mean_h(npairs)` → f64 | IPO training from sampled preferences / mean implicit-reward margin over the preference pairs among `npairs` draws (v1.2.2: a tied draw has no winner and is skipped, as training skips it; 0 if every draw ties) | demo, golden |
| `IPO_BETA()` → f64 | the IPO regularization strength β (0.5 → target margin 1.0) | demo, suite |
| `kto_backward_example(states, actions, n, label, beta, lamD, lamU, z)` → f64 | one labeled example's KTO gradient (`label`>0 desirable; returns the loss) | demo, suite |
| `kto_example_loss(states, actions, n, label, beta, lamD, lamU, z)` → f64 | forward KTO loss `λ(1−σ(u))` with a detached reference point `z` | suite |
| `kto_logratio(states, actions, n)` → f64 / `kto_zref(kl_st, kl_ac, n, m)` → f64 | bare log-ratio ρ / the detached (clamped ≥0) batch reference point | demo, suite |
| `kto_train(steps, batch, lr)` / `kto_mean_rho(n, thresh, want_desirable)` → f64 | unpaired KTO training (label by batch-mean count; `1 ≤ batch ≤ KTO_BMAX()`) / mean ρ over a population | demo / golden |
| `KTO_BETA()` `KTO_LAMBDA_D()` `KTO_LAMBDA_U()` → f64 | KTO temperature β (0.1) and desirable/undesirable weights (1.0) | demo, suite |

**Not frozen (internal mechanism):** `kto_zref_raw`, `kto_example_loss_live` (the detachment-falsifier
forward), `kto_bufs` (`ipo_bufs` folded into `dpo_bufs` at v1.2.2).

## Preference-file ingestion — `src/pref_ingest.cyr` (added v1.1.2, additive)

The ifran-Lane-3 data path: train the preference losses from a curated
`ifran pref export` JSONL file instead of on-policy rollouts. No new gradient
math — the FD-gated `dpo_backward_pair` / `ipo_backward_pair` /
`kto_backward_example` primitives above do the training.

| symbol | purpose | tested by |
|--------|---------|-----------|
| `tarka --pref <prefs.jsonl> [--epochs N] [--lr X] [--dim D] [--early-stop]` (CLI) | ingest + train + gate DPO/IPO/KTO from an ifran preference export; exit 0 = all gates green, 1 = unreadable / over the cap / nothing ingested / a gate failed, 2 = a usage error. Options since v1.3.0: epochs per optimizer (1..100000, default 300), learning rate (a decimal in (0, 10], default 0.05), policy embedding dim (1..256, default 8), and stopping each optimizer once its gate holds (checked every 10 epochs). The defaults run exactly as before | CI, golden |
| `tarka_pref_ingest(path)` → i64 / `tarka_pref_ingest_opts(path, epochs, lr, dim, early)` → i64 | the driver at the defaults / with its settings (`lr` an f64; added v1.3.0) | CI |
| `pref_parse_uint(s)` → i64 / `pref_parse_dec(s)` → f64 | the option parsers: a 1..9-digit integer or −1 / a positive decimal `I` or `I.F` as f64 bits, or 0 (added v1.3.0) | suite |
| `PI_EPOCHS()` `PI_LR()` `PI_DIM()` | the option defaults: 300, 0.05, 8 (added v1.3.0) | suite |
| `pref_corpus(buf, flen, dst)` → i64 | pass 1 (added v1.2.2): write the decoded text of every well-formed row to `dst` (≥ `flen` bytes) and return its length; `corpus_set` over it gives a byte vocab with an id for every byte pass 2 tokenizes | suite, golden, fuzz |
| `pref_ingest_file(buf, flen)` → i64 | pass 2: parse JSONL lines into the pair/unary tables and return the rows ingested; since v1.3.0 a pair row keeps each side's own length (48 B: stW acW nW stL acL nL). Needs a byte vocab (`corpus_set`, over `pref_corpus`'s text) and a policy over it (`pol_init` with that V) — guarded. A line is accepted only as one JSON object whose `kind` is the string `"pair"` / `"unary"`, whose text fields are JSON strings (full RFC 8259 escapes, `\uXXXX` → UTF-8) and whose unary `label` is the bare number `1` or `-1` (v1.2.2) | suite, golden, fuzz |
| `pref_skip_count(r)` → i64 / `PI_R_*()` / `PI_NREASON()` | rows refused for reason `r` (added v1.2.2): `PI_R_MALFORMED` 1, `PI_R_FIELD` 2, `PI_R_ESCAPE` 3, `PI_R_LABEL` 4, `PI_R_KIND` 5, `PI_R_EMPTY` 6, `PI_R_SAME` 7 (a pair whose two spans are identical — same length and tokens since v1.3.0), `PI_R_CAP` 8, `PI_R_VOCAB` 9 (a byte without a token id); `PI_skipped` is their sum | suite, fuzz |
| `PI_NMAX()` `PI_MAXEX()` `PI_FMAX()` | caps: 48 (state,action) pairs/sequence · 256 rows/kind · 1 MB file (a larger file is refused, v1.2.2) | suite, CI |

**Not frozen (internal mechanism):** `_pi_*` (parse, unescape, tokenize, sequence
shaping, eval/train/fit loops, row accessors), `PI_PAIRB`, the `PR_*` parsed-row cells and the
`PI_*` tables.

## Parity task + sample-efficiency benchmark — `src/parity.cyr`

| symbol | purpose | tested by |
|--------|---------|-----------|
| `bench_median(method, param, lr)` → i64 | rollouts-to-threshold, median of 5 seeds (`method` 0/1/2 = REINFORCE/PPO/GRPO) | demo |
| `par_eval(nroll)` → f64 | mean true parity reward over `nroll` rollouts | demo |
| `par_reinforce_step(B, lr)` / `par_ppo_step(epochs, lr)` / `par_grpo_step(G, lr)` | one training step on the parity task | demo |
| `BENCH_V/D/T/TGT/DECOY/THRESH/MAXROLL/EVAL_EVERY/NSEED()` | parity-task configuration | demo |

## Reasoning — `src/reasoning.cyr`

| symbol | purpose | tested by |
|--------|---------|-----------|
| `reason_setup()` / `reason_setup_q(qsteps)` | set up the reasoning task (partial-trained policy + frozen PRM verifier) | demo |
| `reason_self_consistency(p, N)` → i64 | majority-vote answer over `N` rollouts | demo |
| `reason_accuracy(method, N, nprompts, base_seed)` → i64 | answer accuracy ‰ (`method` 0 = single, 1 = self-consistency) | demo |
| `reason_best_of_n(mode, N, nprompts, base_seed)` → i64 | true reward ×100 (`mode` 0 mean / 1 verifier-selected / 2 oracle) | demo |
| `reason_verifier_gate(n)` → i64 | 1 if the verifier ranks correct above wrong; 0 if either class is empty (v1.2.2; it returned 1) | demo, golden |
| `REASON_V/D/T/TGT/DECOY/STEPS()` | reasoning-task configuration | demo |

## Verifier-guided search — `src/search.cyr`

| symbol | purpose | tested by |
|--------|---------|-----------|
| `search_setup()` / `search_init()` | set up the search task + verifier (and, since v1.2.2, the beam buffers) / allocate the beam buffers for the current `g_T` | demo, units |
| `srch_greedy(nprompts, base_seed)` → i64 | greedy-decode baseline, mean #TARGET ×100 | demo |
| `srch_best_of_n(N, nprompts, base_seed)` → i64 | best-of-N (verifier-selected), mean #TARGET ×100 | demo |
| `srch_beam(B, K, nprompts, base_seed)` → i64 | PRM-guided beam search; `1 ≤ B ≤ BMAX()`, `B*K ≤ BMAX()²` | demo |
| `target_streak(st, ac)` → f64 | the metric (#TARGET in a rollout) | demo |
| `TARGET()` `SR_V/D/T()` `BMAX()` / `g_fwd` | task config / max beam width / forward-pass counter | demo |

## Prior-guided MCTS — `src/mcts.cyr` (added v1.4.0)

PUCT with a behavior-cloned policy prior and a policy-evaluation critic, on the corridor task
(a delayed payoff behind three zero-reward keys), against greedy, best-of-N and four beams given
the same networks. The evaluator is the task's exact reward. See
[ADR 0003](adr/0003-prior-guided-mcts.md) for the four formulations measured and why this one
shipped; the honest result is that a beam given the same prior and value matches MCTS at less
compute.

| symbol | purpose | tested by |
|--------|---------|-----------|
| `mcts_setup()` / `mcts_setup_n(bc_steps, critic_steps, seed)` | the corridor task: a policy behavior-cloned from noisy demonstrations, its critic trained by policy evaluation, the search buffers (the demo's setup: 200, 400, seed 7) | demo, golden, units |
| `mcts_init()` | allocate the beam and tree buffers for the current dims (`mcts_setup_n` calls it) | demo |
| `mc_mcts(S, c, nprompts, base_seed)` → f64 | PUCT: S simulations per step (`2 ≤ S`, `S·T < MC_NMAX()`), exploration weight `c` (f64), subtree reuse, most-visited commit; mean true return | demo, golden, units |
| `mc_beam(B, K, mode, nprompts, base_seed)` → f64 | beam of width `B ≤ MC_BMAX()`: mode 0 exhaustive myopic, 1 exhaustive + V, 2 `K` children from π, 3 `K` from π + V; mean true return | demo, golden |
| `mc_best_of_n(N, nprompts, base_seed)` / `mc_pgreedy(nprompts, base_seed)` → f64 | best-of-N π samples by return / argmax-π decoding | demo, golden |
| `mc_oracle(nprompts, base_seed)` / `mc_oracle_from(p)` → f64 | the true optimum (dynamic programming over r*) over the prompts / from one prompt | demo, units |
| `mc_true_r(x, a)` / `mc_true_return(st, ac)` → f64 | the task's reward / the return of a stored sequence | units |
| `mc_bc_train(steps, batch, lr, psol, eps)` / `mc_critic_train(steps, lr)` | behavior cloning (solver share and noise per mille) / policy evaluation of the frozen π | demo, golden |
| `MC_V/D/T()` `MC_TRAP/K1/K2/K3/GOAL()` `MC_PEN()` `MC_QSCALE()` `MC_BMAX()` `MC_NMAX()` | task configuration, tokens, the out-of-order penalty, PUCT's Q scale, beam / tree caps | units |

**Not frozen (internal mechanism):** `mc_score` / `mc_value` (the counted evaluator and critic
calls), `mc_demo`, `mc_opt_action`, `mc_dp_table`, `mc_fill_rewards`, `mc_prompts`, `mc_sort_desc`,
`mc_cand`, `mc_node`, `mc_puct_pick`, `mc_simulate`, `mc_commit_child`, and the `EB_*` / `EN_*` /
`EC_*` / `ND_*` / `MC_*` tables.

## Device path — `--gpu` (added v1.5.0)

The memo regions' logits tables on the GPU, through rosnet's GPU bundle (`lib/rosnet-gpu.cyr`,
rosnet ADR 0001) on mabda. Only rosnet's **bit-exact class** is used (`gpu_matmul_fwd`), so every
result is the CPU's; the gate is parity, not speed — on the reference iGPU it is slower than the CPU at
every shape tarka has ([ADR 0004](adr/0004-gpu-parity-backend.md)). With `g_gpu = 0` (the default) none
of it runs.

| symbol | purpose | tested by |
|--------|---------|-----------|
| `g_gpu` | rosnet's gate (consumer-set): 1 = a memo miss fills the model's whole logits table on the device; `tarka --gpu` sets it | golden (`--gpu` pass), units, CI |
| `dev_table(E, W, b, Y)` → 0 / non-0 | `Y(V×V) = E·W + b` for every state at once on the device, `D` zero-padded to rosnet's `GPU_TK()`; non-0 = declined (no device, a shape the device does not take, a failed dispatch), `Y` untouched | units |
| `g_dev_fills` / `g_dev_declined` | tables the device filled / declined since start | units, golden, demo |

A declined table is computed row by row on the CPU, as without `--gpu`, and each model's table is
tried once per parameter version.

**Not frozen (internal mechanism):** `pol_miss_logits`, `ref_miss_logits`, `rm_dev_fill`,
`gpu_start`, `gpu_report`, and the `DV_*` / `PM_lg*` / `RF_lg*` / `RC_try` state.

## Report formatting — `src/report.cyr` (added v1.2.2)

| symbol | purpose | tested by |
|--------|---------|-----------|
| `print_f2(val)` | print an f64 with 2 decimals (truncated); NaN as `nan`, the infinities as `inf` / `-inf` | demo, CI |
| `print_cd(v)` | print a ×100 integer as `X.XX` (negatives with a sign) | demo |
| `rp_puts(s)` / `rp_putc(ch)` | print a NUL-terminated string / one byte through the stdlib | demo |

## Command line

`tarka` runs the demo (exit 0 = all gates pass, 1 = a gate failed); `tarka --pref <file>
[options]` runs the ingestion driver (options above); `tarka --help` prints usage; any other
argument, a missing file, an unknown or out-of-range `--pref` option prints usage and exits 2
(v1.2.2; earlier versions ran the demo). A leading `--gpu` (v1.5.0) runs either one with the device
path on; its own lines start with `GPU: `, and every other line is what the run prints without it.

## Quickstart

```sh
cyrius build examples/quickstart.cyr build/quickstart && ./build/quickstart
```

```cyrius
pol_init(16, 8, 24, 3, 1337);   # vocab 16, dim 8, rollout 24, reward token 3
g_datalen = 0;                  # synthetic vocab
rl_eval(200);                   # mean reward before
rl_train(300, 16, lr, 0);       # REINFORCE
rl_eval(200);                   # mean reward after  (≈ 1.6 -> 23.9)
```

See [`examples/quickstart.cyr`](../examples/quickstart.cyr) for the full consumer.
