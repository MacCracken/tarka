# Security Policy

## Reporting

Report vulnerabilities to **cyriusmaccken@gmail.com**. Include reproduction
steps and the tarka version from `VERSION` (currently **1.0.0**). Expect an
initial response within one week. Coordinated disclosure is appreciated — do not
open a public GitHub issue with exploit details.

## Threat model

tarka is a single-process, CPU-only reinforcement-learning + reasoning
**reference**: numerical compute (policy/value nets, reward/process-reward
models, self-consistency, best-of-N, PRM-guided beam search) over rosnet `f64`
tensors, the tyche PRNG, and the akshara tokenizer. It has **no networking** and
**no runtime parser of untrusted input** — the demo drives a hardcoded,
NUL-terminated corpus, and akshara's file/stdin loader paths are deliberately
stubbed unreachable. It is not a service, an installer, or a deserializer of
attacker-controlled model files.

The real attack surface is therefore **memory safety and numeric robustness**,
not injection / authn / web. Cyrius performs no bounds checking — an out-of-range
`store64` is silent heap corruption, not a trap — so the relevant questions are
raw pointer indexing, allocation sizing, index/integer arithmetic, `f64`
divide-by-zero / NaN, unchecked resource caps, and RNG determinism.

A realistic attacker is assumed able to:

- supply the build inputs (source / corpus) — i.e. they already control what gets
  compiled, which is equivalent to code execution, and/or
- call the public RL/reasoning API from their own program with attacker-chosen
  dimensions and hyperparameters.

tarka *does not* defend against:

- an attacker with arbitrary code execution in the build or host process
  (trivially owns the whole address space),
- remote / network attacks (tarka has no networking) or side channels (timing,
  cache),
- the *content* of training data: tarka learns whatever reward signal it is
  given — a poorly-specified reward is a correctness problem, not a vulnerability.

## Attack surfaces & mitigations

| Surface | Mitigation |
|---|---|
| **Public-API preconditions** (a downstream caller passing out-of-range dims) | The 0.8.0 hardening audit found the unchecked preconditions a future caller could trip into silent heap corruption and added **fail-loud guards** (`guard()` in `rl.cyr`: prints `tarka: precondition violated: <msg>` to stderr and `SYS_EXIT(1)`). 13 guard sites: beam `1≤B≤BMAX` / `B*K≤BMAX²`, GRPO `1≤G≤GMAX`, reject-until-different loops `g_V≥2`, measurement count knobs `N≥1` / `nprompts≥1`. The guards are no-ops when preconditions hold (24/24 grad-checks + all gates byte-identical). |
| **Allocation sizing & indexing** | Every `t_alloc` (policy/critic/reward params + grads + Adam moments, rollout buffers, group buffers, beam buffers) covers the maximum index its consumers reach for the dimensions set by `pol_init`/`*_init`. Size products (`g_V*g_D`, `BMAX*BMAX`, `B*K`, `GMAX*g_T`) cannot overflow i64 at any realistic dimension. CLI/count knobs that feed `f64_div` saturate or are guarded; a `0` knob in a non-guarded training entry yields NaN, not memory unsafety (accepted-low, documented in the audit). |
| **Numeric robustness** | Softmax uses max-subtraction so probabilities stay strictly positive, keeping `f64_ln` finite on the reference's own paths. The accepted-low residual (`f64_ln` of a probability that underflowed to exactly 0 → `-inf`; a `PMIN` floor as a future option) has no live trigger. |
| **Input surface** | akshara's loader/stream paths (`secure_read_file`/`read_stdin`/`file_seek`) are confirmed unreachable from tarka and their stubs are side-effect-free (`return -1`, no buffer writes); `corpus_set` reads a NUL-terminated hardcoded string within bounds. tarka does not load checkpoints or model files at runtime. |
| **PRNG** | Deterministic xorshift/splitmix-style PRNG via tyche — used for rollout sampling and weight init only, **not** for any security purpose. The sign bit is masked before `% g_V` (non-negative modulo); seeding is deterministic so the grad-check suite and acceptance gates are reproducible. |
| **Supply chain** | The cyrius toolchain version is pinned in `cyrius.cyml` (single source of truth, **6.2.37**); sibling deps (rosnet / tyche / akshara) are pinned to exact tags and locked in `cyrius.lock`. The pinned toolchain is the one the audit ran against. |

## Audit

A full security / hardening audit shipped at **v0.8.0** (a v1.0 gate):
[`docs/audit/2026-06-22-audit.md`](docs/audit/2026-06-22-audit.md). A 6-dimension
find→adversarially-verify-reachability workflow (beam-buffer-bounds ·
allocation-sizing · index/integer-arithmetic · numeric-robustness ·
caps/API-misuse · input-surface/RNG) found **0 reachable bugs** — tarka is
memory-safe as called today — and closed the memory-safety-relevant
preconditions with the fail-loud guards above, accepting the remaining low-risk
NaN-on-nonsensical-input items with rationale. **Audit: PASS** on cyrius 6.2.37.

## Residual notes

- The guards depend on a local fail-loud `guard()` because stdlib `assert` only
  *prints* and continues; tarka will adopt the stdlib `panic()` when it lands
  (filed cyrius issue `2026-06-22-stdlib-assert-no-fatal-panic.md`).
- The accepted-low residuals (NaN on a `0` count knob in non-search training
  entries; an `f64_ln` `PMIN` floor) are robustness, not memory-safety, items —
  they could become guards if a caller needs them.
