# M5 Ultra · 256GB replication runbook

**oMLX + GLM-5.3-Flash-oQ4e · Protocol 1.0 · 27 September 2026**

**Question:** How much context, concurrency, and supporting software can this Mac run comfortably? Repeat on a 512GB machine to test whether extra memory makes the intended workload practical.

Allow roughly two hours **after setup and downloads**; long prefills may take longer. With 90 minutes, prioritize A → C → F and label the session “abbreviated.” This document defines a protocol; it does not report new measurements.

## 1. Prepare and record

- Verify the actual chip, GPU core count, and installed memory in System Information. Use AC power, the same power mode, normal ventilation, and a quiet machine. Pause unrelated indexing, downloads, and model servers.
- Install a GLM-5.3-capable oMLX build. Version **0.6.3 explicitly adds support**; match the original tester’s exact build where possible. Use the identical **oQ4e checkpoint**, tokenizer, and revision—not merely a similarly named model. [Release notes](https://github.com/jundot/omlx/releases/tag/v0.6.3).
- For A/B, obtain the matching oMLX source checkout and its working Python environment; `scripts/bench.py` must import that same build. Follow the project’s [installation instructions](https://github.com/jundot/omlx#install). Confirm native kernels are available; do not change kernels or acceleration settings between machines.
- Download everything before the session. Record SSD type, free space, and cache location. Keep at least **30GB free** throughout; this is a conservative protocol reserve, not a vendor limit.
- For C–F, share an identical request pack: prompt files, complete request JSON, model IDs, settings, expected answers, and SHA-256 hashes. Record tokenizer-reported input lengths **including templates/system messages**. Reserve space for output within the configured context limit; never silently truncate.

| Session manifest | Fill in before running |
|---|---|
| Operator / date / session ID | |
| Mac model / chip / CPU & GPU cores / RAM | |
| macOS build / power mode / SSD & free space | |
| oMLX version + commit / Python / MLX / mlx-lm | |
| GLM repository + immutable revision / quantization | |
| Model/config/tokenizer checksums or shared manifest | |
| Native kernels / acceleration / KV format | |
| Context limit / prefill chunk / concurrency limit | |
| RAM & SSD cache limits / eviction or pinning policy | |
| Prompt-pack hashes / template / reasoning mode / sampling | |
| Secondary model, embeddings, reranker, speech & harness versions | |
| Idle memory used / pressure / swap used | |
| Acceptable TTFT / E2E / minimum per-request decode speed | |

Use agreed usability limits before comparing machines. Keep settings identical; document every deviation. Do not infer model residency merely because a service is running.

## 2. Set up measurement

Open **Activity Monitor → Memory**. Record idle values, then sample system memory, memory pressure, and swap every five seconds during each row; retain the highest observed values. Screenshots or a screen recording are fine. Record the model/MLX peak separately.

Replace these three paths with existing locations; run the setup in Terminal:

```sh
OMLX_SRC="/absolute/path/to/matching/omlx-checkout"
BENCH_PY="/absolute/path/to/omlx-environment/bin/python"
MODEL_DIR="/absolute/path/to/GLM-5.3-Flash-oQ4e"
RUN_DIR="$PWD/m5-results-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$RUN_DIR"
sw_vers > "$RUN_DIR/macos.txt"
sysctl -n hw.memsize > "$RUN_DIR/physical-memory-bytes.txt"
git -C "$OMLX_SRC" rev-parse HEAD > "$RUN_DIR/omlx-commit.txt"
"$BENCH_PY" -m pip freeze > "$RUN_DIR/python-packages.txt"
"$BENCH_PY" "$OMLX_SRC/scripts/bench.py" --help
```

Optional five-second swap log in a second Terminal; stop it with **Ctrl-C**:

```sh
while true; do
  date -u '+%Y-%m-%dT%H:%M:%SZ'
  sysctl vm.swapusage
  sleep 5
done | tee swap-samples.txt
```

Move that log into the session folder afterward. Swap already present at idle is not new test activity: report **start, peak, and increase**.

### Stop conditions—apply during warmup too

These are agreed test boundaries, not claims about hardware damage limits:

- **Stop immediately:** red memory pressure, an out-of-memory error, server crash, or an unresponsive desktop.
- **Stop escalation:** yellow pressure for 30 seconds; swap rises by 2GiB from that row’s start; or swap rises at every five-second sample for 30 seconds. Also stop if free disk drops below 30GB.
- Cancel a request after **10 minutes without its first token** or **15 minutes total**; record “timeout,” not “failed to fit.” Longer limits require a separately labeled run.
- Cancel the active test, save evidence, and wait for stable green pressure and swap for 60 seconds. Confirm the server actually stopped processing canceled HTTP requests. Do not advance that context/concurrency branch; smaller healthy workloads may continue.

## 3. Run the sequence

**Token sizes:** 32K = 32,768; 64K = 65,536; 128K = 131,072; **200K = 200,000**; 256K = 262,144. These are input tokens, with output additional.

For A/B, stop the serving instance that has GLM loaded before starting the standalone benchmark, avoiding two copies. Use one discarded warmup and 128 requested output tokens. Run each row once for screening; repeat the 32K baseline and the largest healthy intended workload twice more. Report all trials and their median; label unrepeated rows `n=1`.

Define this helper once. Each invocation writes a separate log:

```sh
bench_row () {
  local row_id="$1"
  shift
  "$BENCH_PY" -u "$OMLX_SRC/scripts/bench.py" "$MODEL_DIR" \
    --gen 128 --warmup 1 "$@" 2>&1 | tee "$RUN_DIR/$row_id.txt"
}
```

The standalone runner loads its own engine. It accepts numeric prompt lengths and batches at the **smallest supplied length**. Give it one length per invocation. Commands below follow the [upstream runner](https://github.com/jundot/omlx/blob/main/scripts/bench.py); verify flags against the pinned checkout.

### A · Single-request context curve

**Run one line at a time; check the stop conditions before the next.**

```sh
bench_row A32  --pp 32768
bench_row A64  --pp 65536
bench_row A128 --pp 131072
bench_row A200 --pp 200000
# Stretch only: enough context capacity for 262144 input + 128 output.
bench_row A256 --pp 262144
```

Mark A256 `unsupported` if the model/runtime limit disallows it. A numeric CLI argument does not establish model support. The conversation’s roughly 35 tok/s and 186.6GB at 32K are prior reported observations, not acceptance targets.

### B · Concurrent requests

Reuse A32/A64/A128 as the single-request reference. Then run each line separately:

```sh
bench_row B32x2  --pp 32768  --batch 2
bench_row B32x4  --pp 32768  --batch 4
bench_row B32x8  --pp 32768  --batch 8
bench_row B64x2  --pp 65536  --batch 2
bench_row B64x4  --pp 65536  --batch 4
bench_row B128x2 --pp 131072 --batch 2
# Only after B128x2 remains healthy:
bench_row B128x4 --pp 131072 --batch 4
```

Each invocation also runs a single-request test before the requested batch. Record the batch row separately. A/B are synthetic throughput tests; their generated answers are not accuracy scores.

### C · Keep the whole stack resident

Finish the standalone process, then start the **normal serving configuration**. Load GLM plus the agreed ~27B secondary model, embeddings, reranker, Whisper, and harness/database/index. Run one small request through each model to load weights; verify residency in logs/process memory. Keep supporting services idle during C.

Using the shared request pack, run **64K ×1 → 128K ×1 → 128K ×2 → 128K ×4**, escalating only while healthy. Use different leading prefixes for simultaneous requests and each cold trial. Check cached-token counts, reload messages, and evictions. Record memory after loading the stack and during each test. If anything unloads, mark “residency failed” even if GLM answers.

Use the local OpenAI-compatible endpoint. The request pack must contain actual text and the server’s exact model ID; this is the shape of each JSON file:

```json
{
  "model": "<exact-server-model-id>",
  "messages": [{"role": "user", "content": "<shared-prompt-text>"}],
  "temperature": 0,
  "top_p": 1,
  "max_tokens": 128,
  "stream": true
}
```

Example single request; adjust the port and add your local authentication header if required:

```sh
curl --fail-with-body -N --max-time 900 \
  http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  --data-binary @requests/C64-1.json \
  > "$RUN_DIR/C64-1.sse"
```

For ×2/×4, use the harness to dispatch all requests together and save **per-request timestamps and token usage**. Record its command/version in the manifest. The curl example saves responses but does **not** measure token TTFT. Leave unavailable metrics `N/A`; do not substitute HTTP-header timing. Compare C against the same server requests with GLM alone as a paired control, not directly against A/B’s engine timing.

### D · Cold, warm, and partial-prefix reuse

Unload supporting models; keep GLM loaded. Enable the agreed cache configuration. Use normal API requests, with 128-token output caps:

1. **Cold:** send the shared 128K request with a new leading trial ID. Confirm zero reused tokens.
2. **Warm:** repeat the exact request immediately; then change only the trailing question and send a third request. Record both TTFTs and reused-token counts.
3. **Partial:** preserve the first 65,536 input tokens of the cold request and replace its suffix with a shared alternate suffix, maintaining the total input budget. Record actual reuse; block boundaries may reduce it.

“Cold” means uncached context with a loaded model, not model-load time. A server restart alone need not clear SSD cache. Keep templates/system messages identical. Label RAM versus SSD hits if observable; otherwise “tier unknown.” oMLX documents both [RAM and SSD cache tiers](https://github.com/jundot/omlx#tiered-kv-cache-hot--cold).

### E · Quality at long context

Run the original synthetic evidence pack at **128K**, then **200K or 256K**, choosing the largest healthy size shared by both machines. Use identical prompts, answer keys, and **2,048 output tokens** on both; keep these rows separate from tg128 timing tests. Fix the same reasoning mode.

Score correct conclusions, correct record-ID citations, preserved alert/resolution history, and invented evidence. Report counts and save full answers. The previous chat referenced `m5_ultra_256_vs_512_benchmark_kit.zip`, but its contents were unavailable for this runbook. **Obtain that same pack from the organizer or mark E “not run—pack missing.”** Do not silently substitute a different corpus.

### F · Twenty-minute real-workload check

Restore C’s resident stack. Launch these four shared jobs together: **128K evidence investigation; 64K code analysis/change plan; 32K anomaly detection; 32K document classification with citations**. Use 2,048 output tokens per job and the same reasoning settings. Wait for all four to finish before launching the next wave; stop starting waves at 20 minutes. Record completed jobs, timeouts, unfinished work, correctness, per-job latency, pressure, swap, and reloads.

Keep supporting models resident. If background embedding/speech work is part of the intended deployment, run a second labeled variant with the same files and fixed launch schedule on both machines. Save that schedule. Missing shared jobs mean F is a local demonstration, not a replicated comparison.

## 4. Record and return results

**Definitions:** TTFT is request submission to first generated token (or first visible content for an API client—label which). PP is prompt-processing tokens/sec; TG is decode tokens/sec; E2E is submission to completion. Include queue time in API latency. For concurrency, retain individual latencies and the slowest request; an average can hide a stalled agent.

Record native aggregate TG as reported and separately compute **all output tokens ÷ batch wall-clock seconds** as end-to-end output throughput. Never present aggregate TG as one agent’s speed. Record actual input/output counts and early stops.

The [native implementation](https://github.com/jundot/omlx/blob/main/omlx/admin/benchmark.py) distinguishes engine timing and MLX peak memory. The stock CLI prints only a subset: obtain additional fields from local instrumentation or mark `N/A`. Its `mem` uses decimal GB and is **not whole-machine RAM**. Normalize bytes to GiB (`bytes ÷ 1,073,741,824`) or explicitly retain original units. Native A/B prompt generation is intended to avoid prefix-cache hits; flag any cached-token warning.

Use one row per trial; duplicate this table as needed:

| Run / trial | Input × requests / actual output | Cache / stack | TTFT mean / max (s) | PP / TG per request / TG aggregate (tok/s) | E2E (s) | MLX peak / system peak (GiB) | Swap start → peak (GiB) / pressure | Correct / usable / notes |
|---|---|---|---|---|---|---|---|---|
| A32 / 1 | 32768 ×1 / | cold / GLM | | | | | | N/A / |
| A64 / 1 | 65536 ×1 / | cold / GLM | | | | | | N/A / |
| A128 / 1 | 131072 ×1 / | cold / GLM | | | | | | N/A / |
| A200 / 1 | 200000 ×1 / | cold / GLM | | | | | | N/A / |
| A256 / 1 | 262144 ×1 / | cold / GLM | | | | | | N/A / |
| B__×__ / 1 | | cold / GLM | | | | | | N/A / |
| C__×__ / 1 | | cold / full | | | | | | N/A / |
| D cold / warm / partial | | / GLM | | | | | | |
| E128 / E200 or E256 | | / GLM | | | | | | |
| F / wave __ | mixed ×4 / | / full | | | | | | |

Return the manifest, completed table, raw logs/responses, memory evidence, prompt hashes, and skipped/stopped rows with reasons. Use `not run`, `unsupported`, `stopped`, `timeout`, or `N/A` explicitly. Keep results local; share the folder directly with the other tester.

## 5. What would make 512GB useful?

Compare identical chip/core configurations where possible. Different GPU cores, bandwidth, macOS, kernels, SSDs, or settings make this a **whole-system comparison**, not an isolated RAM experiment.

| Observation on matched workloads | Interpretation |
|---|---|
| Both remain resident, meet latency limits, and avoid rising swap | 256GB appears sufficient for the tested workload. |
| 256GB swaps/evicts while 512GB stays resident and responsive | Extra RAM provides useful capacity for this stack. |
| 512GB supports longer context or more agents at acceptable latency | Report the additional usable workload, not just peak tokens/sec. |
| Both slow down similarly without memory pressure | More RAM alone has not demonstrated a speed benefit. |

Report four answers: **largest comfortable single context; first pressure boundary; usable concurrent-agent count; whether the complete stack stays resident.** More memory does not inherently improve model accuracy or double decode speed. Without a measured 512GB run, describe its benefit as a hypothesis—do not extrapolate a speedup from the 256GB result.
