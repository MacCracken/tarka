# 0003 — Prior-guided MCTS, gated against beams with the same networks

**Status**: Accepted
**Date**: 2026-11-09

## Context

0.7.0 scoped Monte Carlo tree search out: its search task is deterministic with one good
action per step, where verifier-guided beam search is the right tool. The roadmap carried
MCTS forward with one condition — a task where value backup matters, measured against beam
on the matched-compute harness (`g_fwd`: one unit per policy forward, critic forward or
evaluator call; nothing cached).

The task built for it is **the corridor** (`src/mcts.cyr`): three zero-reward keys
K1 → K2 → K3, in order, open a GOAL worth +4 per step; a TRAP pays +1 per step from anywhere;
entering a key or GOAL out of order costs 60. Over T = 16 the optimum is 52–64 by prompt
(mean 54.00 over the demo's 40 prompts); the TRAP path is 16. Four formulations were
measured before one was shipped (40 prompts each; "@n" = forwards per prompt):

1. **A learned Bradley-Terry verifier, plain UCT.** The verifier (trained from sampled BT
   labels, which identify the reward's scale) gave the corridor's zero-reward steps partial
   credit (r(K2, K3) ≈ +1.16 against a true 0). With nothing to back up, an exhaustive beam
   found the optimum at B = 16 (@2.8k); UCT needed ≈ @22k. That run also showed a shortcut in
   the first reward design (a −2 entry penalty made "enter GOAL directly" optimal), fixed
   with the penalty of 60.
2. **The same verifier, penalty 60.** BT cannot represent a −60 cell: once it always loses,
   its labels saturate and training stops near −4. Every verifier-guided method took the
   shortcut the verifier underpriced; UCT, the best optimizer of the verifier, did worst on
   the true reward (−225) — reward over-optimization (Gao et al. 2023), reproduced.
3. **The exact reward as the evaluator, plain UCT.** The exhaustive beam's structural failure
   is real: it holds TRAP-level returns (≈ 24) until B = 512 (@82k). But UCT without a prior
   did worse (6–22 at @76k): with 12-way branching it spreads its visits over nine near-tied
   siblings and never expands three levels down the corridor.
4. **The exact evaluator, a learned prior and value (shipped).**
   - On-policy PPO on r* is captured by the TRAP within 50 steps (π(TRAP) 0.99, the corridor
     ≈ 0: exploring it risks −60), so it gives no usable prior.
   - A prior **behavior-cloned from imperfect demonstrations** (30% follow the optimal plan,
     70% the TRAP, every action randomized with probability 0.2) does: π(K1) ≈ 0.1–0.25,
     π(K2 | K1) ≈ 0.55, π(GOAL | K3) ≈ 0.6. Its critic, trained by policy evaluation, ranks the
     corridor's states above the TRAP's.
   - PUCT (Silver et al. 2017) with that prior and the critic as leaf value reaches the
     optimum (54.00 @1.1k for the demo's prior). Rollout leaves were worse: the cloned policy
     imitates the demonstrations' noise, so its rollouts keep hitting the 60-point penalties.
   - Across three independently trained priors and two prompt sets, PUCT (S 512, c 1) scored
     48.6–55.1; **a beam given the same prior and value** (K = 4 children from π ranked by
     cumulative r* + V) scored 53.1–55.0 at about half the forwards (@≈540); a value-only beam
     solved the task outright for two of the three priors at B = 4.

## Decision

**Ship PUCT with the behavior-cloned prior and the policy-evaluation critic on the corridor,
with the exact reward as the evaluator, and gate it on what the measurements support:** MCTS
reaches 85% of the optimum and beats policy greedy, best-of-N and the two beams without
lookahead (myopic exhaustive, prior-only) at matched-or-higher compute for them; and **a beam
given the same prior and value also reaches 85%, at no more than MCTS's compute**. The demo
prints that beam beside MCTS, so the result cannot be read as "MCTS wins". The 85% floor holds
for all three priors measured (the lowest, 48.6, is 90% of 54).

## Consequences

- **Positive** — tarka has a reference PUCT (priors from π, a learned value, subtree reuse,
  most-visited commits), behavior cloning, and policy evaluation, each tested (units, a golden
  case, abort probes, eight mutants) and measured honestly against the strongest baseline the
  same networks allow.
- **Negative** — the demo's MCTS section makes a weaker claim than 0.7.0's beam section: on a
  deterministic task, lookahead from a learned value is what matters, and a value-ranked beam
  gets it at less compute. In wall time the gap is wider than in forwards (14 ms per prompt for
  MCTS against 0.1 ms for the beam): with cheap model calls, PUCT's bookkeeping dominates.
- **Neutral** — PUCT's result varies with the prior (one of three needs S ≥ 512 to pass 85%).
  The demo uses the first prior trained (seed 7); `docs/benchmarks.md` § 9 reports all three.

## Alternatives considered

- **Keep the learned verifier.** Rejected: experiments 1 and 2 measure the verifier's errors,
  not the search. The exact evaluator is the honest stand-in for a checker or simulator; the
  reasoning section already demonstrates learned verifiers.
- **Tune the task until MCTS beats every beam.** Rejected: that would be choosing the result.
- **Expert iteration / AlphaZero's training loop** (Anthony et al. 2017; Silver et al. 2017):
  train π on MCTS's visit counts, with root exploration noise, from a uniform start. Not tried:
  it needs a first search that finds the corridor, which experiment 3 showed plain UCT cannot
  do at these budgets.
- **A stochastic or adversarial task**, where beam search is structurally weak (it keeps lucky
  branches or ignores the opponent). Not tried here; the natural place to show MCTS's own
  advantage, recorded on the roadmap.
