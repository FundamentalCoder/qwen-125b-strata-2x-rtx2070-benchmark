# Qwen3.8-Flash-Next 125B on Two RTX 2070s — Strata Benchmark

**Community benchmark / single-user results — November 2026**

I ran **Qwen3.8-Flash-Next 125B** locally in **Strata** using two older **NVIDIA RTX 2070 8 GB** GPUs (16 GB VRAM total across two separate cards). This is an informal report of the actual hardware, software, throughput and long-context retrieval tests—not a standardized benchmark suite.

> **Highlights:** **250,000 tokens configured**, **225,354 prompt tokens successfully tested**, **~35–39 output tokens/s** in measured generation runs, with approximately **37 GiB RAM available** in the later idle/loaded snapshot.

## Hardware

| Component | Configuration |
|---|---|
| Host/virtualization | Proxmox VM, GPU passthrough |
| Guest OS | Ubuntu 24.04.5 LTS |
| CPU | AMD EPYC 7452 (32-core processor model) |
| Inference GPUs | **2 × GeForce RTX 2070, 8 GB each** |
| System memory | **78 GiB RAM** during final tests (earlier benchmarks used ~70 GiB) |
| Swap | 8 GiB |
| Storage | Samsung 970 EVO Plus, 1 TB NVMe SSD passed through to VM |
| Driver | NVIDIA **595.99.02** (driver reports CUDA **13.2**; CUDA driver compatibility, not necessarily the toolkit version) |

## Software and model

