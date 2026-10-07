# Strata ablation 20261007-225828

Host big-office, Strata checkout `cb8e9be`, engine 0.1.39, production config `/home/chris/src/Strata/strata-iq3_s.json`.
Prompts: 4096, 32768, 128000 tokens x 15 runs per cell, greedy, reasoning off, 256-token cap, identical prompts in every cell, fresh engine per cell, one discarded 32K warm-up.
Noise thresholds used for verdicts: decode 6%, prefill 3% (mean change across prompt sizes, against the first baseline).

**Drift check (baseline end vs start):** 4096: decode -0.7% prefill -0.3%, 32768: decode -0.7% prefill -0.8%, 128000: decode -0.3% prefill -0.0%. Effects smaller than this are not distinguishable from run-to-run drift.
Same-config output repeatability: 45/45 identical greedy outputs (median common prefix 1182 chars). This is the floor for the 'same outputs' column.

## Results (medians; change vs first baseline)

| cell | status | decode 4096 | decode 32768 | decode 128000 | prefill 4096 | prefill 32768 | prefill 128000 | hit @max | PCIe share @max | drafts | W | other CPU s | same outputs | verdict |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| baseline | ok | 188.1 (+0.0%) | 187.0 (+0.0%) | 179.3 (+0.0%) | 5,547 (+0.0%) | 6,991 (+0.0%) | 6,660 (+0.0%) | 0.982 | None | 80% | 574.3 | 0.14 | - |  |
| prefill=auto:32768 | ok | 187.2 (-0.5%) | 184.8 (-1.2%) | 177.8 (-0.8%) | 5,523 (-0.4%) | 7,409 (+6.0%) | 7,284 (+9.4%) | 0.98 | None | 78% | 575.2 | 0.06 | 1/45 | WIN prefill |
| baseline-end | ok | 186.7 (-0.7%) | 185.7 (-0.7%) | 178.7 (-0.3%) | 5,530 (-0.3%) | 6,936 (-0.8%) | 6,658 (-0.0%) | 0.982 | None | 80% | 576.9 | 0.08 | 45/45 | within noise |

Decode ranges (min-max per size) and every per-run number are in `cells/*/result.json`; GPU/CPU samples in `cells/*/telemetry.csv`.

## Combination

Not needed: 1 winning knob(s).

## Suggested production change

Knob values that cleared the noise threshold on their own (check the 'same outputs' column before adopting anything that changes answers, e.g. pf-fused):
- `prefill` -> `auto:32768` (WIN prefill)

## Validity checks

- Runs where the engine's prompt count differed from the local tokenizer: 135.
- Max reused (cached) prompt tokens in any measured run: 0 (should be 0).
- Production restore: answering on :8080 after 1397s total.
