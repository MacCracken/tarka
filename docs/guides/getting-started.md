# Getting started with tarka

## Build

```sh
cyrius deps                              # resolve dependencies
cyrius build src/main.cyr build/tarka    # compile
cyrius test                              # run [build].test + tests/*.tcyr
```

## Layout

- `src/main.cyr` — entry point + the demo (runs every milestone, prints the acceptance gates) and the CLI.
- `src/test.cyr` — top-level test entry referenced by `cyrius.cyml [build].test` (a stub).
- `tests/tarka.tcyr` — the finite-difference grad-check suite (59).
- `tests/pref_ingest.tcyr` — `--pref` ingestion (117); `tests/units.tcyr` — the memo's and the aggregated backward's rules, the task helpers, the corridor and the tree's books, the device path, the casino corridor (118).
- `tests/golden.tcyr` — the bit-for-bit gate: 16 digests per target, each checked on the CPU and again under `--gpu` ([ADR 0002](../adr/0002-optimizations-preserve-bits.md), [ADR 0004](../adr/0004-gpu-parity-backend.md)).
- `tests/abort/*.cyr` — guard probes (each must exit 1); `tests/fixtures/` — the `--pref` fixture.
- `tests/tarka.bcyr` / `tests/tarka.fcyr` — benchmarks / the seeded `--pref` fuzz harness.

### Modules (named by what they contain — no opaque milestone codes)

| file | contents |
|------|----------|
| `src/rl.cyr` | the minimal rosnet-backed policy + on-policy REINFORCE + Adam + the fail-loud `guard()` + the memo and its `--gpu` table fill (`dev_table`) |
| `src/actorcritic.cyr` | value critic, GAE, PPO (clipped surrogate), GRPO (group-relative) |
| `src/reward.cyr` | Bradley-Terry learned reward model `r_θ(s,a)` (own embedding, log-sigmoid) |
| `src/parity.cyr` | the parity control task + per-method RL steps + the sample-efficiency benchmark |
| `src/preference.cyr` | preference-trained reward (ORM/PRM) + the process-vs-outcome transmission demo |
| `src/reasoning.cyr` | inference-time deliberation: self-consistency + verifier best-of-N |
| `src/search.cyr` | PRM-guided beam search (the genuine-search task) + matched-compute harness |
| `src/mcts.cyr` | prior-guided MCTS (PUCT) on the corridor task, against beams given the same prior + value |
| `src/chance.cyr` | MCTS with chance nodes on the casino corridor, against sampled beams; expert iteration |
| `src/dpo.cyr` | the frozen reference policy, DPO, the RLHF KL-to-reference penalty |
| `src/preference_ext.cyr` | IPO (margin-regularized) and KTO (unpaired) on the same reference |
| `src/report.cyr` | the printers shared by the demo and the `--pref` driver |
| `src/pref_ingest.cyr` | `tarka --pref`: the validating JSONL ingestion + DPO / IPO / KTO driver |

Each module builds on the ones it lists in its header ("Include AFTER …"); `main.cyr` and the
suites include them in dependency order. See [`../benchmarks.md`](../benchmarks.md) for what each
module demonstrates.

## Adding a feature

1. Edit `src/main.cyr` (or add a new module and `include` it in `main.cyr` and the suites).
2. Add a test: a grad-check in `tests/tarka.tcyr` for any new backward, a case in
   `tests/units.tcyr` otherwise, and a `tests/golden.tcyr` case for a new trainer.
3. Run `cyrius test` (and the fuzz harness when touching `--pref`).
4. Bump `VERSION` and add a CHANGELOG entry before tagging.

See [`../adr/template.md`](../adr/template.md) when a non-trivial design choice deserves an ADR.
