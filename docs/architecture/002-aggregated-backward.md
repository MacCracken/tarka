# 002 — The aggregated backward: when `G_*` is complete

The policy's logits for state x are E[x]·W + b, so the backward of a seed `s` (the dlogits
`(probs − onehot(a))·X` at one token) is linear in `s`: `G_W += E[x]ᵀ s`, `G_b += s`,
`G_E[x] += s·Wᵀ`. All tokens of one state share E[x], so the sum over a batch of those
tokens' backwards equals **one** backward of the sum of their seeds. A REINFORCE step at the
demo's shape (batch 16 × T 24 = 384 tokens, V 28) needs at most 28 `linear_bwd` calls, not 384.
1.3.0 does this inside tarka's own training loops (`src/rl.cyr` `pol_bwd_seeded`,
`pol_bwd_flush`, `pol_bwd_discard`).

## The invariant

**Inside a memo region, `G_E` / `G_W` / `G_b` do not hold the policy gradient until
`pol_bwd_flush` runs.**

- **Deferral is scoped to memo regions** ([note 001](001-distribution-memo.md)), the same
  scope as the distribution cache: only tarka's own loops. Outside a region `pol_bwd_seeded`
  back-propagates at once, so a caller that reads `G_*` right after a backward — the
  finite-difference grad-checks, a consumer's own loop — sees exactly the per-token result.
- **Inside a region**, each seed is added to a pending row for its state (`PB_rows`); the
  first time a state is touched it is appended to `PB_list`, so the flush order is
  first-touch order and deterministic.
- **The flush points are the gradient's consumers.** `pol_scale_grads` and `pol_adam_step`
  flush before they read `G_*`; every training step in tarka passes through one of them
  before its gradient is used. `pol_zero_grads` discards anything pending (it starts the
  gradient over).
- **Nothing may be left pending when the outermost region closes.** `memo_end` aborts (a
  guard) if it is, because that would be a gradient silently lost. `tests/abort/deferred_lost.cyr`
  checks the guard; the mutation run showed a missing flush is caught.
- **Only the policy's backward is aggregated.** The critic's (`critic_backward`) and the
  reward model's (`rm_accum_seed`) still run per token; the critic adds into the shared
  `G_E` directly, before the policy's deferred rows are flushed into it.

## What it changes

The policy gradient is the same sum taken in a different order, so its low bits differ from
the per-token backward: `tests/units.tcyr` holds the two to 1e-12 relative on a rollout that
revisits states, and every hand-derived backward is still FD-checked through the immediate
path. Training results move by that rounding — a labeled behavior change under
[ADR 0002](../adr/0002-optimizations-preserve-bits.md); 1.3.0 re-recorded 13 of the 14 golden
cases (all but `search`, which trains only the reward model). Every demo number still prints
the same at two decimals, and every gate holds.

## Cost

Three tables per `pol_init`: V×V pending rows, a V-entry list and a V-entry mark array
(6,720 bytes at V 28, 528 KB at V 256). A pending add is one V-wide `t_add_into` per token, against a
D×V `linear_bwd` it replaces.
