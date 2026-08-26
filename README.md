# Local AI Speed Simulator

**A single-page tool that shows you how fast an AI model would run on your own hardware — before you spend thousands.**

https://eugeneclaw.github.io/local-ai-speed-simulator/

---

## What it does

Pick a workload, pick a model, pick a machine. The simulator models the **full inference pipeline** — prefill (prompt processing), decode (token generation), KV cache memory, and multi-GPU scaling — and tells you:

- How many tokens per second the model would write
- How long until the first word appears (TTFT)
- How long the full reply would take
- Whether the model even fits in memory
- What it would cost in electricity vs. the equivalent cloud API call

The whole thing runs in your browser. No accounts, no tracking, no data leaves your machine.

## How to use

1. **Pick your workload** — quick question, coding agent, deep session, etc.
2. **Pick your harness** — none, light, typical agent (Hermes/Cline/Aider), or heavy custom
3. **Pick a model** — DeepSeek-V4-Flash, GLM-5.2, Qwen 3, Llama 4, etc.
4. **Pick your hardware** — DGX Spark (1-8×), RTX 5090, Mac Studio M5/M3/M2 Ultra, Mac Studio M5 Max, or enter your own
5. **See the result** in the right panel — speed, TTFT, memory fit, monthly cost vs cloud

### Sharing a configuration

Click **SHARE** in the top right. The URL encodes your full config (scenario, model, quant, hardware, count, prompt size, country, cloud service). Send that URL to anyone — they'll see your exact setup.

### Real-time simulator

Step 4 plays back the request in real time at the calculated speed. If it feels slow, that's the point. There's a `skip to end` button if you'd rather see the summary.

## Accuracy

The engine is calibrated against **published real-world benchmarks** with typical accuracy of **±15%**. The current calibration set:

