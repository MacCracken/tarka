# Security Policy

## Reporting

Report vulnerabilities to **cyriusmaccken@gmail.com**. Include reproduction
steps and the tarka version from `VERSION`. Expect an initial response within
one week. Coordinated disclosure is appreciated — do not open a public GitHub
issue with exploit details.

## Threat model

tarka is a single-process, CPU-only reinforcement-learning + reasoning
**reference**: numerical compute (policy/value nets, reward/process-reward
models, preference optimization, self-consistency, best-of-N, PRM-guided beam
search) over rosnet `f64` tensors, the tyche PRNG, and the akshara tokenizer. It
has **no networking** and runs no processes.

It reads **one untrusted input**: `tarka --pref <file>`, a JSONL preference
export (an [ifran](https://github.com/MacCracken/ifran) `pref export`), which may
come from an untrusted curator. Everything else — the demo's corpus, the tasks,
the hyperparameters — is compiled in.

Cyrius performs no bounds checking — an out-of-range `store64` is silent heap
corruption, not a trap — so the relevant questions are raw pointer indexing,
allocation sizing, index/integer arithmetic, `f64` divide-by-zero / NaN,
unchecked resource caps, and RNG determinism.

A realistic attacker is assumed able to:

- write the bytes of a preference file a user passes to `--pref`, and/or
- call the public RL/reasoning API from their own program with attacker-chosen
  dimensions, token ids and hyperparameters, and/or
- supply the build inputs (source / corpus) — i.e. they already control what gets
  compiled, which is equivalent to code execution.

tarka *does not* defend against:

- an attacker with arbitrary code execution in the build or host process
  (trivially owns the whole address space),
- remote / network attacks (tarka has no networking) or side channels (timing,
  cache),
- the *content* of training data: tarka learns whatever preferences or reward it
  is given — a poisoned preference set trains a poisoned policy; that is a
  data-provenance problem, not a vulnerability.

## Attack surfaces & mitigations

| Surface | Mitigation |
|---|---|
| **`--pref` file** | Read capped: a file over 1 MB is refused, not truncated. Each line must be one JSON object (nothing but whitespace around it) whose `kind` is the string `"pair"` / `"unary"`, whose text fields are JSON strings, and whose unary label is the bare number `1` or `-1`; bayan refuses duplicate keys. Strings decode with the full RFC 8259 escape set (`\uXXXX` → UTF-8; an invalid escape refuses the row). The byte vocab is built over the **decoded** bytes, so every field byte has a token id, and a byte without one refuses the row — never an id of −1 in the tables (the 1.2.1 HIGH finding). Caps: 48 (state, action) pairs per sequence, 256 rows per kind. Every refusal is counted by reason and printed. Fuzzed (4,000 seeded rounds per CI run) for: every stored id indexes the policy, no row lost or double-counted, Δ exactly 0 at θ = ref (a canary for a write into the live parameters). |
| **Public-API preconditions** | Fail-loud guards (`guard()` in `rl.cyr`: prints `tarka: precondition violated: <msg>` to stderr and exits 1, on x86_64 and aarch64): `pol_init` dims ≥ 1; every token argument in `0..V-1` (`pol_logits` / `pol_dist`, `ref_logits` / `ref_dist`, `rm_reward`, `pol_logp` / `ref_logp`, the seed kernel); batch and count knobs ≥ 1 (a 0 used to scale the gradient by 1/0 and poison every parameter with NaN); beam `1≤B≤BMAX` / `B*K≤BMAX²`; GRPO `1≤G≤GMAX`; KTO `1≤batch≤KTO_BMAX`; reject-until-different loops `g_V≥2`; and each module's buffers match the current dims (`ac_init` / `rm_init` / `dpo_init` / `search_init` after every `pol_init`). The guards are no-ops when preconditions hold — every result is bit-identical (`tests/golden.tcyr`). CI builds 13 probes (`tests/abort/`) that each violate one and must exit 1 with the message. |
| **Allocation sizing & indexing** | Every `t_alloc` covers the maximum index its consumers reach for the dims its `*_init` recorded; buffers allocated on first use re-size when the rollout length changes. Size products (`g_V*g_D`, `g_V*g_V`, `BMAX*BMAX`, `GMAX*g_T`, `KTO_BMAX*g_T`) cannot overflow i64 at any realistic dimension; `--pref` sizes are bounded by the 1 MB cap. |
| **The distribution memo** | Inside tarka's own loops, per-state outputs are cached; a cached row is used only if filled in the current region's generation at the model's current parameter version, and every parameter writer bumps the version ([architecture note 001](docs/architecture/001-distribution-memo.md)). Out-of-range states are guarded before the cache is indexed. Each invalidation rule is unit-tested and mutation-checked. |
| **Numeric robustness** | Softmax uses max-subtraction so probabilities stay strictly positive, keeping `f64_ln` finite on the reference's own paths. NaN / Inf print as `nan` / `inf` rather than as numbers. Residual (accepted, no live trigger): `f64_ln` of a probability that underflowed to exactly 0. |
| **akshara's loaders** | `secure_read_file` / `read_stdin` / `file_seek` are stubbed side-effect-free (`return -1`) and unreachable from tarka; the `--pref` file is read with the stdlib's `file_read_all`. |
| **PRNG** | Deterministic xorshift via tyche — used for rollout sampling and weight init only, **not** for any security purpose. The sign bit is masked before `% g_V` (non-negative modulo). |
| **Supply chain** | The cyrius toolchain version is pinned in `cyrius.cyml` (the single source of truth); rosnet / tyche / akshara are pinned to exact tags and `cyrius.lock` records every vendored file's hash (`cyrius deps --verify` and `--locked` in CI; `.gitattributes` keeps `lib/` byte-exact). CI's third-party actions are pinned by commit SHA, the CI token is read-only, and checkout does not persist credentials. |

## Audits

- [`docs/audit/2026-11-09-audit.md`](docs/audit/2026-11-09-audit.md) (1.2.2) — the P(-1)
  sweep, including the first audit of the `--pref` path: one HIGH (an escape-decoded
  byte's id −1 indexed the policy's tensors), seven MEDIUM, ten LOW, all fixed;
  17-mutant mutation run, all killed.
- [`docs/audit/2026-06-22-audit.md`](docs/audit/2026-06-22-audit.md) (0.8.0) — the v1.0
  gate: 0 reachable bugs, the first fail-loud guards.

## Residual notes

- The guards depend on a local fail-loud `guard()` because stdlib `assert` only
  *prints* and continues.
- The accepted residual (an `f64_ln` `PMIN` floor) is robustness, not memory safety.