| Component | Configuration |
|---|---|
| Inference engine | [Niko1221/Strata](https://github.com/Niko1221/Strata), prebuilt CUDA engine **v0.1.59** at initial setup; initial repository checkout at commit `13eedf5` |
| Model | [Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next): **125B** main-model parameters, about **6B active per token** |
| GGUF distribution | [alesha-pro / abliterated GSQ-RCO Strata GGUF](https://huggingface.co/alesha-pro/Qwen3.8-Flash-Next-abliterated-GSQ-RCO-Strata-GGUF) |
| Quantization | **IQ2_XS / GSQ-RCO**, two GGUF shards, ~**68 GB** download |
| Abliteration | Huihui refusal-direction **control vector**, applied at inference (projection, strength 1.0, layers 4–44); the GGUF weights themselves are unchanged |
| API | Strata OpenAI-compatible chat completions |
| Context | **250,000 tokens configured**; base model advertises native context up to **262,144** |
| KV cache | **Q4_0**, streaming enabled, `--kv-resident 32768` in the observed engine command |
| GPU roles | GPU 0: helper; GPU 1: main; all other GPUs excluded |
| RAM-reserve setting | `STRATA_RAM_KEEP_GIB=16` (a *soft target*, not a hard memory cap) |
| Vision | Disabled in this deployment; API advertised text input/output only |
| MTP setup | `--mtp off` requested during setup; see speculation caveat below |

Relevant setup choices (not the full installation command):

```bash
./setup.sh --setup --yes --family qwen --model IQ2_XS \
  --gpu-roles "0=helper,1=main,2=off" \
  --context 250000 --kv q4_0 --kv-streaming auto \
  --vision no --mtp off \
  --experimental-speed-projection /path/to/Huihui-Qwen3.8-Flash-Next-refusal-direction-r.gguf
```

## Measured results

| Test | Input tokens | Output tokens | Wall time | Reported decode speed | Result |
|---|---:|---:|---:|---:|---|
| Short prime-number code request (8k configuration) | 74 | 183 | 6.49 s | ~28.2 tokens/s **end-to-end**¹ | Correct response |
| Coding generation (8k configuration) | ~100 | 1,200 | 32.05 s | **39.0 tokens/s**² | Output limit reached |
| Coding generation (32k configuration) | ~100 | 1,200 | 32.29 s | **38.8 tokens/s**² | Output limit reached |
| Needle retrieval (120k configuration, requested 110k at 50% depth) | 107,561 | — | **133 s** | — | **FOUND (1/1)** |
| Long-prompt generation (120k configuration) | 107,595 | 700 | **108.06 s** | **35.1 tokens/s**² | Marker found |
| Needle retrieval (250k configuration, requested 230k at 50% depth) | **225,354** | — | **307 s** | — | **FOUND (1/1)** |

¹ This rate divides generated tokens by the *whole request duration*; it is not the engine's decode-only rate.

² Decode rates are from the engine's timings, not simply total output divided by wall time.

### Prefill measurements and caching

The **107,595-token** generation run reported:

```text
Prompt tokens (total):    107,595
Cached tokens:             43,008
Uncached prompt tokens:    64,587
Reported prefill rate:      734.4 tokens/s (uncached tokens only)
Prefill time:               87.949 s
Generated tokens:           700
Decode time:                19.937 s
Reported decode rate:       35.1 tokens/s
Total request time:         108.06 s
```

**Important:** **734.4 tokens/s is not a cold-prefill measurement for all 107,595 tokens.** The engine reused 43,008 cached tokens. A completely cold run could perform differently. For the short ~100-token coding prompts, reported prefill rates were about **58.8 tokens/s** (8k) and **57.5 tokens/s** (32k); rates for such tiny inputs are not representative of long-prompt throughput.

### Long-context retrieval

These were single-run **needle-in-a-haystack** checks using Strata's `tools/needle_bench.py` at **50% depth**:

```bash
# In the Strata checkout, against a running local API:
.venv/bin/python tools/needle_bench.py \
  --lengths 110k --depths 50 \
  --url http://127.0.0.1:18080 --timeout 3600

.venv/bin/python tools/needle_bench.py \
  --lengths 230k --depths 50 \
  --url http://127.0.0.1:18080 --timeout 3600
```

Results:

- **110k requested:** `FOUND (107,561 prompt tokens, 133 s)`
- **230k requested:** `FOUND (225,354 prompt tokens, 307 s)`

The benchmark's requested length is not identical to the actual API prompt token count. **250,000** is the configured maximum context, **not** the largest prompt fully verified in this report. The longest successful measured prompt is **225,354 tokens**; this does **not** establish reliability at every position, over multiple trials, or with large outputs near the context limit.

## Memory / GPU observations

With the 250k configuration loaded, a later snapshot showed:

| Resource | Observed usage |
|---|---:|
| RTX 2070 #0 | **7,040 MiB / 8,192 MiB** |
| RTX 2070 #1 | **6,946 MiB / 8,192 MiB** |
| System RAM | **~37 GiB available / 78 GiB total** |
| Swap | ~**1.3 GiB used / 8 GiB** |

The systemd service was configured with `STRATA_RAM_KEEP_GIB=16`; that setting is **not a guaranteed RAM limit**. RAM, swap use and speed can change under concurrent load and during long prompts. These numbers describe one observed state, not peak-memory measurements across every test.

## Limitations and interpretation

- All results are **informal single-system tests**, not multi-run statistical benchmarks. The needle tests were **1/1 at one depth**, not a comprehensive context-quality evaluation.
- Reported speeds mix different request types and context configurations. **Decode throughput** and **end-to-end throughput** should not be compared as if they were identical.
- The 110k prefill result includes **43,008 cached tokens**. Do not interpret the reported 734.4 tokens/s as full cold-prefill speed.
- Model-output quality was not formally scored. One long coding response included an invalid Python dataclass argument, so these numbers should **not** be interpreted as a coding accuracy score.
- Setup requested `--mtp off`, but the observed engine launch still included `--spec 4`, and one timing record contained draft counters. The exact speculation path was **not independently isolated or benchmarked**; no claim of a proven speculation-free run is made here.
- The 125B checkpoint is capable of multimodal use in other configurations, but **this deployment was text-only** (`--vision no`).
- **No other model server is included in these performance numbers.** All results are from the described Strata deployment.

## Credits and references

- **Base model:** [Qwen/Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next)
- **Quantization:** [ISTA-DASLab GSQ-RCO GGUF](https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF)
- **GGUF distribution and projection vector:** [alesha-pro/Qwen3.8-Flash-Next-abliterated-GSQ-RCO-Strata-GGUF](https://huggingface.co/alesha-pro/Qwen3.8-Flash-Next-abliterated-GSQ-RCO-Strata-GGUF)
- **Inference engine and benchmark utility:** [Niko1221/Strata](https://github.com/Niko1221/Strata)

Model weights and quantization are the work of their respective authors. The benchmark figures above were recorded on the described personal setup; they are **not official Strata, Qwen or ISTA-DASLab benchmark results**.
