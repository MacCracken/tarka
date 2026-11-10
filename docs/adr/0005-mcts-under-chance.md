# 0005 — MCTS under chance, and expert iteration

**Status**: Accepted
**Date**: 2026-11-09

## Context

[ADR 0003](0003-prior-guided-mcts.md) shipped prior-guided PUCT on a deterministic task. There,
a beam given the same prior and value matched it at about half the compute, so the honest result
was that lookahead from the value solved the task and the tree added nothing. ADR 0003 left MCTS's
own advantage to a task with chance or an opponent, where a beam keeps the branches that turned
out lucky. It also left expert iteration (training π on MCTS's visit counts) for later.

The task built for it is **the casino corridor** (`src/chance.cyr`):

- **The corridor**, as in 1.4.0 (K1 → K2 → K3 open GOAL, +4 per step; TRAP +1 per step;
  out-of-order entries cost 60), plus one random action.
- **GAMBLE.** With probability 1/4 it lands on JACKPOT, which opens GOAL at once (a shortcut past
  the three keys). Otherwise it lands on BUST, costing 8.
- **The numbers.** From a neutral token with 16 steps left, the corridor is worth 52. A gamble is
  worth 45 in expectation and 60 when it pays off. The optimal policy never gambles, so the
  optimum, 54.89 over the demo's 40 prompts, is exact and needs no sampling to measure.

**The setting:**
- **The planners' sampler.** The planners get a sampler only, never the outcome probabilities:
  each call is one counted forward. That is the honest stand-in for a simulator, or for an
  environment one can try things in.
- **Real outcomes.** They come from their own stream: a hash of the episode and the step, the same
  for every method, and never seen by a planner.
- **Closed loop.** Every method acts one real step at a time and re-plans from the state it is in.
- **The networks.** The prior is behavior-cloned from imperfect demonstrations: 30% follow the
  optimal plan, 30% gamble until a jackpot and then take GOAL, and 40% take the TRAP, with every
  action randomized with probability 0.2. The value is that policy's critic, trained by policy
  evaluation in the stochastic task.

Measured for three independently trained priors (40 episodes, common random numbers; `@` is
forwards per episode):

| method | seed 7 | seed 11 | seed 13 |
|--------|--------|---------|---------|
| greedy | 32.10 @16 | 30.80 | 28.50 |
| best-of-64 (closed loop) | 39.45 @17.4k | 44.47 | 42.10 |
| beam B16 K4 | 39.52 @14.8k | 45.60 | 46.95 |
| beam B32 K4 | 37.22 @27.0k | 46.42 | 44.42 |
| beam B16 K4, 4 outcome samples per child | 47.29 @53.9k | 47.89 | 49.95 |
| MCTS, chance nodes, S 64 | 45.22 @7.8k | 54.20 | 35.10 |
| **MCTS, chance nodes, S 128** | **52.79 @16.5k** | **54.29** | **51.29** |
| *gamble made a certain bust, same networks:* beam B16 K4 | 53.39 @14.8k | 53.62 | 52.82 |
| *expert iteration (S 128 × 32 episodes):* greedy | 49.50 @16 | 43.79 | 40.50 |
| *expert iteration:* MCTS S 64 | 54.89 @8.6k | 54.89 | 54.89 |

- **Why the beams lose.** The beams and best-of-N gamble 0.3–1.9 times per episode, and MCTS S128
  never does. A beam ranks its candidates by sampled outcomes, so the candidates that hit
  the jackpot rise to the top: the optimizer's curse (Smith & Winkler 2006).
- **Averaging is not enough.** Averaging four samples per child narrows the gap (47–50), but a beam
  entry still carries one realized outcome forward, so everything planned after it inherits that
  luck. Widening the beam does not help: B32 is no better than B16.
- **MCTS keeps its score.** Its chance nodes hold a child per outcome, and Q(gamble) averages
  every outcome the tree has seen.
- **The randomness is the cause.** With the same networks and the gamble made a certain bust, the
  beam recovers to within 2.1 points of the optimum.
- **Expert iteration** is the root's visit distribution at each real step, distilled into π by
  soft-target cross-entropy (`Σ_a p_a·(π − onehot(a)) = π − p`, the existing seed kernel). It
  raises greedy decoding by 12–17 points at 16 forwards. As MCTS's prior it reaches the optimum at
  S 64, half the simulations of S 128 with the cloned prior. With 4× the data (S 256 × 64 episodes),
  the distilled greedy reaches 49.5 / 49.5 / 54.9.
- **The distilled prior fixes the beam too:** 50.39 for seed 7, and 52.6–53.8 with 4× the data.
  The prior stops proposing the gamble.

## Decision

**Ship MCTS with chance nodes, the sampled beams and expert iteration on the casino corridor,
and gate three claims in the demo (seed 7):**

1. **Search under chance.** MCTS S128 reaches 90% of the optimum. It beats best-of-64 and the
   averaged beam, each at matched or higher compute, and it beats the plain beam B16 K4.
2. **The cause is chance.** The plain beam is below 90% of the optimum here, and above it with the
   same networks and the gamble made certain.
3. **Expert iteration.** The distilled π decodes greedily at least 10 points above the cloned one.
   As MCTS's prior, at S64, it scores at least as high as the cloned prior at S128, for no more
   compute.

The demo also prints the distilled prior's beam, so the third claim cannot be read as "only MCTS
benefits". All three claims hold for all three priors measured. The lowest MCTS S128 score, 51.29,
is 93% of the optimum.

## Consequences

- **Positive**
  - tarka has a reference MCTS with chance nodes: PUCT over actions, a child per sampled outcome,
    and subtree reuse that follows the real outcome.
  - Its comparison against beams is a falsifiable one, with the ablation that names the cause.
  - It has expert iteration, built on the existing seed kernel and aggregated backward with no
    new backward math.
  - Coverage: units, a golden case on both targets, five abort probes, fourteen mutants, and three
    bench rows.
- **Negative**
  - **Cost.** MCTS needs a simulator call on every traversal of an action, where 1.4.0's
    deterministic edges were scored once. S 128 costs about the same compute as the plain beam,
    and 1.4× the wall time per episode (the bench rows).
  - **Demo length.** The demo grows by its slowest section yet: about 2.4 s on a loaded host.
- **Neutral**
  - **S 64 is unreliable** with the cloned prior (35–54). The gate uses S 128.
  - **The distillation's value depends on its data.** At S 128 × 32 episodes the distilled greedy
    is 40.5–49.5, and with 4× the data 49.5–54.9. The demo uses the smaller set; § 11 of the
    benchmarks reports both.

## Alternatives considered

- **Give the planners the outcome probabilities.** An expectation-ranked beam (expectimax) would
  then avoid the gamble as MCTS does. Rejected: with an exact model the comparison is no longer
  about sampling, and this task's question is what each search does with samples alone.
- **A two-player game** (minimax backup against an opponent's replies). Not built: it needs a
  second policy and a different backup. It is recorded on the roadmap as the other half of MCTS's
  own advantage.
- **Sparse sampling** (Kearns, Mansour & Ng 2002): a full tree of sampled outcomes, width C per
  action. It is the unbounded cousin of the chance-node tree, exponential in depth. MCTS spends
  its samples adaptively instead.
- **Tune q or L until the beam loses by more.** Rejected, as in ADR 0003. The parameters were
  fixed before the first measurement (q = 1/4, L = 8, chosen so the gamble loses 7 in expectation
  from a neutral token). The ablation, not the margin, is the evidence.
