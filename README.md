# tarka

> तर्क — *logic / reasoning / inference*. The sovereign **reasoning &
> reinforcement-learning** reference in [Cyrius](https://github.com/MacCracken/cyrius),
> the AGNOS "assembly-up" systems language where *everything is i64* and floating
> point is IEEE-754 `f64` bit-patterns driven by `f64_*` builtins.

tarka is the **agency counterpoint to [attn11](https://github.com/MacCracken/attn11)**.
attn11 proves that *gradient-based representation learning* is expressible from the
assembly up — forward pass, hand-derived backprop, Adam, all on raw `f64` arrays
with no BLAS, no libc, no autodiff. **tarka proves the same for *deliberate,
reward-driven problem-solving*** — the policy reasons in multiple steps, and
learning optimizes a *reward / verifier*, not next-token likelihood.

No RL framework. No autodiff. The policy gradient, the advantage estimator, the
value critic, and the search are all written by hand.

## What it owns

Everything past attn11's one basic REINFORCE objective:

- **RL training** — REINFORCE → GRPO / PPO with a value critic + GAE → reward
  models / process-reward models
- **Reasoning** — multi-step chain-of-thought rollouts, self-consistency,
  verifier-guided search (→ tree search / MCTS)

## How it fits the stack

tarka is a **sibling consumer** of the substrate attn11 stands on, not a layer on
top of it:

| Need | Provider |
|------|----------|
| f64 tensors, matmul + gradient | [rosnet](https://github.com/MacCracken/rosnet) |
| deterministic PRNG (rollout sampling, init) | [tyche](https://github.com/MacCracken/tyche) |
| byte / BPE tokenizer | attn11's checkpoint format (shared) |
| stats / epistemology | [pramana](https://github.com/MacCracken/pramana) |

The policy/value networks reassemble from rosnet primitives — the same way attn11's
transformer does — so tarka does **not** depend on attn11-the-binary. See
[ADR 0001](docs/adr/0001-tarka-scope-and-rl-migration.md) for the boundary.

## Build

```sh
cyrius deps                              # resolve stdlib + rosnet/tyche
cyrius build src/main.cyr build/tarka    # compile to a static ELF
cyrius test                              # grad checks + smoke tests
```

## License

GPL-3.0-only
