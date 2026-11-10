# Contributing to tarka

## Development

1. Install the Cyrius toolchain at the version pinned in `cyrius.cyml`
   (`[package].cyrius`). The pin is the single source of truth — never hardcode
   a version elsewhere.
2. `cyrius deps` — resolve stdlib + the sibling deps (rosnet / tyche / akshara)
   into `lib/`.
3. `cyrius build src/main.cyr build/tarka` — compile the demo (every milestone
   end-to-end; prints `ALL GATES PASS`, ≈ 0.7 s).
4. `cyrius test` — the suites: the finite-difference grad-checks (`tests/tarka.tcyr`),
   the `--pref` ingestion suite (`tests/pref_ingest.tcyr`), the unit suite
   (`tests/units.tcyr`) and the golden gate (`tests/golden.tcyr`), plus the
   `[build].test` stub (`src/test.cyr`).
5. `CYRIUS_DCE=1 cyrius build tests/tarka.fcyr build/fuzz && ./build/fuzz` — the
   seeded `--pref` fuzz harness (4,000 rounds).
6. `CYRIUS_DCE=1 cyrius bench tests/tarka.bcyr` — the benchmarks;
   `scripts/bench-history.sh` appends their medians to `bench-history.csv`.
7. `tests/abort/*.cyr` — each must exit 1 with `tarka: precondition violated`
   (CI builds and runs every one).

There is no Makefile — `cyrius build` / `cyrius test` are the loop. See
[`CLAUDE.md`](CLAUDE.md) for the full development process and
[`docs/development/state.md`](docs/development/state.md) for live state (module
map, dep pins, test count).

## Grad checks are the contract

tarka has no RL framework and no autodiff — the policy gradient, the value/critic
backward, the advantage estimator, and the reward-model backward are all derived
by hand. Hand-derived backprop is the part most likely to be subtly wrong, so
**every backward function MUST be verified against central finite differences in
`tests/tarka.tcyr` before it lands.** A new op without a passing grad check is
incomplete. The gate is per-op max relative error `< 1e-5` (FD-exact paths held
to `1e-9`). Beyond the per-op checks, each method also clears a falsifiable
**acceptance gate** in the demo (see [`docs/benchmarks.md`](docs/benchmarks.md))
— e.g. PPO/GRPO must beat bare REINFORCE on rollouts-to-threshold, a learned
reward must transmit to a fresh policy, beam search must beat best-of-N at
matched compute. A change that touches a method must keep its gate green.

## Numeric rules

- Cyrius has no float type — an `f64` is its bit pattern in an i64. Use the
  `f64_*` builtins, never `+`/`*` on float values.
- Build precise constants from integer ratios or runtime math (e.g.
  `f64_div(f64_from(1), f64_from(10000))` for `1e-4`); long-digit float literals
  mis-parse.
- `var buf[N]` declares N **bytes** inside a function and N **i64 slots** (N × 8
  bytes) at module scope. Forward writes; backward accumulates into parameter
  grads. No allocation inside the training/rollout loop — buffers are
  `*_init`-allocated (or allocated once on first use) and reused.

## Process

- One change at a time. Never bundle unrelated changes in a single PR.
- Test after every change; grad-check after every backward-touching change; keep
  the affected acceptance gate green after any method change.
- Performance claims must include numbers — `before → after` with the bench/gate
  name, from `scripts/bench-history.sh` in one session. A refactor or an
  optimization must keep every bit: `tests/golden.tcyr` must stay green on x86_64
  and aarch64 ([ADR 0002](docs/adr/0002-optimizations-preserve-bits.md)); a change
  that moves a bit is a behavior change, re-records only the cases it touches, and
  says so in the CHANGELOG.
- Breaking changes get a `Breaking` section in [`CHANGELOG.md`](CHANGELOG.md)
  with a migration paragraph. The public 1.x surface is frozen — see
  [`docs/api.md`](docs/api.md); the internal mechanism is explicitly out of the
  freeze.
- Do not commit/push or use `gh` — the maintainer handles git operations.

## License

GPL-3.0-only.
