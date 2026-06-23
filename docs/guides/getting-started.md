# Getting started with tarka

## Build

```sh
cyrius deps                              # resolve dependencies
cyrius build src/main.cyr build/tarka    # compile
cyrius test                              # run [build].test + tests/*.tcyr
```

## Layout

- `src/main.cyr` — entry point + the demo (runs every milestone, prints the acceptance gates).
- `src/test.cyr` — top-level test entry referenced by `cyrius.cyml [build].test`.
- `tests/tarka.tcyr` — primary grad-check suite, 24/24 (`cyrius test` auto-discovers).
- `tests/tarka.bcyr` / `tests/tarka.fcyr` — benchmark / fuzz harnesses (`cyrius bench` / `cyrius fuzz`).

### Modules (named by what they contain — no opaque milestone codes)

| file | contents |
|------|----------|
| `src/rl.cyr` | the minimal rosnet-backed policy + on-policy REINFORCE + Adam + the fail-loud `guard()` |
| `src/actorcritic.cyr` | value critic, GAE, PPO (clipped surrogate), GRPO (group-relative) |
| `src/reward.cyr` | Bradley-Terry learned reward model `r_θ(s,a)` (own embedding, log-sigmoid) |
| `src/parity.cyr` | the parity control task + per-method RL steps + the sample-efficiency benchmark |
| `src/preference.cyr` | preference-trained reward (ORM/PRM) + the process-vs-outcome transmission demo |
| `src/reasoning.cyr` | inference-time deliberation: self-consistency + verifier best-of-N |
| `src/search.cyr` | PRM-guided beam search (the genuine-search task) + matched-compute harness |

The order above is the include order in `main.cyr` and the dependency order (each builds on the
earlier ones). See [`../benchmarks.md`](../benchmarks.md) for what each module demonstrates.

## Adding a feature

1. Edit `src/main.cyr` (or add a new module and `include` it).
2. Add a test case to `tests/tarka.tcyr`.
3. Run `cyrius test`.
4. Bump `VERSION` and add a CHANGELOG entry before tagging.

See [`../adr/template.md`](../adr/template.md) when a non-trivial design choice deserves an ADR.
