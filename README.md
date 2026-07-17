**English** · [Français](README.fr.md)

# Local LLM Hardware Guide for Compute Autonomy (Mid-2026)
### Mac Studio M5 Ultra 512GB vs Thunderbolt 5 clusters vs NVIDIA vs AMD, for running frontier open models at home

> A practical, sourced decision guide for running frontier open-weight models (DeepSeek, Qwen, GLM, Kimi, MiniMax, Mistral) locally and privately, written mid-June 2026. The core insight that decides everything: **memory bandwidth, not raw compute, is the bottleneck for token generation.**

*Text licensed under CC BY-NC-ND 4.0. All figures are sourced and dated; rumors are flagged as such.*

---

## Abstract

Running frontier language models locally is a trade-off between three variables: **memory capacity** (fitting the model), **memory bandwidth** (speed), and **ecosystem maturity**. Most guides get it wrong by looking at raw compute (TFLOPS, GPU cores). For token generation, what matters is bandwidth.

Key conclusions (mid-2026):

1. **Memory bandwidth is the bottleneck.** On Apple Silicon, doubling GB/s doubles generated tokens per second. GPU cores serve prompt processing and training, not generation speed.
2. **Recent Chinese models are MoE.** Active parameters, not the total, drive speed. This is what makes unified-memory inference viable.
3. **A large unified-memory Mac wins on capacity, not speed.** It fits a 400-880GB model no consumer card can hold, but generates slowly (15-50 tok/s depending on the model).
4. **Thunderbolt 5 clustering gives capacity, not speed.** Plugging two machines together does not double single-session speed. It doubles the memory pool.
5. **The DGX Spark and AMD Strix Halo are 128GB low-bandwidth boxes** (256-273 GB/s). They can neither hold nor quickly run the real 2026 flagships.
6. **Local complements frontier subscriptions, it does not replace them** on the hardest 20% of tasks. It wins on privacy, unlimited volume, and availability.

## 1. Memory bandwidth, the queen of inference

For every generated token, the machine must re-read the model's active weights from memory. Generation speed is therefore bounded by bandwidth:

> tokens/second ≈ (bandwidth × efficiency) / (bytes of active params read per token)

Comparison (figures confirmed as of 2026-06-18 unless noted):

| Hardware | Bandwidth | Usable memory | Note |
|----------|----------:|--------------:|------|
| Apple M5 | 153 GB/s | 32GB | 7B ceiling |
| Apple M5 Pro | 307 GB/s | 64GB | 14-30B sweet spot |
| Apple M5 Max (32-core) | 460 GB/s | 128GB | |
| Apple M4 Max | 546 GB/s | 128GB | |
| Apple M5 Max (40-core) | 614 GB/s | 128GB | best single die 2026 |
| Apple M3 Ultra | 819 GB/s | 96GB (512GB pulled, see §6) | |
| Apple M5 Ultra | >1000 GB/s (leaked, unreleased) | up to 512GB by design (uncertain) | see §6 |
| AMD Ryzen AI Max+ 395 (Strix Halo) | ~256 GB/s (215 measured) | 128GB (96GB as VRAM) | |
| NVIDIA DGX Spark (GB10) | 273 GB/s | 128GB | |
| NVIDIA RTX 6000 Pro Blackwell | 1792 GB/s | 96GB | |
| NVIDIA RTX 5090 | 1792 GB/s | 32GB | speed reference |

The trap: a Mac's unified memory is not *faster*, it is *larger*. An RTX 5090 is twice as fast as an M3 Ultra but caps at 32GB. The Mac holds a giant model no consumer card can fit, but generates slowly. Capacity vs speed.

A detail that matters: the context cache (KV-cache) consumes memory on top of the model weights. On a large model at 32k tokens it can take 100-220GB. On 512GB unified RAM, macOS leaves roughly 70-75% to the GPU.

## 2. MoE: active parameters are what count

2026 frontier open models are Mixture of Experts (MoE): only a fraction of parameters activate per token. This is what makes unified-memory inference viable. A 1-trillion-parameter model with 32B active only re-reads 32B of weights per token, not 1T.

**On Apple Silicon, prefer MoE.** A dense 120B model is slow because all its parameters are active per token.

## 3. Frontier open models, June 2026

