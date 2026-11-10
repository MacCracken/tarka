# 0004 — `--gpu` as a bit-exact parity backend, not a speedup

**Status**: Accepted
**Date**: 2026-11-09

## Context

The 1.5.x roadmap item was `--gpu` through rosnet's GPU bundle (`dist/rosnet-gpu.cyr`, rosnet
ADR 0001, on mabda), with the CPU path kept as the reference: its golden digests must not move,
and GPU results are checked against it within rosnet's bit-exact and tolerance classes.

Before designing anything, the bundle was measured at tarka's shapes. The host is the reference
dev host: an AMD Ryzen 7 5800H with its integrated Radeon (GFX9), mabda 4.2.0's native path, and
rosnet 1.4.1, whose device suite passes 60/60 there. Times are per call, CPU first:

| op (M × K × N) | CPU | device | |
|----------------|----:|-------:|---|
| all logits, 12 × 16 × 12 (the corridor) | 3.5 µs | 69 µs | 20× slower |
| all logits, 28 × 16 × 28 (the demo; D 8 padded) | 13 µs | 98 µs | 7.4× |
| all logits, 92 × 16 × 92 (the review's `--pref` file) | 92 µs | 459 µs | 5.0× |
| matmul 128 × 64 × 128 | 0.51 ms | 3.43 ms | 6.7× |
| matmul 255 × 64 × 256 | 2.03 ms | 16.6 ms | 8.2× |
| matmul 64 × 256 × 256 | 1.88 ms | 13.4 ms | 7.1× |
| matmul 256 × 64 × 256 | 1.86 ms | declined | M·N past mabda's dispatch cap (65,535) |
| Adam, 476 parameters (the demo's policy) | 12.6 µs | 108 µs | 8.6× |
| Adam, 33,024 parameters | 0.57 ms | 2.81 ms | 4.9× |

Every matmul result was bit-identical to `linear_fwd`, row by row. A dispatch costs about 70 µs
before any work, and rosnet's kernel (one thread per output, serialized k-tiles) runs slower than
the CPU at every size, even at shapes larger than any tarka uses. Two more facts set the scope:

- **`gpu_adam_step` is not bit-exact for tarka.** It computes 1 − β as 1/10 and 1/1000, and the bias
  correction as 1 − β^step with `pow`. tarka's Adam uses `1 − 0.9` and running products, which
  round differently. On tarka's parameters it is a tolerance-class op.
- **`gpu_linear_bwd` is rosnet's tolerance class** (~1e-13). It sums in another order, so a backward
  on the device would move every trained bit.

The measurements were put to the user with two options: defer the GPU line, or ship `--gpu` as
parity only. The user chose to ship it as parity only.

## Decision

**`--gpu` fills the memo regions' logits tables on the device, in rosnet's bit-exact class only,
and its gate is parity, not speed.**

- **What runs on the device.** A memo miss inside a region, with rosnet's `g_gpu` set, computes the
  model's whole table `Y(V×V) = E·W + b` for every state at once. That is one `gpu_matmul_fwd`
  (`dev_table`, `src/rl.cyr`) per parameter version, for each of the policy, the frozen reference
  and the reward model.
- **Padding.** `D` is zero-padded to rosnet's k-tile (`GPU_TK`, 16). Each padded term adds
  (+0)·(+0), which leaves a sum unchanged unless the sum is −0. These sums start from a bias tarka
  never makes −0: `t_zero` writes +0, and an exact cancellation rounds to +0.
- **Declines.** A table the device declines (no device, a shape it does not take, a failed dispatch)
  is computed row by row on the CPU, as without `--gpu`. Each model's table is tried once per
  version.
- **What stays on the CPU.** Everything else: per-token forwards outside a region (rollouts,
  search, MCTS), the critic, every backward, and Adam.
- **The parity gate.**
  - `tests/golden.tcyr` runs every case twice, on the CPU and under `g_gpu`, against the same
    digests. It requires the device to be used when one is present, and every table to be
    declined when none is.
  - `tests/units.tcyr` checks each row against `linear_fwd` (D 8, 16 and 17), a decline (on every
    host), the `--gpu`-off path, and memo equivalence for the three models.
  - CI compares the `--gpu` demo and `--pref` output with the CPU's, byte for byte, apart from the
    `GPU: ` lines. On a runner, which has no device, that exercises the fallback; on a host with a
    device, it is the parity check.
  - Eleven mutants of the device path are all killed.

## Consequences

- **Positive**
  - tarka is a second consumer of rosnet's GPU bundle (rosnet ADR 0001 waits for one). It
    exercises the bundle's bit-exact claim on a different model, and its fallback contract on a
    host with no device.
  - The parity harness is ready for a device, or a batched kernel, that would make the device pay.
  - On the reference host, the demo under `--gpu` dispatches 6,321 tables, declines none, and
    prints the CPU's output.
- **Negative**
  - `--gpu` is slower on the reference host. The demo takes 31% longer (2.81 → 3.67 s, medians
    of three alternating runs on a host loaded by other jobs); its 6,321 dispatches account for the
    gap. In the same bench run, a REINFORCE step took 1.7× longer and a V28 D8 table 8.2× longer.
  - Every build now carries mabda and the GPU bundle, even though nothing in them runs without
    `--gpu`: the binary grows 1.30 → 2.14 MB. `--gpu` also opens the DRM render node (`SECURITY.md`).
- **Neutral**
  - aarch64 builds, and under qemu it declines every table cleanly.
  - mabda is the bundled stdlib leaf of cyrius 6.7.18 (4.2.0), as in attn11, so there is no new
    pinned git dependency.

## Alternatives considered

- **Defer the GPU line.** It was recommended, given the measurements. The user chose parity-only
  instead, which keeps the CPU's bits and records the cost honestly.
- **A tolerance class (`--gpu-tc`)** for the backward and Adam. Rejected for now: it would give up
  byte identity for no speedup. It is recorded on the roadmap with the condition that would justify
  it: a device on which the dispatch pays.
- **Making tarka's Adam match rosnet's scalars** (1 − β as 1/10, the bias correction by `pow`). That
  would be a labeled behavior change, moving every golden case, so that `gpu_adam_step` could be
  bit-exact. It is not worth doing while the device is slower.
- **Per-token forwards on the device.** A 1 × D row is not a dispatch worth making. The
  matched-compute accounting (0.7.0) also counts those forwards one by one.
