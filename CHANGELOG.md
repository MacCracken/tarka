# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.1.0] - 2026-06-22

**M0 — Scaffold.** tarka is established as the sovereign reasoning &
reinforcement-learning reference in Cyrius — the agency counterpoint to attn11.
attn11 owns representation / training mechanics (forward, backprop, Adam, SFT
objectives); tarka owns everything reward/verifier-driven and the deliberation it
trains. The two are sibling consumers of `rosnet` (tensors + gradient) and `tyche`
(PRNG), not a chain.

### Added
- Initial project scaffold (`cyrius init`).
- Project identity, goal, and the attn11 boundary in `CLAUDE.md` / `README.md`.
- `cyrius.cyml` deps wired: stdlib + `rosnet` 0.2.0 (CPU) + `tyche` 0.1.1.
- [ADR 0001](docs/adr/0001-tarka-scope-and-rl-migration.md) — tarka scope and the
  attn11 REINFORCE migration boundary (sibling-on-rosnet, not chained-on-attn11).
- Roadmap M1–M4 (REINFORCE migration → GRPO/PPO + critic → reward / process-reward
  models → verifier-guided reasoning).
