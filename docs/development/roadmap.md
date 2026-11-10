# tarka — Roadmap

> What ships next, by release line. Shipped work is in [`CHANGELOG.md`](../../CHANGELOG.md)
> (0.1.0 → 1.6.0); live state is in [`state.md`](state.md).
>
> **The 1.x rule.** A 1.x release may add symbols to the frozen API ([`docs/api.md`](../api.md))
> and may move results as a *labeled behavior change* — it re-records the golden cases it
> touches and names them in the CHANGELOG ([ADR 0002](../adr/0002-optimizations-preserve-bits.md)).
> It may not change or remove a frozen signature. Anything that would is in the
> [2.0.0 backlog](#200-backlog--breaks-the-frozen-1x-api).

## 1.x.x — unscheduled

Recorded so they are not lost; each waits for a caller that needs it.

- **MCTS against an opponent.** 1.6.0 showed MCTS's advantage under chance
  ([ADR 0005](../adr/0005-mcts-under-chance.md)). The other half is a two-player task, where a
  beam that maximizes over both players' moves ignores the opponent's replies. It needs a second
  policy (or self-play) and a minimax backup. Gate it as 1.4.0 and 1.6.0 were: against the
  strongest beam the same networks allow, with an ablation that names the cause.
- **A GPU path that pays.** 1.5.0's `--gpu` is bit-exact and slower than the CPU at every shape
  tarka has on the reference iGPU ([ADR 0004](../adr/0004-gpu-parity-backend.md)). It waits for a
  device or a batched kernel on which a dispatch beats the CPU. Then a tolerance class
  (`--gpu-tc`: the aggregated backward through `gpu_linear_bwd`, Adam through `gpu_adam_step`)
  would trade byte identity for speed, gated within rosnet's ~1e-13 class. Making tarka's Adam
  bit-exact with rosnet's (1 − β as 1/10, the bias correction by `pow`) is a labeled behavior
  change that moves every golden case.
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
