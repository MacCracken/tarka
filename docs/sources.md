# Sources

Every algorithm, formula and default constant in tarka, with the source it comes from and
where it lives. tarka implements each from the paper's equations, not from another
implementation; the hand-derived backwards are checked against central finite differences
in `tests/tarka.tcyr`.

## Policy gradient and optimization

| What | Source | Where |
|---|---|---|
| REINFORCE: ∇ log π(a\|s) weighted by the return; the softmax-CE seed `probs − onehot(a)` is −∇ log π | Williams, R. J. (1992). *Simple statistical gradient-following algorithms for connectionist reinforcement learning.* Machine Learning 8, 229–256. | `src/rl.cyr` `rl_backward` |
| Reward baseline (an exponential moving average of past rewards, β = 0.9) to reduce variance without bias | Williams (1992), §5; Sutton, R. S. & Barto, A. G. (2018). *Reinforcement Learning: An Introduction*, 2nd ed., §13.4. | `src/rl.cyr` `RL_BETA`, `rl_train` |
| Adam, with bias correction; β₁ = 0.9, β₂ = 0.999, ε = 1e-8 (the paper's defaults) | Kingma, D. P. & Ba, J. (2015). *Adam: A Method for Stochastic Optimization.* ICLR. arXiv:1412.6980. | `src/rl.cyr` `adam_one`, `ADAM_*` |
| Max-shifted softmax (subtract the max logit before `exp`) | Standard; e.g. Goodfellow, Bengio & Courville (2016), *Deep Learning*, §4.1. | `src/rl.cyr` `softmax_into` |

## Actor-critic

| What | Source | Where |
|---|---|---|
| Generalized advantage estimation: δₜ = rₜ + γV(sₜ₊₁) − V(sₜ), Aₜ = δₜ + γλAₜ₊₁; γ = 0.99, λ = 0.95 | Schulman, J., Moritz, P., Levine, S., Jordan, M. & Abbeel, P. (2016). *High-Dimensional Continuous Control Using Generalized Advantage Estimation.* ICLR. arXiv:1506.02438. | `src/actorcritic.cyr` `gae_advantages`, `GAMMA`, `LAM` |
| PPO clipped surrogate −min(ρA, clip(ρ, 1−ε, 1+ε)A), ε = 0.2; value-loss weight c₁ = 0.5; multi-epoch reuse of one batch | Schulman, J., Wolski, F., Dhariwal, P., Radford, A. & Klimov, O. (2017). *Proximal Policy Optimization Algorithms.* arXiv:1707.06347. | `src/actorcritic.cyr` `ppo_scale`, `ppo_loss`, `CLIP_EPS`, `VAL_COEF` |
| Advantage normalization (mean 0, std 1 per batch) | Common PPO practice; see Andrychowicz, M. et al. (2021). *What Matters in On-Policy Reinforcement Learning?* ICLR. arXiv:2006.05990. | `src/actorcritic.cyr` `gae_normalize` |
| GRPO: group-relative advantage (R − mean) / (std + ε) over G samples of one prompt, no critic, PPO's clip | Shao, Z. et al. (2024). *DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models.* arXiv:2402.03300. | `src/actorcritic.cyr` `grpo_group_adv`, `grpo_coeff` |

## Reward models and preference learning

| What | Source | Where |
|---|---|---|
| Bradley-Terry pairwise preference model P(W ≻ L) = σ(r_W − r_L) | Bradley, R. A. & Terry, M. E. (1952). *Rank Analysis of Incomplete Block Designs: I. The Method of Paired Comparisons.* Biometrika 39(3/4), 324–345. | `src/reward.cyr` `bt_loss` |
| A reward model learned from human preferences, then optimized by RL | Christiano, P. F. et al. (2017). *Deep Reinforcement Learning from Human Preferences.* NeurIPS. arXiv:1706.03741. | `src/reward.cyr`, `src/preference.cyr` |
| RLHF's KL penalty to a frozen reference policy, β·KL(π_θ ‖ π_ref) | Ouyang, L. et al. (2022). *Training Language Models to Follow Instructions with Human Feedback.* NeurIPS. arXiv:2203.02155. | `src/dpo.cyr` `kl_div_at`, `kl_backward_state` |
| Process supervision (a reward per reasoning step) versus outcome supervision (one reward per answer) | Lightman, H. et al. (2024). *Let's Verify Step by Step.* ICLR. arXiv:2305.20050; Uesato, J. et al. (2022). *Solving Math Word Problems with Process- and Outcome-Based Feedback.* arXiv:2211.14275. | `src/preference.cyr` (PRM vs ORM) |
| DPO: Δ = β[(log π_θ − log π_ref)_w − (log π_θ − log π_ref)_l], L = −ln σ(Δ), β = 0.1 | Rafailov, R. et al. (2023). *Direct Preference Optimization: Your Language Model is Secretly a Reward Model.* NeurIPS. arXiv:2305.18290. | `src/dpo.cyr` `dpo_delta`, `dpo_backward_pair` |
| IPO: L = (h − 1/(2β))², h the bare log-ratio margin | Azar, M. G. et al. (2024). *A General Theoretical Paradigm to Understand Learning from Human Preferences.* AISTATS. arXiv:2310.12036. | `src/preference_ext.cyr` `ipo_*` |
| KTO: unpaired desirable / undesirable examples against a detached reference point z; value 1 − σ(β(ρ − z)) | Ethayarajh, K., Xu, W., Muennighoff, N., Jurafsky, D. & Kiela, D. (2024). *KTO: Model Alignment as Prospect Theoretic Optimization.* ICML. arXiv:2402.01306. | `src/preference_ext.cyr` `kto_*` |

The KL-penalty gradient dKL/dlogitₖ = pₖ(fₖ − KL), fₖ = ln pₖ − ln qₖ, is derived in the
header of `src/dpo.cyr`; the IPO and KTO seed signs in the header of
`src/preference_ext.cyr`. All three are finite-difference checked.

## Inference-time reasoning

| What | Source | Where |
|---|---|---|
| Self-consistency: sample N reasoning chains and take the majority answer | Wang, X. et al. (2023). *Self-Consistency Improves Chain of Thought Reasoning in Language Models.* ICLR. arXiv:2203.11171. | `src/reasoning.cyr` `reason_self_consistency` |
| Verifier best-of-N: sample N, keep the one a learned verifier scores highest | Cobbe, K. et al. (2021). *Training Verifiers to Solve Math Word Problems.* arXiv:2110.14168. | `src/reasoning.cyr` `reason_best_of_n`, `src/search.cyr` `srch_best_of_n` |
| Over-optimizing a learned proxy reward (why the true reward, not the verifier score, is the metric) | Gao, L., Schulman, J. & Hilton, J. (2023). *Scaling Laws for Reward Model Overoptimization.* ICML. arXiv:2210.10760. | `src/reasoning.cyr` (metric choice) |
| UCT: UCB1 applied to tree search | Kocsis, L. & Szepesvári, C. (2006). *Bandit Based Monte-Carlo Planning.* ECML. | ADR 0003 (plain UCT, measured and not shipped) |
| PUCT: Q + c·P(a)·√N / (1 + n), priors from a policy, leaves valued by a learned value, the most-visited child committed | Silver, D. et al. (2017). *Mastering the Game of Go without Human Knowledge.* Nature 550, 354–359; Rosin, C. D. (2011). *Multi-armed bandits with episode context.* Annals of Mathematics and Artificial Intelligence 61, 203–230. | `src/mcts.cyr` `mc_puct_pick`, `mc_simulate`, `mc_mcts` |
| Behavior cloning: supervised learning of a policy from demonstrations | Pomerleau, D. A. (1991). *Efficient Training of Artificial Neural Networks for Autonomous Navigation.* Neural Computation 3(1), 88–97. | `src/mcts.cyr` `mc_bc_train` |
| Expert iteration: train the policy on the search's output (considered, not used) | Anthony, T., Tian, Z. & Barber, D. (2017). *Thinking Fast and Slow with Deep Learning and Tree Search.* NeurIPS. arXiv:1705.08439. | ADR 0003 |
| PRM-guided beam search over steps at matched compute | Snell, C., Lee, J., Xu, K. & Kumar, A. (2025). *Scaling LLM Test-Time Compute Optimally can be More Effective than Scaling Model Parameters.* ICLR. arXiv:2408.03314; Yao, S. et al. (2023). *Tree of Thoughts.* NeurIPS. arXiv:2305.10601. | `src/search.cyr` `srch_beam` |

## Numerics and test machinery

| What | Source | Where |
|---|---|---|
| Stable sigmoid (`exp` only of a non-positive argument) and stable softplus −ln σ(Δ) | Standard; e.g. Murphy, K. P. (2022). *Probabilistic Machine Learning: An Introduction*, §2.4. | `src/reward.cyr` `sigma_stable`, `bt_loss` |
| Central finite differences (f(x+h) − f(x−h)) / 2h, error O(h²), h = 1e-4 | Standard; e.g. Nocedal, J. & Wright, S. J. (2006). *Numerical Optimization*, 2nd ed., §8.1. | `tests/tarka.tcyr` |
| FNV-1a hash (64-bit offset basis 0xcbf29ce484222325, prime 0x100000001b3), folded over 64-bit words | Fowler, G., Noll, L. C., Vo, K.-P. & Eastlake, D. *The FNV Non-Cryptographic Hash Algorithm*, IETF draft-eastlake-fnv. | `tests/golden.tcyr` |
| xorshift64 PRNG (dependency) | Marsaglia, G. (2003). *Xorshift RNGs.* Journal of Statistical Software 8(14). | tyche |
