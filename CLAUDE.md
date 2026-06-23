# tarka — Claude Code Instructions

> **Core rule**: this file is **preferences, process, and procedures** —
> durable rules that change rarely. Volatile state (current version,
> module line counts, supported backends, test counts, dep-gap status,
> consumers) lives in [`docs/development/state.md`](docs/development/state.md).
> Do not inline state here.

## Project Identity

**tarka** (तर्क — *logic / reasoning / inference*, the Nyaya-school word for
disciplined reasoning) — the sovereign **reasoning & reinforcement-learning**
reference in Cyrius. It is the **agency counterpoint to
[attn11](https://github.com/MacCracken/attn11)**: where attn11 proves that
*gradient-based representation learning* is expressible in an "assembly-up,
everything-is-i64" systems language, tarka proves the same for *deliberate,
reward-driven problem-solving*. Pairs with the existing `pramana` (epistemology /
stats) lib in the Sanskrit reasoning lane.

- **Type**: Binary
- **License**: GPL-3.0-only
- **Language**: Cyrius (toolchain pinned in `cyrius.cyml [package].cyrius`)
- **Version**: `VERSION` at the project root is the source of truth — do not inline the number here
- **Standards**: [First-Party Standards](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/first-party-standards.md) · [First-Party Documentation](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/first-party-documentation.md)

## Goal

tarka owns **all reinforcement learning and reasoning** in the AGNOS ML stack —
everything past attn11's single basic-REINFORCE objective. That means the
reward/verifier-driven training family (REINFORCE → GRPO/PPO with a value critic
and GAE → reward models / process-reward models) **and** the deliberation it
trains (multi-step chain-of-thought rollouts, self-consistency, verifier-guided
search up to tree search / MCTS). All of it hand-written on raw `f64` arrays — no
RL framework, no autodiff — the same sovereign discipline as attn11.

**Boundary with attn11** (see [ADR 0001](docs/adr/0001-tarka-scope-and-rl-migration.md)):
attn11 is the **representation / training-mechanics** reference (forward,
hand-derived backprop, Adam, SFT objectives — AR / diffusion / ternary). tarka is
the **agency / deliberation** reference. They are **sibling consumers of `rosnet`**
(the f64 tensor + gradient substrate) and `tyche` (PRNG), not a chain — tarka's
policy reassembles from rosnet primitives rather than depending on attn11-the-binary.
The byte/BPE tokenizer is shared from attn11's checkpoint format. attn11's
`--objective rl` (REINFORCE, shipped 1.7.0) **migrates here** at M1.

## Current State

> Volatile state lives in [`docs/development/state.md`](docs/development/state.md) —
> current version, surface area, in-flight work, consumers, dep gaps.
> Refreshed every release.

This file (`CLAUDE.md`) is durable rules.

## Scaffolding

Project was scaffolded with `cyrius init` (greenfield) or `cyrius port` (Rust → Cyrius migration). **Do not manually create project structure** — use the tools. If a tool is missing something, fix the tool.

## Quick Start

```sh
cyrius deps                          # resolve sibling deps
cyrius build src/main.cyr build/tarka
cyrius test                          # run [build].test + tests/*.tcyr
```

## Key Principles

- **Correctness over cleverness** — if it's wrong, the bugs own you
- Test after every change, not after the feature is "done"
- ONE change at a time — never bundle unrelated changes
- Research before implementation — check vidya / existing patterns
- Build with `cyrius build`, not raw `cat file | cc5` — the manifest auto-resolves deps and prepends includes
- Source files only need project includes — stdlib / external deps auto-resolve from `cyrius.cyml`
- Every buffer declaration is a contract: `var buf[N]` = N **bytes**, not N entries
- `&&` / `||` short-circuit; mixed expressions require explicit parens

## Rules (Hard Constraints)

- **Do not commit or push** — the user handles all git operations
- **Never use `gh` CLI** — use `curl` to the GitHub API if needed
- Do not skip tests before claiming changes work
- Do not use `sys_system()` with unsanitized input — command injection
- Do not trust external data (file / network / args) without validation
- Do not modify `lib/` files (vendored stdlib / dep symlinks)
- Do not hardcode toolchain versions in CI YAML — `cyrius = "X.Y.Z"` in `cyrius.cyml` is the source of truth

## Documentation

- [`docs/adr/`](docs/adr/) — Architecture Decision Records (*why X over Y?*)
- [`docs/architecture/`](docs/architecture/) — Non-obvious constraints (*what's true about the code?*)
- [`docs/guides/`](docs/guides/) — Task-oriented how-tos
- [`docs/examples/`](docs/examples/) — Runnable examples
- [`docs/development/state.md`](docs/development/state.md) — Live state snapshot
- [`docs/development/roadmap.md`](docs/development/roadmap.md) — Milestones through v1.0

## Process

1. **Work phase** — features, roadmap items, bug fixes
2. **Build check** — `cyrius build`
3. **Test + benchmark additions** for new code
4. **Internal review** — performance, memory, correctness, edge cases
5. **Documentation** — update CHANGELOG, `docs/development/state.md`, any ADR the change earned
6. **Version sync** — `VERSION`, `cyrius.cyml`, CHANGELOG header

