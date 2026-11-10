# tarka — Roadmap

> What ships next, by release line. Shipped work is in [`CHANGELOG.md`](../../CHANGELOG.md)
> (0.1.0 → 1.3.0); live state is in [`state.md`](state.md).
>
> **The 1.x rule.** A 1.x release may add symbols to the frozen API ([`docs/api.md`](../api.md))
> and may move results as a *labeled behavior change* — it re-records the golden cases it
> touches and names them in the CHANGELOG ([ADR 0002](../adr/0002-optimizations-preserve-bits.md)).
> It may not change or remove a frozen signature. Anything that would is in the
> [2.0.0 backlog](#200-backlog--breaks-the-frozen-1x-api).

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

- **An aggregated backward for the reward model.** 1.3.0 aggregates the policy's backward per
  state ([architecture note 002](../architecture/002-aggregated-backward.md)); the reward model's
  (`rm_accum_seed`) still runs one `linear_bwd` per step, with a one-hot seed. The same rewrite
  applies — and moves the RM's bits, so it is a labeled behavior change. RM training is a small
  share of the demo today (the PRM step row of `bench-history.csv`: 24 µs).
- **A `PMIN` floor for `f64_ln`.** A probability that underflows to exactly 0 gives `−inf`. Both
  audits accepted it: max-shifted softmax keeps probabilities positive on every path tarka drives.

## 2.0.0 backlog — breaks the frozen 1.x API

- **Retire the single-n preference signatures** (`dpo_delta`, `dpo_pair_loss`,
  `dpo_backward_pair`, `ipo_h`, `ipo_pair_loss`, `ipo_backward_pair`): since 1.3.0 they are the
  two-length forms at `nW = nL = n`.
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
