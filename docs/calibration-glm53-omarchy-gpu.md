# GLM-5.3-Flash NVMe mode on a 5950X / 2-channel DDR4 / x8 / one x2 NVMe box: sim vs measured

Records: `hardware/omarchy-gpu-rtx3090x8-kc3000`, `recipe/glm53-rtx3090x8-{55,90}g-kc3000`,
`runs/omarchy-gpu-glm53-v41-nvme-{55,90}g`. Image `glm53-flash-offload@sha256:99b8926e` (v4.1-nvme), measured 2026-10-08.

## Sim vs measured (decode tok/s, trace G002)

| RAM cap | sim before the run | sim with these records | measured kit C1 / lab C1 | kit C2 / C3 / C4 |
|---|---|---|---|---|
| 55 GiB | 7.0 | 8.5 | 5.2 / 6.1 | 4.85 / 4.71 / not reached |
| 90 GiB | 15.0 | 16.3 | 10.95 / 11.3 | 10.99 / 10.9 / 10.53 |

- "Before the run" used guesses in the hardware record (`bw_qd1_gbps` 1.5, `latency_ms` 0.1) and the reference
  recipe's budget convention (5,087 / 8,781 RAM slots). The committed records drop the unmeasured qd1/latency values
  (`spec.resolve` defaults apply) and size the budget to the engine's reported RAM tier (4,565 / 8,547 slots), so the
  sim is more optimistic: the qd1 guess alone moved 55 GiB C1 from 9.6 to 7.0.
- Lane costs used in both: `zerocopy.per_expert_ms` 0.70 (9,474,048 B / 13.5 GB/s, derived) and `cpu.per_expert_ms`
  0.25 (a guess for 13 lane threads at 3.4 GHz on 2-channel DDR4). `fixed_ms` is copied from G067 (EPYC, x16).

## Derived CPU-lane cost (90 GiB, where the NVMe tier no longer binds)

Per-token ms = fixed 14 + max(gpu lane, cpu lane). The sim at 0.25 gives gpu 29.32 ms, cpu 46.12 ms, and 135.6
CPU-lane experts per token. The CPU-lane terms other than per-expert cost are 46.12 - 135.6 x 0.25 = 12.22 ms
(42 x 0.145 ms handoff plus NVMe landing waits).

- Kit C1 10.95 tok/s = 91.32 ms. MoE = 91.32 - 14 = 77.32 ms. Per expert = (77.32 - 12.22) / 135.6 = **0.480 ms**.
- Lab C1 11.3 tok/s = 88.50 ms. Per expert = (74.50 - 12.22) / 135.6 = 0.459 ms.

So the proposed data point is `cpu.per_expert_ms` ≈ 0.48 (0.46-0.48). This is **derived, not measured**. It is
recorded in `recipe.calibration.derived`, and `lane_overrides` is not changed. It assumes the sim's 0.25-ms lane split,
`fixed_ms` 14 and zero-copy 0.70 all hold here, and none of the three is measured on this box. If the planner
re-splits at a slower CPU lane, the same C1 needs 0.72-0.84 (bisection with `sim`). The probe gives a cross-check:
decode-like windows show PCIe rx at 4.0-5.1 GB/s, which is 38-49 expert-sized reads per token, against 26 in the
0.25 split. The engine already sends more work over PCIe than that split assumes, so the gap is not all CPU lane.
For reference, the EPYC 7443P runs 22 threads at 0.104 ms per expert.

## Utilization without engine counters

The public image writes no live-counters file, so `moetier probe report` returns `time_split` unknown 1.0 and no
per-kind stats. The run records instead carry an `unsplit` kind per bench step, computed with `moetier.probe`'s own
`_stats` and ceilings, with NVMe read from `nvme1n1` because there is no md array. Chokepoints give only the
resource nearest its ceiling.

| step | 55 GiB nearest ceiling | 90 GiB nearest ceiling | 90 GiB CPU groups busy (main / ctl / lane) |
|---|---|---|---|
| lab | NVMe 83 % | NVMe 39 % | 91 / 86 / 80 % |
| kit sweep | NVMe 92 % | PCIe H2D 45 % | 92 / 89 / 83 % |

At 55 GiB, decode is bound by the one x2 drive. At 90 GiB, the host CPU side (CPU lane plus the main and controller
threads) is the likely limit. That is an inference, because the CPU lane has no ceiling: `ddr_practical_gbps` is not
measured, and the per-lane counters need the instrumented engine.
