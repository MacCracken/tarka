# tarka — Public API (frozen at v1.0.0)

> Satisfies the v1.0 criterion *"Public RL/reasoning API frozen — every exported symbol
> documented and tested."*

## Freeze policy

The **stable public API** below is frozen for the 1.x series: signatures and semantics will not
change without a major version bump. Each symbol is exercised by the demo (`src/main.cyr`), the
grad-check suite (`tests/tarka.tcyr`, 24/24), and/or the quickstart consumer
(`examples/quickstart.cyr`) — noted per row.

**Not frozen (internal):** the per-element optimizer internals (`adam_one`, `*_adam_step`),
gradient accumulators / zeroers (`*_zero_grads`, `*_backward`, `gae_*`, `ppo_snapshot`,
`grpo_group_adv`/`grpo_ratio`/`grpo_coeff`), reward-model internals (`rm_logits`,
`sigma_stable`, `bt_loss`, `rm_accum_seed`, `*_fill`, `orm_bufs`, `rm_sample_to`), the
matched-compute counting wrappers (`cnt_*`), and module-private helpers (`par_rw`, `RUNI`,
`reason_oracle`/`reason_answer`, `target_wrong`, `pol_argmax`). These are the reference's
*mechanism*; they are readable and grad-checked but may change between minor versions.

Convention: dimensions/seed are set once by `pol_init` (+ the `*_init` allocators); `g_datalen`
selects the corpus (`0` = synthetic vocab). All buffers are allocated once in `*_init` and
reused — see [`benchmarks.md`](benchmarks.md) § Performance notes.

## Policy + REINFORCE — `src/rl.cyr`

| symbol | purpose | tested by |
|--------|---------|-----------|
| `pol_init(V, D, T, target, seed)` | allocate + seed the policy (vocab, embed dim, rollout len, reward token) | demo, suite, quickstart |
| `rl_train(steps, batch, lr, log_every)` | on-policy REINFORCE training | demo, quickstart |
| `rl_eval(nroll)` → f64 | mean reward over `nroll` rollouts | demo, quickstart |
| `rl_rollout(prompt)` / `rl_prompt()` → i64 / `rl_reward()` → f64 | sample a rollout / draw a prompt / score the current rollout | demo, suite |
| `pol_logits(x)` / `pol_softmax()` / `pol_sample()` → i64 | per-step distribution / normalize / sample (custom decoding) | demo, suite |
| `guard(cond, msg)` | fail-loud precondition guard (prints + `SYS_EXIT`) — the hardening primitive | demo (every guarded entry) |
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
| `reason_verifier_gate(n)` → i64 | 1 if the verifier ranks correct above wrong | demo |
| `REASON_V/D/T/TGT/DECOY/STEPS()` | reasoning-task configuration | demo |

## Verifier-guided search — `src/search.cyr`

| symbol | purpose | tested by |
|--------|---------|-----------|
| `search_setup()` / `search_init()` | set up the search task + verifier / allocate the beam buffers | demo |
| `srch_greedy(nprompts, base_seed)` → i64 | greedy-decode baseline, mean #TARGET ×100 | demo |
| `srch_best_of_n(N, nprompts, base_seed)` → i64 | best-of-N (verifier-selected), mean #TARGET ×100 | demo |
| `srch_beam(B, K, nprompts, base_seed)` → i64 | PRM-guided beam search; `1 ≤ B ≤ BMAX()`, `B*K ≤ BMAX()²` | demo |
| `target_streak(st, ac)` → f64 | the metric (#TARGET in a rollout) | demo |
| `TARGET()` `SR_V/D/T()` `BMAX()` / `g_fwd` | task config / max beam width / forward-pass counter | demo |

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
