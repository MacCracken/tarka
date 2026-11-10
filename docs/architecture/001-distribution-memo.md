# 001 — The distribution memo: when a cached π(·|x) is valid

tarka's policy is a bigram: π(·|x) = softmax(E[x]·W + b) depends only on the state token x
and the parameters. Inside one training step (between two Adam steps) or one evaluation
loop, the parameters do not change, so the at most V distinct distributions get recomputed
hundreds of times — a REINFORCE step at the demo's shape runs 768 forwards over 28 states.
The same holds for the frozen reference π_ref and the reward model r_θ. 1.2.2 caches them
(`src/rl.cyr` `pol_dist`, `src/dpo.cyr` `ref_dist`, `src/reward.cyr` `rm_reward`); the
invariant below is what makes a cached row safe to use.

## The invariant

A cached row x is used only if it was filled **in the current memo generation, at the
model's current parameter version**.

- **Regions.** The cache is consulted only inside a memo region (`memo_begin` …
  `memo_end`, nesting by depth). tarka opens regions in its own training and evaluation
  loops only — `rl_train`, `rl_eval`, `ppo_train`, `grpo_train`, the parity steps and
  `eff_run`, the reward-model trainers and accuracies, the learned-reward PPO / GRPO steps,
  `reason_*`, `dpo_train`, `dpo_mean_kl`, `ipo_train`, `ipo_mean_h`, `kto_train`,
  `kto_mean_rho`, `rm_train_target`, and the `--pref` train / eval loops. Outside a region,
  `pol_dist(x)` is exactly `pol_logits(x); pol_softmax()`.
- **Generations.** Entering the outermost region bumps `g_memo_gen`; each model syncs its
  version to the generation on first use, which retires every row filled before the region.
  So code outside a region — a consumer, or the finite-difference grad-checks — may write
  `P_*` directly between calls and never see a stale row.
- **Versions.** Every write to a model's parameters inside a region goes through a function
  that bumps its version: the policy's `pol_adam_step` and `pol_init` (`PM_epoch`), the
  reference's `dpo_snapshot` and `dpo_init` (`RF_epoch`), the reward model's `rm_adam_step`
  and `rm_init` (`RC_epoch`). Those are the only writers (`t_randn` / `t_zero` / `t_copy` /
  Adam), checked by grep when the cache landed; a new writer must bump.
- **Copies.** A hit copies the row into `S_probs` / `S_refprobs` (the reward model returns
  the one logit it needs). Callers that modify `S_probs` in place — every backward seeds it —
  therefore never touch the cached row.

## What it does not cache

- **The beam-search demo** (`srch_*`, `cnt_*`). Its result is reported at matched compute,
  counted in forward passes; caching there would make the count describe work that did not
  run. The search *verifier's training* (`rm_train_target`) is cached.
- **The forward-only loss functions the grad-checks perturb** (`pol_ce`, `ppo_loss`,
  `val_loss`): they are called outside regions, where nothing is cached anyway.
- **`S_logits`.** A hit does not refresh it. Inside a region nothing reads `S_logits` after
  `pol_dist` (only `pol_argmax`, which is search-only and uncached, does).

## Why the bits are identical

A miss runs the same `pol_logits` + `pol_softmax` on the same parameters as before; a hit
returns those bits. Rollouts draw the same random numbers, so every later step is the same.
`tests/golden.tcyr` holds this to the bit on x86_64 and aarch64
([ADR 0002](../adr/0002-optimizations-preserve-bits.md)); `tests/units.tcyr` checks each
rule above in isolation, and the mutation run showed each missing bump is caught.

## Cost

Three V×V f64 tables (the policy's, the reference's, the reward model's) and three V-entry
stamp arrays, allocated by the matching `*_init`: 6.3 KB at V = 28, 512 KB at V = 256.
