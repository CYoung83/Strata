# Strata ablation 20261007-224039

Host big-office, Strata checkout `cb8e9be`, engine 0.1.39, production config `/home/chris/src/Strata/strata-iq3_s.json`.
Prompts: 4096, 32768, 128000 tokens x 5 runs per cell, greedy, reasoning off, 256-token cap, identical prompts in every cell, fresh engine per cell, one discarded 32K warm-up.
Noise thresholds used for verdicts: decode 6%, prefill 3% (mean change across prompt sizes, against the first baseline).

**Drift check (baseline end vs start):** 4096: decode -0.4% prefill -1.0%, 32768: decode -0.6% prefill -1.2%, 128000: decode +0.0% prefill -0.9%. Effects smaller than this are not distinguishable from run-to-run drift.
Same-config output repeatability: 15/15 identical greedy outputs (median common prefix 1188 chars). This is the floor for the 'same outputs' column.

## Results (medians; change vs first baseline)

| cell | status | decode 4096 | decode 32768 | decode 128000 | prefill 4096 | prefill 32768 | prefill 128000 | hit @max | PCIe share @max | drafts | W | other CPU s | same outputs | verdict |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| baseline | ok | 186.6 (+0.0%) | 188.2 (+0.0%) | 172.3 (+0.0%) | 5,430 (+0.0%) | 6,998 (+0.0%) | 6,644 (+0.0%) | 0.98 | None | 80% | 564.9 | 24.34 | - |  |
| prefill=auto:32768 | ok | 183.7 (-1.6%) | 175.4 (-6.8%) | 171.7 (-0.3%) | 5,393 (-0.7%) | 7,384 (+5.5%) | 7,300 (+9.9%) | 0.981 | None | 79% | 578.5 | 23.52 | 0/15 | within noise |
| baseline-end | ok | 185.8 (-0.4%) | 187.1 (-0.6%) | 172.3 (+0.0%) | 5,373 (-1.0%) | 6,914 (-1.2%) | 6,584 (-0.9%) | 0.98 | None | 80% | 566.5 | 24.7 | 15/15 | within noise |

Decode ranges (min-max per size) and every per-run number are in `cells/*/result.json`; GPU/CPU samples in `cells/*/telemetry.csv`.

## Combination

Not needed: 0 winning knob(s).

## Suggested production change

No single knob cleared the noise threshold. Production is at or near a local optimum for these knobs.

## Validity checks

- Runs where the engine's prompt count differed from the local tokenizer: 45.
- Max reused (cached) prompt tokens in any measured run: 0 (should be 0).
- Production restore: answering on :8080 after 555s total.
