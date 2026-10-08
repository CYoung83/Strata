# RTX 5090 / Ryzen 9 9950X3D, IQ3_S: one-setting ablation, 450 W vs 600 W power limit, background CPU load

Measured on 2026-10-07 by [CYoung83](https://github.com/CYoung83) on the same box as
[`2026-10-03-dev-5090`](../2026-10-03-dev-5090/). Three runs of one harness
([`strata_ablate.py`](strata_ablate.py)), each against the box's production config:

| Run | Folder | GPU power limit | Background CPU load | Cells | Prompts per size |
| --- | --- | --- | --- | --- | ---: |
| A | [`data/A-450w-one-knob`](data/A-450w-one-knob/) | 450 W | 4-thread embedding server running | baseline, 7 one-setting changes, baseline | 5 |
| B | [`data/B-600w`](data/B-600w/) | 600 W | same | baseline, `--prefill auto:32768`, baseline | 5 |
| C | [`data/C-600w-embedder-paused`](data/C-600w-embedder-paused/) | 600 W | embedding server paused | same as B | 15 |

Main limitation: engine 0.1.39, two patch releases behind the current one, and synthetic prompts only, with no
quality or recall check.

## Hardware and software

- NVIDIA GeForce RTX 5090 32 GB, driver 610.57.04. PCIe Gen 5 x16 maximum link, and Gen 5 x16 in every telemetry
  sample taken during a request.
- AMD Ryzen 9 9950X3D (16 cores, 32 threads, AVX-512). 89 GiB RAM visible to the OS. Model storage: not recorded.
- Ubuntu, kernel 7.0.0-38-generic.
- Strata engine 0.1.39 (`/metrics` self-report), local source checkout `cb8e9be`. That checkout predates the
  2026-10-06 history rewrite, so the hash is not on the current `main`. CUDA toolkit version for this build: not
  recorded.
- Power limit: 450 W (run A) and 600 W (runs B, C), set with `nvidia-smi -pl`. Clocks not fixed.
- Background: a llama-server running Qwen3-Embedding-8B-Q8_0 on the CPU (`-t 4`, `--device none`), at about 400%
  CPU and 17 GB RSS throughout runs A and B. Processes other than Strata used 75-102 CPU-seconds during each 128K
  request in A and B. In run C the server and its client were stopped with `SIGSTOP` for the whole run, and the
  same figure was 0.17-0.55 CPU-seconds. The Hermes agent and its tools (Obsidian, tailscaled) were idle.

## Model and configuration

- `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF` IQ3_S (2 shards plus the PLE shard); revision not recorded. Pack
  `packs/iq3_s`, expert profile `data/expert-profile.bin`, MTP `mtp/rt`, vision encoder on the GPU (no request
  carried an image).
- Calibration not run; experimental speed projection not enabled.

Production config, the baseline for every cell (exact files: `data/*/cells/*/config.json`):

```text
--pack packs/iq3_s --native ...IQ3_S-00001-of-00002.gguf --ple-gguf ...IQ3_S-00002-of-00002.gguf
--expert-profile data/expert-profile.bin --expert-cache auto --prefill auto --spec 4 --mtp mtp/rt
--max-context 250000 --kv int8 --kv-resident 32768 --pcie-frac 0.00 --spec-min-p 0.70 --vision
--vram-reserve-mib 700 --conversation-cache-mib 6144 --conversation-cache-slots 3
--conversation-cache-min-free-mib 2048
env: STRATA_PF_FUSED=1
```

At ready the engine reported `spec=6`, `mtp_max=4`, `lookup=3`, 15 expert-pool workers, 11,751 expert-cache slots
(22,829 MiB) and 283 MiB VRAM free. Exceptions: 11,545 slots with `--kv-resident 65536`; 11,741 slots and 281 MiB
free with `--max-context 262144`.

## Method

- Each cell starts a fresh `serve/server.py --engine strata` on a private port, with the production config plus
  one change. The engine is stopped between cells. Load took 18 s (30 s for the first cell of run A).
- Prompts are built once per run with the checkout's tokenizer and chat template: a synthetic Python module and a
  request to explain it, with a unique nonce line per prompt. Every cell of a run sends the same prompt texts.
  Runs B and C send the same first 5 prompts per size as run A, so cells can be compared prompt by prompt across
  runs.
- Each cell sends one discarded 32K warm-up request (64-token cap), then the measured requests one at a time:
  `temperature` 0, `reasoning_effort` `none`, `max_tokens` 256, streaming. Every measured request generated 256
  tokens and reused 0 prompt tokens.
- Prompt tok/s = prompt tokens read / `prompt_ms`; decode tok/s = `engine_generated` / `decode_ms`. Both come from
  the engine's per-request record in `/metrics`. TTFT is measured at the client, to the first streamed content
  token, and includes queueing (none here, one request at a time).
- Telemetry: `nvidia-smi` every 2 s (board power, SM clock, utilization, PCIe link) and `/proc` CPU time of every
  other process per request.
- The engine's prompt count was 40 tokens below the harness's tokenizer-plus-template count on every request
  (4,052 / 32,692 / 127,954 against 4,092 / 32,732 / 127,994). The size labels below (4K, 32K, 128K) use the
  engine's counts.

## Results

Per-run data for every cell, size and request is in `data/*/cells/*/result.json`.

| Run | Configuration | Prompt tokens | Reused | Generated | Runs | Prompt tok/s median (range) | Decode tok/s median (range) | TTFT s median (range) | Drafts accepted |
| --- | --- | ---: | ---: | ---: | ---: | --- | --- | --- | ---: |
| A | baseline (start) | 4,052 | 0 | 256 | 5 | 5,306 (4,635–5,551) | 185.7 (164.3–190.6) | 0.78 (0.74–0.89) | 79.8% |
| A | baseline (start) | 32,692 | 0 | 256 | 5 | 6,494 (6,423–6,565) | 187.0 (177.7–195.0) | 5.07 (5.02–5.13) | 80.7% |
| A | baseline (start) | 127,954 | 0 | 256 | 5 | 6,144 (6,095–6,267) | 171.4 (169.8–178.6) | 20.95 (20.54–21.11) | 79.6% |
| A | `--pcie-frac 0.20` | 4,052 | 0 | 256 | 5 | 5,246 (4,526–5,503) | 178.6 (166.1–193.3) | 0.79 (0.75–0.91) | 78.6% |
| A | `--pcie-frac 0.20` | 32,692 | 0 | 256 | 5 | 6,357 (6,277–6,433) | 183.3 (178.1–191.5) | 5.18 (5.12–5.25) | 79.1% |
| A | `--pcie-frac 0.20` | 127,954 | 0 | 256 | 5 | 6,061 (6,048–6,157) | 174.9 (169.4–182.1) | 21.23 (20.90–21.28) | 83.0% |
| A | `--pcie-frac 0.55` | 4,052 | 0 | 256 | 5 | 5,236 (4,529–5,485) | 180.7 (167.3–185.7) | 0.79 (0.75–0.91) | 79.1% |
| A | `--pcie-frac 0.55` | 32,692 | 0 | 256 | 5 | 6,320 (6,248–6,375) | 186.2 (174.6–192.9) | 5.21 (5.17–5.27) | 81.2% |
| A | `--pcie-frac 0.55` | 127,954 | 0 | 256 | 5 | 6,036 (6,023–6,128) | 170.6 (169.1–181.8) | 21.32 (21.00–21.37) | 81.2% |
| A | `--prefill auto:32768` | 4,052 | 0 | 256 | 5 | 5,217 (4,596–5,492) | 183.0 (163.3–187.7) | 0.79 (0.75–0.90) | 78.5% |
| A | `--prefill auto:32768` | 32,692 | 0 | 256 | 5 | 6,617 (6,560–6,689) | 174.4 (171.9–187.9) | 4.98 (4.92–5.02) | 78.1% |
| A | `--prefill auto:32768` | 127,954 | 0 | 256 | 5 | 6,522 (6,513–6,624) | 170.8 (167.6–182.8) | 19.74 (19.44–19.77) | 81.8% |
| A | `--kv-resident 65536` | 4,052 | 0 | 256 | 5 | 5,188 (4,600–5,446) | 176.4 (167.1–184.0) | 0.79 (0.76–0.89) | 77.9% |
| A | `--kv-resident 65536` | 32,692 | 0 | 256 | 5 | 6,283 (6,238–6,352) | 180.9 (173.0–187.1) | 5.24 (5.18–5.28) | 79.9% |
| A | `--kv-resident 65536` | 127,954 | 0 | 256 | 5 | 6,017 (6,013–6,109) | 167.0 (161.3–174.7) | 21.39 (21.07–21.40) | 77.3% |
| A | `--max-context 262144` | 4,052 | 0 | 256 | 5 | 5,206 (4,512–5,460) | 181.1 (163.7–188.7) | 0.79 (0.76–0.91) | 79.0% |
| A | `--max-context 262144` | 32,692 | 0 | 256 | 5 | 6,294 (6,228–6,352) | 182.8 (177.6–188.8) | 5.23 (5.18–5.29) | 78.7% |
| A | `--max-context 262144` | 127,954 | 0 | 256 | 5 | 6,025 (6,009–6,108) | 169.2 (164.0–182.6) | 21.36 (21.07–21.41) | 80.7% |
| A | `--spec-min-p 0.50` | 4,052 | 0 | 256 | 5 | 5,209 (4,544–5,438) | 183.5 (168.4–191.2) | 0.79 (0.76–0.91) | 66.1% |
| A | `--spec-min-p 0.50` | 32,692 | 0 | 256 | 5 | 6,284 (6,245–6,353) | 182.6 (178.4–196.5) | 5.24 (5.18–5.27) | 70.1% |
| A | `--spec-min-p 0.50` | 127,954 | 0 | 256 | 5 | 6,018 (6,010–6,111) | 179.4 (168.1–187.6) | 21.38 (21.06–21.41) | 69.0% |
| A | `STRATA_PF_FUSED` unset | 4,052 | 0 | 256 | 5 | 4,745 (4,206–4,955) | 185.2 (166.0–189.2) | 0.87 (0.83–0.98) | 78.4% |
| A | `STRATA_PF_FUSED` unset | 32,692 | 0 | 256 | 5 | 6,106 (6,051–6,176) | 183.3 (171.3–192.5) | 5.39 (5.33–5.44) | 79.3% |
| A | `STRATA_PF_FUSED` unset | 127,954 | 0 | 256 | 5 | 5,836 (5,826–5,924) | 172.5 (164.4–177.0) | 22.05 (21.72–22.08) | 81.9% |
| A | baseline (end) | 4,052 | 0 | 256 | 5 | 5,215 (4,208–5,474) | 184.9 (163.9–189.3) | 0.79 (0.75–0.98) | 79.8% |
| A | baseline (end) | 32,692 | 0 | 256 | 5 | 6,270 (6,220–6,349) | 186.0 (176.4–193.5) | 5.25 (5.19–5.29) | 80.7% |
| A | baseline (end) | 127,954 | 0 | 256 | 5 | 6,012 (6,000–6,104) | 171.4 (169.6–178.0) | 21.40 (21.09–21.45) | 79.9% |
| B | baseline (start) | 4,052 | 0 | 256 | 5 | 5,430 (4,705–5,702) | 186.6 (165.2–191.6) | 0.76 (0.72–0.88) | 79.8% |
| B | baseline (start) | 32,692 | 0 | 256 | 5 | 6,998 (6,932–7,074) | 188.2 (178.5–195.9) | 4.71 (4.66–4.75) | 80.7% |
| B | baseline (start) | 127,954 | 0 | 256 | 5 | 6,644 (6,626–6,770) | 172.3 (170.3–178.8) | 19.38 (19.02–19.43) | 79.9% |
| B | `--prefill auto:32768` | 4,052 | 0 | 256 | 5 | 5,393 (4,723–5,670) | 183.7 (163.7–188.6) | 0.77 (0.73–0.87) | 78.5% |
| B | `--prefill auto:32768` | 32,692 | 0 | 256 | 5 | 7,384 (7,324–7,482) | 175.4 (173.2–189.3) | 4.46 (4.41–4.50) | 78.1% |
| B | `--prefill auto:32768` | 127,954 | 0 | 256 | 5 | 7,300 (7,291–7,434) | 171.7 (169.1–184.2) | 17.65 (17.33–17.67) | 81.8% |
| B | baseline (end) | 4,052 | 0 | 256 | 5 | 5,373 (4,677–5,678) | 185.8 (164.2–190.4) | 0.77 (0.73–0.88) | 79.8% |
| B | baseline (end) | 32,692 | 0 | 256 | 5 | 6,914 (6,826–6,978) | 187.1 (177.4–195.0) | 4.77 (4.72–4.83) | 80.7% |
| B | baseline (end) | 127,954 | 0 | 256 | 5 | 6,584 (6,572–6,685) | 172.3 (170.2–179.0) | 19.55 (19.26–19.59) | 79.9% |
| C | baseline (start) | 4,052 | 0 | 256 | 15 | 5,547 (4,739–5,799) | 188.1 (169.7–195.3) | 0.74 (0.71–0.87) | 78.9% |
| C | baseline (start) | 32,692 | 0 | 256 | 15 | 6,991 (6,916–7,086) | 187.0 (176.3–197.1) | 4.71 (4.65–4.76) | 79.8% |
| C | baseline (start) | 127,954 | 0 | 256 | 15 | 6,660 (6,593–6,723) | 179.3 (169.4–191.9) | 19.33 (19.15–19.52) | 80.8% |
| C | `--prefill auto:32768` | 4,052 | 0 | 256 | 15 | 5,523 (4,825–5,761) | 187.2 (169.0–191.3) | 0.75 (0.72–0.85) | 77.2% |
| C | `--prefill auto:32768` | 32,692 | 0 | 256 | 15 | 7,409 (7,296–7,474) | 184.8 (177.6–193.4) | 4.45 (4.41–4.52) | 78.6% |
| C | `--prefill auto:32768` | 127,954 | 0 | 256 | 15 | 7,284 (7,276–7,398) | 177.8 (166.6–187.8) | 17.68 (17.41–17.70) | 79.6% |
| C | baseline (end) | 4,052 | 0 | 256 | 15 | 5,530 (4,713–5,765) | 186.7 (168.6–193.6) | 0.75 (0.72–0.87) | 78.9% |
| C | baseline (end) | 32,692 | 0 | 256 | 15 | 6,936 (6,844–7,007) | 185.7 (175.6–195.9) | 4.75 (4.71–4.82) | 79.8% |
| C | baseline (end) | 127,954 | 0 | 256 | 15 | 6,658 (6,581–6,689) | 178.7 (168.8–191.3) | 19.34 (19.25–19.56) | 80.8% |

In every cell, the first 4K request after the warm-up has the lowest prompt and decode rate (expert-cache hit rate
about 92.7%, against 97-98% for the rest). It sets the low end of each 4K range.

### Baseline repeatability

In each run, the baseline at the start and the baseline at the end produced identical greedy text and identical
accepted-draft counts on every prompt (15/15, 15/15, 45/45). Per prompt, their decode rates differed by a median of
0.5% (maximum 1.1%). Within one cell, decode spans about 164-197 tok/s across prompts, and that spread repeats
prompt by prompt between the two baselines.

### Run A: one setting changed at a time, 450 W

Medians from the table, change relative to the mean of the two baselines (decode 185.3 / 186.5 / 171.4 tok/s,
prompt 5,261 / 6,382 / 6,078 tok/s at 4K / 32K / 128K). "Same text" counts measured prompts whose greedy output
matched the baseline.

| Change | Decode 4K / 32K / 128K | Prompt 4K / 32K / 128K | Same text |
| --- | --- | --- | ---: |
| `--pcie-frac 0.20` | -3.6% / -1.7% / +2.0% | -0.3% / -0.4% / -0.3% | 1/15 |
| `--pcie-frac 0.55` | -2.5% / -0.2% / -0.5% | -0.5% / -1.0% / -0.7% | 0/15 |
| `--prefill auto:32768` | -1.2% / -6.5% / -0.4% | -0.8% / +3.7% / +7.3% | 0/15 |
| `--kv-resident 65536` | -4.8% / -3.0% / -2.6% | -1.4% / -1.6% / -1.0% | 0/15 |
| `--max-context 262144` | -2.3% / -2.0% / -1.3% | -1.0% / -1.4% / -0.9% | 0/15 |
| `--spec-min-p 0.50` | -1.0% / -2.1% / +4.7% | -1.0% / -1.5% / -1.0% | 1/15 |
| `STRATA_PF_FUSED` unset | -0.1% / -1.7% / +0.6% | -9.8% / -4.3% / -4.0% | 1/15 |

- Every change altered the greedy output on at least 14 of 15 prompts. The decode column therefore mixes speed
  with differences in the generated text and its draft acceptance. The prompt column is not affected by this.
- Draft acceptance with `--spec-min-p 0.50` was 66.1-70.1%, against 79.6-80.7% in the baselines (0.70).
- `docs/DETAILS.md` lists the IQ3 packs as "about even" with `STRATA_PF_FUSED=1`. Here, unsetting it lowered the IQ3_S
  prompt rate by 9.8% / 4.3% / 4.0%.
- Expert-cache hit rate was 97.3-98.6% (median per size) in every cell of all three runs.
- `/metrics` reported `pcie_share` as null on every request, including the `--pcie-frac 0.20` and `0.55` cells.

### Power limit: 450 W (run A) vs 600 W (run B)

Paired by prompt: the same 5 prompts per size in the same configuration. Greedy output was identical on all 45
pairs. Mean of per-prompt ratios, 600 W over 450 W:

| Configuration pair | Decode 4K / 32K / 128K | Prompt 4K / 32K / 128K |
| --- | --- | --- |
| baseline (start) | +0.5% / +0.5% / +0.4% | +2.2% / +8.0% / +8.4% |
| baseline (end) | +0.5% / +0.6% / +0.6% | +5.0% / +9.8% / +9.4% |
| `--prefill auto:32768` | +0.5% / +0.7% / +0.7% | +3.2% / +11.8% / +12.0% |

Mean board power during 32K and 128K requests in the baseline cells: 428-451 W at the 450 W limit, 520-578 W at
the 600 W limit.

### `--prefill auto` vs `auto:32768`, 15 prompts per size (run C)

| | 4K | 32K | 128K |
| --- | ---: | ---: | ---: |
| Decode median, `auto` (mean of both baselines) | 187.4 | 186.3 | 179.0 |
| Decode median, `auto:32768` | 187.2 (-0.1%) | 184.8 (-0.8%) | 177.8 (-0.7%) |
| Decode mean, `auto` / `auto:32768` | 187.3 / 186.4 | 186.9 / 183.5 | 180.0 / 177.3 |
| Prompt median, `auto` | 5,539 | 6,964 | 6,659 |
| Prompt median, `auto:32768` | 5,523 (-0.3%) | 7,409 (+6.4%) | 7,284 (+9.4%) |

Decode standard deviation across the 15 prompts was 4.5-7.6 tok/s per cell. With 5 prompts per size (runs A and
B), 32K decode with `auto:32768` was -6.5% both times. The prompt-rate change at 32K and 128K had the same sign and
a similar size in all three runs (+3.7% / +7.3% in A, +6.2% / +10.4% in B, +6.4% / +9.4% in C). The engine log
shows the 32K prompts read in 8,192-token chunks with `auto` on this box.

### Background CPU load: embedding server running (run B) vs paused (run C), 600 W

Paired by prompt (run C's first 5 prompts per size are run B's 5), over the three matching cells:

| Size | Same greedy text | Decode per-prompt change | Prompt per-prompt change |
| --- | ---: | --- | --- |
| 4K | 15/15 | +0.9% to +3.2%, mean +1.6% | +0.0% to +2.2%, mean +0.6% |
| 32K | 0/15 | -9.0% to +6.7%, mean +1.1% | -1.1% to +0.9%, mean -0.3% |
| 128K | 0/15 | -9.4% to +12.3%, mean +3.3% | -0.7% to +1.2%, mean -0.1% |

At 4K the outputs and draft counts matched pair for pair, so the 4K decode change is a like-for-like speed
difference. At 32K and 128K, pausing the CPU process changed the greedy output of every prompt and the accepted
drafts per prompt. The power-limit change above changed none. The 32K and 128K decode figures here therefore
include differences in the generated text. `docs/DETAILS.md` lists `--adapt-swaps 0` and `--pcie-frac 0` among
the switches for byte-identical repeats. These runs used `--pcie-frac 0.00` and the default adaptation settings.

## Correctness and limitations

- Speed only: no needle, task or quality check. Prompts are synthetic code-explanation requests. Greedy, reasoning
  off, 256 output tokens.
- Engine 0.1.39. Releases since then (0.1.40.x) were not measured.
- One machine. Run A and B cells have 5 prompts per size, so decode comparisons between configurations that change
  the output rest on 5 prompts each.
- Runs A and B had the CPU embedding server running. Run C differs from B in that alone, and from A in both power
  limit and background load.
- `STRATA_PF_FUSED=1` is part of the baseline. Its effect on output was not evaluated beyond the "same text" count
  above. Per `docs/DETAILS.md`, `STRATA_PF_FUSED=0` keeps the previous prompt kernels with byte-identical answers.

## Files

- `data/<run>/cells/<NN-cell>/`
  - `config.json`: the exact engine config for the cell.
  - `result.json`: per request: `prompt_tokens`, `reused`, `prompt_ms`, `decode_ms`, `generated`, `prefill_tok_s`,
    `decode_tok_s`, `ttft_s` (s, client), `hit_rate` (expert cache), `pcie_share`, `drafts_offered` /
    `drafts_accepted`, `finish`, telemetry means over the request window (`power_w_mean` W, `sm_mhz_mean`,
    `gpu_util_mean` %, `pcie_gen_min`, `pcie_width_min`, `cpu_busy_mean` % whole machine), `other_cpu_s_total`
    and `other_cpu_top` (CPU-seconds of other processes during the request), `out_sha` and the first 4,000
    characters of `text` (the generated output). Plus per-size `summary` medians and the engine's startup
    `engine_info`.
  - `telemetry.csv`: 2 s samples: unix time, board power W, SM MHz, temperature C, GPU utilization %, VRAM used
    MiB, PCIe gen, PCIe width, whole-machine CPU busy %.
  - `server.out`: the server's console output for the cell.
- `data/<run>/hw.txt`: `nvidia-smi`, `lscpu`, `free -g` and `uname -a` at the start of the run.
- `data/<run>/production.json`: the production server's command line and config, from which the cells are derived.
- `strata_ablate.py`: the harness. `--dry-run` builds the prompts and prints the plan. A normal run stops the
  production server, runs the cells and restarts the production server. `--only`, `--runs` and `--targets` select
  cells, prompts per size and sizes.