| Model | Total / active | Recommended format | Footprint | Fits 512GB? | Speed (~1000 GB/s) |
|-------|----------------|--------------------|----------:|:-----------:|--------------------|
| **GLM-5.2** (Z.ai, Jun 16) | 744B / 40B | Unsloth UD-Q4_K_XL | ~410-450GB | yes (tight) | ~20-28 tok/s |
| **Qwen3.5-397B-A17B** (Feb 16) | 397B / 17B | MLX 8-bit | ~400GB | yes | ~35-55 tok/s |
| **DeepSeek V4-Flash** (Apr 24) | 284B / 13B | MLX 8-bit | ~285GB | yes, roomy | ~45-65 tok/s |
| **DeepSeek V4-Pro** (Apr 24) | 1.6T / 49B | UD-Q4 | ~880GB | no at Q4 (2 nodes) | ~18-24 tok/s |
| **Kimi K2.6** (Moonshot) | 1T / 32B | UD-Q3 / Q4 | Q3 ~420GB / Q4 ~550GB | Q3 yes, Q4 no | ~25-32 tok/s |
| **MiniMax M3** (June) | low active, multimodal | MLX 4-bit | moderate | yes | fast |
| **Mistral Large 3** | MoE (DeepSeek V3 arch) | Q4/Q8 | size-dependent | yes | decent |

GLM-5.2 tops the Artificial Analysis open-model index at time of writing. DeepSeek V4-Pro leads open-weight SWE-bench Verified (~80.6%). All under permissive licenses (MIT for DeepSeek V4).

## 4. Quantization: Unsloth dynamic vs MLX vs full

- **Full (BF16/FP16, 2 bytes/param):** maximum quality, but a 671B model is 1.3TB. Only mid-size models fit at full precision.
- **Unsloth Dynamic GGUF (UD 2.0):** keeps important layers (attention, first and last) at high precision, compresses the rest. About 95% of full quality at half the size. This is how you fit the giants (GLM-5.2 744B, DeepSeek V4-Pro 1.6T, Kimi 1T) on a single machine.
- **MLX (4/8-bit):** native Apple Silicon quants, often faster than GGUF on Mac. Prefer for mid-size models (Qwen3.5-397B, DeepSeek V4-Flash).

## 5. Thunderbolt 5 clustering: capacity, not speed

Myth to correct: "if I plug in two machines, do I drop from 1000 GB/s to what?" Wrong question. **Each machine keeps its internal bandwidth.** What you add is a Thunderbolt 5 link between them:

- TB5 = 80 Gbps = **10 GB/s** (balanced mode, preferred for clustering).
- With RDMA (macOS 26.2): 5-9 microsecond latency, low-latency transfer.
- This 10 GB/s bridge is roughly **100x slower** than internal memory (~1000 GB/s).

Consequence: clustering distributes a model too big for one machine. It **doubles capacity** (memory pool), it does **not double** single interactive-session speed. Measured figures (EXO / MLX, Apple Silicon):

- DeepSeek 671B: **21.1 tok/s on 1 node → 27.8 tok/s on 2 nodes** (~30% gain, not 2x).
- Kimi K2: **5 tok/s on standard link → 25 tok/s with RDMA** (RDMA is the unlock).
- Trillion-parameter models: ~28 tok/s on cluster.

Clustering is only justified to hold a model that does not fit on one machine (e.g. DeepSeek V4-Pro 1.6T at Q4), or to serve multiple users in parallel (batch throughput does scale). For solo interactive use, one well-sized machine beats several poorly used ones. And the cluster is incremental: add a node later.

## 6. The 512GB context and the DRAM crisis

Two facts weighing on any mid-2026 purchase:

- **Apple pulled the 512GB option from the Mac Studio M3 Ultra in March 2026**, due to the DRAM shortage (memory reallocated to AI-accelerator HBM). By May 2026 the current Mac Studio caps at 96GB. Sources: MacRumors, Tom's Hardware, Notebookcheck.
- **Tim Cook announced (WSJ, 2026-06-17) "unavoidable" price increases**, calling memory demand a hundred-year flood. No amount or timeline given.

The Mac Studio M5 Ultra was not announced at WWDC (June 8-12, 2026) and is expected around October 2026. Two leak camps disagree on 512GB:

- **Supply-chain leak camp** (Notebookcheck, TrendForce, BigGo): silicon designed for 36 CPU cores, 84 GPU cores, up to 512GB, >1000 GB/s bandwidth.
- **DRAM reality camp** (Macworld, Gurman): Apple may cap the Ultra at 256GB.

