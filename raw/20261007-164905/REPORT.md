# Strata ablation 20261007-164905

Host big-office, Strata checkout `cb8e9be`, engine 0.1.39, production config `/home/chris/src/Strata/strata-iq3_s.json`.
Prompts: 4096, 32768, 128000 tokens x 5 runs per cell, greedy, reasoning off, 256-token cap, identical prompts in every cell, fresh engine per cell, one discarded 32K warm-up.
Noise thresholds used for verdicts: decode 6%, prefill 3% (mean change across prompt sizes, against the first baseline).

**Drift check (baseline end vs start):** 4096: decode -0.4% prefill -1.7%, 32768: decode -0.5% prefill -3.4%, 128000: decode +0.0% prefill -2.2%. Effects smaller than this are not distinguishable from run-to-run drift.
Same-config output repeatability: 15/15 identical greedy outputs (median common prefix 1188 chars). This is the floor for the 'same outputs' column.

## Results (medians; change vs first baseline)

| cell | status | decode 4096 | decode 32768 | decode 128000 | prefill 4096 | prefill 32768 | prefill 128000 | hit @max | PCIe share @max | drafts | W | other CPU s | same outputs | verdict |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| baseline | ok | 185.7 (+0.0%) | 187.0 (+0.0%) | 171.4 (+0.0%) | 5,306 (+0.0%) | 6,494 (+0.0%) | 6,144 (+0.0%) | 0.98 | None | 80% | 448.7 | 25.93 | - |  |
| pcie-frac=0.20 | ok | 178.6 (-3.8%) | 183.3 (-2.0%) | 174.9 (+2.0%) | 5,246 (-1.1%) | 6,357 (-2.1%) | 6,061 (-1.4%) | 0.977 | None | 80% | 444.5 | 26.35 | 1/15 | within noise |
| pcie-frac=0.55 | ok | 180.7 (-2.7%) | 186.2 (-0.4%) | 170.6 (-0.5%) | 5,236 (-1.3%) | 6,320 (-2.7%) | 6,036 (-1.8%) | 0.986 | None | 80% | 445.3 | 26.6 | 0/15 | within noise |
| prefill=auto:32768 | ok | 183.0 (-1.5%) | 174.4 (-6.7%) | 170.8 (-0.4%) | 5,217 (-1.7%) | 6,617 (+1.9%) | 6,522 (+6.1%) | 0.981 | None | 79% | 443.7 | 25.84 | 0/15 | within noise |
| kv-resident=65536 | ok | 176.4 (-5.0%) | 180.9 (-3.3%) | 167.0 (-2.6%) | 5,188 (-2.2%) | 6,283 (-3.3%) | 6,017 (-2.1%) | 0.982 | None | 78% | 445.9 | 26.6 | 0/15 | within noise |
| max-context=262144 | ok | 181.1 (-2.5%) | 182.8 (-2.2%) | 169.2 (-1.3%) | 5,206 (-1.9%) | 6,294 (-3.1%) | 6,025 (-1.9%) | 0.982 | None | 79% | 443.8 | 26.69 | 0/15 | within noise |
| spec-min-p=0.50 | ok | 183.5 (-1.2%) | 182.6 (-2.4%) | 179.4 (+4.7%) | 5,209 (-1.8%) | 6,284 (-3.2%) | 6,018 (-2.1%) | 0.983 | None | 68% | 444.5 | 26.54 | 1/15 | within noise |
| pf-fused=off | ok | 185.2 (-0.3%) | 183.3 (-2.0%) | 172.5 (+0.6%) | 4,745 (-10.6%) | 6,106 (-6.0%) | 5,836 (-5.0%) | 0.977 | None | 80% | 448.1 | 27.49 | 1/15 | worse |
| baseline-end | ok | 184.9 (-0.4%) | 186.0 (-0.5%) | 171.4 (+0.0%) | 5,215 (-1.7%) | 6,270 (-3.4%) | 6,012 (-2.2%) | 0.98 | None | 80% | 447.6 | 26.57 | 15/15 | within noise |

Decode ranges (min-max per size) and every per-run number are in `cells/*/result.json`; GPU/CPU samples in `cells/*/telemetry.csv`.

## Combination

Not needed: 0 winning knob(s).

## Suggested production change

No single knob cleared the noise threshold. Production is at or near a local optimum for these knobs.

## Validity checks

- Runs where the engine's prompt count differed from the local tokenizer: 135.
- Max reused (cached) prompt tokens in any measured run: 0 (should be 0).
- Production restore: answering on :8080 after 1706s total.
