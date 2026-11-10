# tarka — Roadmap

> What ships next, by release line. Shipped work is in [`CHANGELOG.md`](../../CHANGELOG.md)
> (0.1.0 → 1.2.2); live state is in [`state.md`](state.md).
>
> **The 1.x rule.** A 1.x release may add symbols to the frozen API ([`docs/api.md`](../api.md))
> and may move results as a *labeled behavior change* — it re-records the golden cases it
> touches and names them in the CHANGELOG ([ADR 0002](../adr/0002-optimizations-preserve-bits.md)).
> It may not change or remove a frozen signature. Anything that would is in the
> [2.0.0 backlog](#200-backlog--breaks-the-frozen-1x-api).

## 1.3.x — training throughput and preference fidelity

Each item is a behavior change under ADR 0002: every demo and `--pref` gate stays green, the
grad-checks hold, and the golden cases it moves are re-recorded and named.

- **Per-state aggregated backward.** `linear_bwd` now dominates a training step (≈ 0.9 µs per
  token at V 28, D 8; a REINFORCE step runs 384). For a bigram, every token of one state shares
  one embedding row, so summing a batch's seeds per state and running one backward per state cuts
  the work by the batch's repetition factor. It reorders the gradient's sums, which is why it is
  not an optimization. *Done when:* the REINFORCE / PPO / GRPO / DPO / KTO step rows of
  `bench-history.csv` drop, measured in one session against 1.2.2.
- **A two-length preference primitive (additive).** DPO / IPO score both sides of a pair at the
  shared minimum length (the single-n `dpo_backward_pair` / `ipo_backward_pair` / `dpo_delta`).
  Add variants that take each side's own length — the standard DPO objective — FD grad-checked,
  alongside the single-n ones; `--pref` switches to them, so an unequal pair is no longer
  truncated. *Done when:* the new primitives pass their grad-checks and a pair whose sides differ
  only past the shorter one's end trains (today it is refused as identical spans).
- **`--pref` training knobs.** Epochs (300 per optimizer), learning rate (0.05) and embedding
  dim (8) are compiled in; at the caps with a large vocab a run takes tens of seconds (41 s for
  the 2026-11-09 review's 103 KB file at V 92). Add flags with today's values as defaults, and
  optionally stop an optimizer once its gate holds. *Done when:* the default run is
  byte-identical to 1.2.2's.
- **`guard()` on the stdlib's `panic`.** `lib/assert.cyr` now has `panic` / `assert_fatal`, which
  SECURITY.md's residual note was waiting for. Keep the `tarka: precondition violated: <msg>`
  line and exit status 1 — the 13 abort probes check both.

## 1.4.x — search

- **MCTS on a task that warrants it.** 0.7.0 scoped it out: the search task is deterministic with
  one good action per step, where beam search is the right tool. MCTS needs a task where value
  backup matters — stochastic transitions, or several good actions with delayed payoff — and is
  measured against beam on the existing matched-compute harness (`g_fwd`, uncached).

## 1.5.x — GPU backend

- **`--gpu` through rosnet's GPU bundle** (rosnet-gpu + mabda). The CPU path stays the reference
  oracle: its golden digests do not move, and GPU results are checked against it within rosnet's
  bit-exact / tolerance classes. The prerequisites are met: the GPU bundle compiles at cyrius
  ≥ 6.6.19 (the pin is 6.7.18), and since rosnet 1.2.2 it no longer needs the consumer to define
  `puts`.

## 1.x.x — unscheduled

Recorded so they are not lost; each waits for a caller that needs it.

- **A `PMIN` floor for `f64_ln`.** A probability that underflows to exactly 0 gives `−inf`. Both
  audits accepted it: max-shifted softmax keeps probabilities positive on every path tarka drives.

## 2.0.0 backlog — breaks the frozen 1.x API

- **Retire the single-n preference signatures** once the two-length ones (1.3.x) have a release
  behind them.
- **Drop the unused `log_every` parameter** of `rl_train`, `ppo_train` and `grpo_train`: no code
  reads it, and every caller passes a value that does nothing.
- **Model handles instead of module globals.** The policy, the frozen reference and the reward
  model are module-global state (`g_V` / `g_D` / `g_T`, `P_*`), one of each per process; 1.2.2's
  dims guards (`ac_ready`, `dpo_ready`, `rm_reward`, `srch_beam`) exist because a `pol_init` can
  silently invalidate another module's buffers. A handle per model (its dims, parameters and
  cache) would make the reference a second policy instance rather than a parallel copy of the
  policy code, and the guards structural.
- **Extract reusable primitives to a shared lib** (the rosnet / tyche / akshara pattern) once a
  second consumer appears. It moves public symbols out of tarka.

## Out of scope

- **The transformer itself** — that is attn11 / rosnet. tarka consumes a policy network; it does
  not re-implement attention.
- **A production agent runtime** — tarka is the reference; the runtime is downstream
  (daimon-adjacent).
