# 0001 — tarka scope and the attn11 RL-migration boundary

**Status**: Accepted
**Date**: 2026-06-22

## Context

[attn11](https://github.com/MacCracken/attn11) is the ecosystem's from-scratch
*trained* transformer — the reference that gradient-based learning is expressible
in Cyrius's "everything-is-i64" model (forward, hand-derived backprop, Adam, all on
raw `f64`). At 1.7.0 it shipped a basic **REINFORCE** objective (`--objective rl`,
M17): on-policy policy-gradient toward a deterministic scalar reward, riding the
observation that for a softmax policy `∇log π(a) = −∇CE(a)`, so the policy gradient
*is* the existing softmax-CE backward scaled by the advantage `(R − b)`. attn11's
own changelog flags **"PPO/GRPO + richer rewards"** as the documented follow-on.

That follow-on is a different kind of work from what attn11 is *for*. attn11's
mission is representation / training mechanics. The follow-on is **agency**: a
policy that *reasons* across multiple steps and *learns from a reward / verifier*
rather than from next-token likelihood — value critics, advantage estimators,
reward models, and search. Continuing to grow it inside attn11 would blur attn11's
single, teachable thesis ("learning is expressible from the assembly up") and bolt a
second, larger thesis ("deliberate problem-solving is too") onto the same binary.

The user's framing: attn11 has the tokenizer and the supervised learner; the
**reasoning / RL counterpoint wants its own home.**

## Decision

Stand up **tarka** (तर्क — *reasoning / logic / inference*) as the sovereign
**reinforcement-learning & reasoning reference**, owning **all** RL — REINFORCE
through GRPO/PPO + critic, reward / process-reward models, and verifier-guided
reasoning (CoT, self-consistency, MCTS). attn11 reverts to a **pure SFT/diffusion
training reference**; its `--objective rl` surface **migrates to tarka**.

**Structural choice — sibling, not chain.** tarka does **not** depend on
attn11-the-binary. Both are **consumers of the same substrate**:

- **rosnet** — f64 tensor storage, matmul + gradient (the policy/value networks
  reassemble from these primitives, exactly as attn11's transformer does).
- **tyche** — deterministic PRNG (on-policy rollout sampling; weight init).
- attn11's **byte/BPE tokenizer** is the one genuinely shared artifact (its
  checkpoint format). M1 decides extract-to-lib vs vendor-the-module.

So the migration of REINFORCE is a **re-expression on rosnet**, not a literal lift
of attn11's in-model `D_logits` seeding — the same algorithm, rebuilt on the shared
substrate tarka will grow PPO/critics on anyway.

## Consequences

- **Positive** — each repo keeps one clean thesis; attn11 stops accreting RL surface.
  tarka starts on the substrate (rosnet/tyche) it needs for the *advanced* stack, so
  M2 (PPO/critic) is a continuation, not a re-platforming. "Consumer, not duplicate"
  and "monolithic by design" (separately-releasable repos, coupling at the contract)
  are both honored.
- **Negative** — removing `--objective rl` **de-features a shipped attn11 release
  (1.7.0)**. This is a deliberate, user-confirmed cross-repo change executed as its
  own step (with attn11's version bump on the user's tag), not bundled into tarka's
  scaffold. Re-expressing REINFORCE on rosnet is more work than a copy — but it is
  work M2 would force regardless.
- **Neutral / follow-on** — the **tokenizer-sharing** question is opened, not closed:
  M1 must choose between extracting attn11's tokenizer to a small shared lib (cleanest
  long-term; a second consumer justifies it) and vendoring the module into tarka
  (faster; defers the extraction). GPU execution is explicitly deferred post-1.0,
  mirroring attn11's M18 sequencing.

## Alternatives considered

- **Keep REINFORCE in attn11; tarka owns only PPO-and-up.** Rejected: splits "RL"
  across two repos at an arbitrary line (why is REINFORCE training but PPO isn't?),
  and the user chose a clean "all RL moves" boundary.
- **tarka chains on attn11-the-binary** (consume its transformer directly).
  Rejected: attn11 exposes no `[lib]` policy surface, and chaining would couple
  tarka's release cadence to attn11's. rosnet already *is* the extracted shared
  substrate — siblings-on-rosnet is the lower-coupling shape.
- **Inference-time reasoning only** (no RL training in tarka; CoT/MCTS/verifiers
  only). Rejected: severs reasoning from the RL that trains it; the whole point is to
  *train* deliberate problem-solving, not just decode it.