Honest conclusion: the silicon can do 512GB, but the commercial existence of that config, and its price, remain uncertain.

## 7. Which machine to buy, by goal

| Goal | Recommendation | Why |
|------|----------------|-----|
| **Run the open giants (744B-1.6T) locally and privately** | Mac Studio M5 Ultra 512GB (1 unit) | Only consumer box that holds them in one unit |
| **Max speed on 70-235B + dual-use with video/diffusion** | RTX 6000 Pro Blackwell 96GB | 1792 GB/s, mature CUDA, also serves rendering |
| **DeepSeek V4-Pro 1.6T at full Q4** | 2× Mac Studio Ultra TB5 cluster | The only case that needs a 1TB pool |
| **Budget, ~70B models** | AMD Strix Halo 128GB | ~2000 dollars, but low bandwidth |
| **Prototyping / CUDA fine-tuning** | NVIDIA DGX Spark | Excellent prompt processing, CUDA ecosystem |
| **Portable inference, mid tier** | MacBook Pro M5 Max 128GB | 614 GB/s, but ~100GB ceiling |

Why the DGX Spark and AMD are not frontier machines: both are 128GB boxes at ~256-273 GB/s. They can neither hold the giants (128GB ceiling vs 410-880GB needed) nor run them fast. Proof: the DGX Spark on a 120B model generates only **38.5 tok/s**, beaten by three used RTX 3090s. The DGX Spark remains excellent for CUDA development and fine-tuning; it is not a frontier inference machine.

**Guiding principle:** the most efficient hardware is the cheapest one that actually reaches the goal. A 2000-dollar box that cannot hold your target model is not "efficient", it misses the target.

## 8. The economics of compute autonomy

A high-memory workstation at 8000-12000 dollars equals several dozen months of frontier subscriptions. Over time it amortizes. But be honest about the **quality gap**: even the best open models at Q4 stay below closed frontier models on complex reasoning and hard code, and quantization degrades further.

Local **excels** at: absolute privacy (data never leaves the machine), unlimited volume without metering, permanent availability, and routine tasks (summaries, extraction, private RAG, routine code).

Local **fails** to replace frontier on the hardest 20% of tasks.

**Verdict: local complements subscriptions, it does not replace them.** Keep a frontier subscription for the difficulty peak, move volume and confidential work local.

## 9. Caveats and uncertainties

- The Mac Studio M5 Ultra and its 512GB option are unconfirmed (supply-chain rumors, expected ~October 2026).
- M5 Ultra speeds are extrapolated from the measured M5 Max, to be confirmed on real benchmarks.
- Hardware prices are volatile due to the DRAM crisis; re-source at purchase time. Add local taxes.
- Model memory footprints vary with the exact quantization format and context length.

---

## Sources

- Apple M5 Pro/Max specs and bandwidth: Apple Newsroom, AppleInsider, Notebookcheck (March 2026).
- M3 Ultra 512GB removal: MacRumors (2026-03-05), Tom's Hardware, Notebookcheck, Cult of Mac.
- Tim Cook price increases: Wall Street Journal via MacRumors (2026-06-17).
- M5 Ultra leaks: Notebookcheck, TrendForce, BigGo Finance; conservative analysis Macworld.
- Models: GLM-5.2 (Artificial Analysis), DeepSeek V4 (Artificial Analysis, Morph), Qwen3.5-397B-A17B (MarkTechPost), Kimi K2.6 (Artificial Analysis), MiniMax M3 (MindStudio).
- Quantization: Unsloth Dynamic GGUF documentation; MLX.
- Thunderbolt 5 clustering and RDMA: AppleInsider, Jeff Geerling, Creative Strategies, Engadget (macOS 26.2).
- DGX Spark: IntuitionLabs, HotHardware (273 GB/s, 38.5 tok/s gen on 120B, 4699 dollars).
- AMD Ryzen AI Max+ 395: AMD official blog, Notebookcheck (256 GB/s, 96GB VRAM).

## License

Text licensed under [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/). Figures are sourced as of mid-June 2026 and will age; verify current data before purchase decisions. This is an independent technical guide, not financial or purchasing advice.

By [Ismaël Joffroy Chandoutis](https://ismaeljoffroychandoutis.com).
