# runs-on-a-3060

Bench logs and commands for local LLM and image models on **RTX 3060 12GB**.

Not a guide or opinion piece. Tables, flags, and measured numbers from one machine. Narrative posts (X, etc.) can link here; this repo stays reference-only.

**X:** [@ItsmeAjayKV](https://x.com/ItsmeAjayKV)

## Test machine

| | |
|--|--|
| GPU | RTX 3060 12GB (~11887 MiB) |
| CPU | Ryzen 9 9900X (12c / 24t), `-t 12` in most benches |
| RAM | 64 GB DDR5 @ 6000 MT/s |
| OS | Ubuntu 24.04 |

Details: [HARDWARE.md](HARDWARE.md).

Hybrid MoE configs place large expert weights in **system RAM**. Match your RAM when comparing.

**KV cache is required context for LLM rows.** Flags: `-ctk` (K) / `-ctv` (V). See [techniques/kv-cache.md](techniques/kv-cache.md).

## Index

| Path | Contents |
|------|----------|
| [models/](models/) | LLM models: commands, offload, **KV**, pp/tg, VRAM |
| [image/](image/) | Image gen (Bonsai Image 4B) |
| [runtimes/](runtimes/) | llama.cpp / ik_llama / tq3 / other builds used |
| [techniques/](techniques/) | KV cache, MoE flags, MTP, power limit |
| [recipes/](recipes/) | Short command snippets |
| [data/underclock/](data/underclock/) | GPU sample CSVs and plots |

## Results snapshot

Values are from this lab only. Full commands live on the model/runtime pages.

**Metrics**

| Label | Meaning |
|-------|---------|
| **pp** | prefill / prompt processing (tok/s) — how fast the prompt is ingested |
| **tg** | text generation / decode (tok/s) — how fast new tokens are produced |
| **KV** | `-ctk` / `-ctv` cache types |

Prefill and generation are **both** logged. Do not compare a row’s tg to another’s pp. Prefill size is noted (e.g. pp4096 = 4096-token prompt test).

### LLM — prefill + generation

| Model | Quant / setup | Runtime | Offload | KV (`ctk`/`ctv`) | pp (tok/s) | tg (tok/s) | Notes |
|-------|---------------|---------|---------|------------------|------------|------------|-------|
| Qwen3.6 35B-A3B | TQ3_4S, 4k | llama.cpp-tq3 | ngl 99, ncmoe 32 | **q4_0 / tq3_0** | pp4096 ~**600–620** | ~**54** | hybrid |
| Qwen3.6 35B-A3B | TQ3_4S, 128k | llama.cpp-tq3 | ngl 99, ncmoe 16–24 | **q4_0 / tq3_0** | pp131072 ~**500–530** | ~**58–60** | long-context prefill still high |
| Qwen3.6 35B-A3B | Q5 Unsloth | mainline | ncmoe 32 | **q8_0 / q8_0** | pp4096 ~**500** | ~**48** | large host RAM |
| Qwen3.6 35B-A3B | Q5 Unsloth | ik_llama | exps=CPU | **q4_0 / q4_0** | pp4096 ~**825** | ~**46** | large host RAM |
| Qwen3.6 35B-A3B | Q6 Unsloth | ik_llama | ncmoe 32 | **q8_0 / q8_0** | pp4096 ~**800–810** | ~**47** | large host RAM |
| Qwen3.6 35B-A3B | Q4_K_XL horizontal | [TheTom turboquant](https://github.com/TheTom/llama-cpp-turboquant) | ngl 99, ncmoe 48 | **turbo4 / turbo3** | pp4096 ~**480** | ~**43** | |
| Qwen3.6 35B-A3B | Q4_K_XL vertical | same | ngl 20 | **turbo4 / turbo3** | pp4096 ~**560** | ~**30** | same KV; slower gen |
| Gemma 4 12B | Q5_K_XL | llama.cpp | full GPU | **q8_0 / q8_0** | pp4096 ~**1070–1150** | ~**30–33** | |
| Gemma 4 12B | Q6_K_XL | llama.cpp | full GPU | **q8_0 / q8_0** | pp4096 ~**1110** | ~**26** | peak VRAM ~11.3 GB |
| Gemma 4 12B | Unsloth Q4 non-QAT @ 32k depth | llama.cpp + [llama-benchy](https://github.com/eugr/llama-benchy) | full GPU | **q8_0 / q8_0** | pp4096@d32k ~**820** | tg256 ~**37** | ~9980 MB VRAM |
| Gemma 4 12B | Unsloth **QAT** Q4 @ 32k, no MTP | same | full GPU | **q8_0 / q8_0** | pp4096@d32k ~**880–1000** | tg256 ~**34–39** | ~9380 MB VRAM |
| Gemma 4 12B | Unsloth **QAT** Q4 @ 32k, **MTP n-max 4** | same + draft MTP | full GPU | **q8_0 / q8_0** | pp4096@d32k ~**1011** | tg256 ~**40.5** | see MTP section |
| Gemma 4 26B-A4B | Q6, ncmoe 32 | llama.cpp | ngl 99, ncmoe 32 | **q8_0 / q8_0** | pp4096 ~**540** | ~**34** | peak VRAM ~3.8 GB |
| Gemma 4 26B-A4B | Q6, ncmoe 20 | llama.cpp | ngl 99, ncmoe 20 | **q8_0 / q8_0** | pp4096 ~**690** | ~**39** | peak VRAM ~10.1 GB |
| Ternary Bonsai 27B | Q2_0 | llama-server + [llama-benchy](https://github.com/eugr/llama-benchy) | ngl 999 | **not recorded** | pp24576 ~**440** | tg256 ~**23–25** | long PP tests 24k–49k |
| Diffusion Gemma 26B | Q4 | llama.cpp `nvidia-diffusion-gemma` | ngl 15 | **not recorded** | prefill ~**32–35** (short) | ~**0.5 s/step** | text diffusion; peak ~10450 MB |

Details: [models/](models/) · [runtimes/comparison.md](runtimes/comparison.md).

### MTP (speculative decoding)

MTP is **not** the same as a plain `llama-bench` tg row. Draft depth (`n-max`), accept rate, and extra VRAM matter. Full notes: [techniques/mtp.md](techniques/mtp.md) · [models/qwen3.6-35b-a3b.md](models/qwen3.6-35b-a3b.md).

**Qwen3.6 35B MTP Q6** — short interactive-style prompts, **KV `q8_0` / `q8_0`**, hybrid MoE (example: `ngl 5–6`, `ncmoe 24–32`):

| Mode | tg (tok/s) | VRAM (approx) | Notes |
|------|------------:|---------------|-------|
| no MTP | ~**20–22** | ~5.9–8.0 GB | baseline on partial GPU expert budget |
| MTP `n-max` 2–3 | ~**31–36** | ~8.0–10.5 GB | best band in lab sweeps |
| MTP `n-max` 5–6 | often **worse** than 2–3 | ~8–10.5 GB | over-draft; accept rate / speed drop |

**Gemma 4 12B Unsloth QAT Q4** — long-context [llama-benchy](https://github.com/eugr/llama-benchy), **KV `q8_0` / `q8_0`**, draft MTP `n-max` 4 (2026-06-09):

| Mode | depth | pp4096 (t/s) | tg256 (t/s) |
|------|------:|-------------:|------------:|
| QAT, no MTP | 32k | ~1000 | ~**39** |
| **QAT + MTP n-max 4** | 32k | ~**1011** | ~**40.5** |
| QAT, no MTP | 64k | ~810 | ~**39** |
| **QAT + MTP n-max 4** | 64k | ~808 | ~**35** |

Non-QAT Unsloth Q4 baseline at 32k: pp ~**820**, tg ~**37** (~9980 MB VRAM). Full depth tables (32k–256k): [models/gemma4-12b.md](models/gemma4-12b.md).

Other MTP-related logs:

| What | Where |
|------|--------|
| Qwen MTP n-max sweeps (ngl / ncmoe matrix) | [models/qwen3.6-35b-a3b.md](models/qwen3.6-35b-a3b.md) § E |
| Qwen 27B MTP commands | [models/qwen3.6-27b.md](models/qwen3.6-27b.md) |
| Gemma 4 12B Q5 + MTP draft (server + llama-benchy) | [models/gemma4-12b.md](models/gemma4-12b.md) |
| Gemma 4 12B **QAT Q4** with/without MTP | [models/gemma4-12b.md](models/gemma4-12b.md) § QAT |

### Image

| Model | Stack | Time | Peak VRAM |
|-------|-------|-----:|----------:|
| Bonsai Image 4B ternary | [Bonsai-image-demo](https://github.com/PrismML-Eng/Bonsai-image-demo) / gemlite | ~9.7 s (~2.4 s/step) | ~6585 MB |

## Runtimes used

| Runtime | Link | Used for |
|---------|------|----------|
| llama.cpp | [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) | Gemma, general GGUF, server, MTP branches |
| ik_llama.cpp | [ikawrakow/ik_llama.cpp](https://github.com/ikawrakow/ik_llama.cpp) | Qwen Q5/Q6 hybrid |
| llama.cpp-tq3 | [turbo-tan/llama.cpp-tq3](https://github.com/turbo-tan/llama.cpp-tq3) | TQ3_4S, `ctv tq3_0` |
| llama-cpp-turboquant | [TheTom/llama-cpp-turboquant](https://github.com/TheTom/llama-cpp-turboquant) | `turbo4` / `turbo3` KV on Q4_K_XL |
| llama.cpp (diffusion) | branch **`nvidia-diffusion-gemma`** on llama.cpp ([lnigam remote used in lab](https://github.com/lnigam/llama.cpp); still llama.cpp, not a separate project) | Diffusion Gemma binary `llama-diffusion-gemma-cli` |
| llama-benchy | [eugr/llama-benchy](https://github.com/eugr/llama-benchy) | HTTP benches vs `llama-server` (long context) |

Side-by-side commands: [runtimes/comparison.md](runtimes/comparison.md).

## Command convention

```bash
export MODEL_DIR=/path/to/ggufs
# from the runtime build directory
./build/bin/llama-bench ... -ctk <type> -ctv <type> ...
./build/bin/llama-server ... -ctk <type> -ctv <type> ...
```

**Logged fields (LLM):** runtime, `-ngl`, `-ncmoe` / expert offload, `-t`, **`-ctk` / `-ctv`**, **pp (prefill) t/s**, **tg (generation) t/s**, peak VRAM, host RAM context. MTP rows also need `n-max` / draft settings.

## Name disambiguation

| Name | Kind |
|------|------|
| Ternary Bonsai 27B | text LLM (GGUF) |
| Bonsai Image 4B ternary | image gen (PrismML) |
| Diffusion Gemma | text diffusion LLM |

## Contributing

[CONTRIBUTING.md](CONTRIBUTING.md) — GPU, CPU, system RAM, runtime, **KV cache types**, exact command, numbers.

## License

[CC-BY-4.0](LICENSE)