| Setup | Calc | Benchmark |
|---|---|---|
| 1× DGX Spark + DSv4-Flash IQ2 | 18.9 t/s | 19.1 t/s ✓ |
| 1× DGX Spark + DSv4-Flash IQ2 + MTP | ~32 t/s | 23–37 t/s single-request (independent [reproduction on one Spark](https://github.com/emiluzelac/deepseek-v4-flash-0731-on-one-dgx-spark), drafter on) |
| 1× DGX Spark + DSv4-Flash IQ2, 127K ctx | ~1,250 tok/s prefill | ~1,000 tok/s (same reproduction) |
| 2× DGX Spark + DSv4-Flash Q4 + MTP | 49 t/s | ~55 t/s ✓ |
| Mac Studio M3 Ultra 512GB + GLM-5.2 Q4 | 14.6 t/s | ~17 t/s (Q4 GGUF bpw ≈ 4.5) |

**The hard part of LLM inference is the AGENTIC WALL** — for agentic workloads, the prompt (system prompt, AGENTS.md, tool definitions, accumulated context) is 10K–60K tokens before the model writes a word. This dominates total time. The simulator makes this visible: change the workload from "casual chat" to "autonomous agent" and watch prefill time explode. The wall's wording scales with what your hardware actually delivers — on enough compute it correctly reports that the wall has collapsed instead of dramatising a few seconds.

**Speeds are peak; typical is lower.** All displayed token speeds are **peak** (best-effort, unloaded, tuned setup). Real-world runs land roughly **40–70% of that** — contention, thermals, serving overhead and drafter acceptance variance all bite — and the rail shows that "typical ≈ X–Y" band under the headline. The same framing applies to prefill/TTFT: treat times as best-case-at-peak.

**Conversation memory is now MLA-accurate.** DeepSeek/Kimi-style MLA models keep a small per-layer latent KV cache (≈ L × 576 × 2 bytes/token — one shared latent + rope, not heads×dims): DSv4-Flash at 131K context uses ~9 GB — so the full agentic workload genuinely fits one Spark at IQ2, and long contexts stop falsely overflowing. (Kimi K3/K2.6 are modeled as MLA too. Non-MLA models keep the GQA formula.)

**DSv4-Flash fits a single Spark — the simulator now shows it.** The default **Q4** build (155 GB) genuinely needs two Sparks; the well-known one-Spark deployment is the ~88 GiB optimized build (≈ **IQ2**, 97 GB — the same ballpark as an independent [reproduction](https://github.com/emiluzelac/deepseek-v4-flash-0731-on-one-dgx-spark) that measured ~1,000 tok/s prefill and 23–37 tok/s single-request decode on one Spark). The tool now **auto-selects the best quant that fits your machine** (with a note saying so) instead of just showing "doesn't fit", and the fit line tells you which quant would fit when you're on one that doesn't.

**Machine presets.** Each device loads its typical real-world serving config: a **DGX Spark defaults to MTP + vLLM enabled** (the standard stack on it), so comparisons show configured boxes, not bare defaults — the 18.9 t/s row above is the bare no-MTP baseline. Switching device reloads the preset; the Advanced knobs let you tweak from there. One honest caveat: the verified benchmark rows are **measured configs (MTP without vLLM)** — turning vLLM on stacks an extra ~1.5× on top, so the DGX preset's numbers sit at the optimistic end of the real-world range.

**Long-context prefill is degraded to reality.** The raw compute ceiling for prefill is optimistic at agent contexts; the model now degrades prefill steeply past 32K so a 1× Spark reads at ~1,000–1,500 tok/s at 127K (matching the reproduction) — agentic "thinking time" is displayed truthfully (~20 s to ingest a 23K agent prompt).

**Concurrency shares real bandwidth.** Simultaneous users now split decode realistically (aggregate throughput plateaus near ~2× a single request, per batched-serving reproductions), instead of each user keeping nearly full speed. Memory (KV cache) still sets the hard cap on how many fit.

**Multi-unit scaling favours even counts.** Tensor parallelism prefers 2/4/6/8 nodes; odd counts (3/5/7) land between the even tiers — they buy memory pool size and concurrency (expert parallelism), with little per-request speed over the even count below them.

**Quant speed is monotonic (lower = faster).** Decode is memory-bandwidth bound, so smaller quants mean fewer bytes per token and higher tok/s: IQ1 > IQ2 > IQ3 ≈ Q4 > Q5 > Q6 > Q8 on FP4-capable NVIDIA kit (Q4 gets a native-4-bit nudge on Blackwell). Very low quants pay a small dequant tax but still beat larger quants. The quality trade-off is the opposite way round — pick the largest quant that fits, and faster low-quants are a bonus, not a contradiction.

**Mac Studio M5 Ultra / M5 Max** have no independent benchmarks yet — their compute figures follow Apple's launch claims (LLM prompt processing ≈ **4×** an M3 Ultra for the M5 Ultra, **3.9×** an M4 Max for the M5 Max, per the Aug 2026 Mac Studio press release). Treat their results as estimates until measured.

## What's in the box

Just `index.html`. Single file, no build step, no dependencies. Copy it anywhere and open it in a browser.

## Running locally

```bash
git clone https://github.com/EugeneClaw/local-ai-speed-simulator.git
cd local-ai-speed-simulator
open index.html
```

Or just open `index.html` directly — `file://` works fine. (Share button requires a real URL, but otherwise everything works.)

## Hardware coverage

**NVIDIA consumer/prosumer** (memory pools across units):
- DGX Spark — 1× to 8× (128GB each, 273 GB/s, 1000 TFLOPS; TP favours 2/4/6/8 — odd counts add memory pool & concurrency more than per-request speed)
- RTX 5090 / RTX 4090 / RTX 3090 / RTX 6000 Ada — 1× to 8×

**Apple Silicon** (max 4×):
- Mac Studio M5 Ultra (96/256/512 GB) — 80-core GPU with Neural Accelerators, 1.2 TB/s; Apple rates LLM prefill ~4× an M3 Ultra
- Mac Studio M5 Max (36/48/64/128 GB) — 40-core GPU, 614 GB/s; Apple rates LLM prefill ~3.9× an M4 Max
- Mac Studio M3 Ultra (96/256/512 GB)
- Mac Studio M2 Ultra (192 GB)
- Mac Pro M2 Ultra
- MacBook Pro M4 Max
- Mac Mini M4 Pro

M5 Max and M5 Ultra Studios can **cluster up to 4× over Thunderbolt 5 + RDMA** — they share one memory pool and Apple rates 4 nodes ≈ 3× a single Studio. Older Macs run one copy per machine (memory not shared).

**Custom** — enter your own RAM/bandwidth/compute if you don't see your rig.

**Not included**: data-center GPUs (H100/H200/B200/MI300X). This tool is for local inference buyers, not cloud operators.

## Model coverage

15 models including DeepSeek-V4-Flash-0731, GLM-5.2, GLM-5.2-Air, DeepSeek V3, Qwen 3 235B/72B, GPT-OSS 120B, Llama 4 120B, Llama 3.3 70B, Mistral Large, GLM-4.5, Qwen 3.6 27B. Plus custom for anything missing.

**Marked as estimates**: MiniMax M3, Kimi K3, Kimi K2.6 — specs not independently verified.

## Privacy

- No accounts, no cookies, no analytics, no tracking
- No network calls at runtime
- All state in URL hash + localStorage (theme only)
- CSP locks down `connect-src` — no external requests possible

## License

MIT — see [LICENSE](LICENSE).

## Contributing

Issues and PRs welcome on GitHub. The whole point of the tool is that anyone considering local AI inference can answer "is it worth it for me?" — if you have benchmark data that improves the calibration, send it.
