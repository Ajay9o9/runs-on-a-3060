# runs-on-a-3060

Bench logs and commands for local LLM, image, and video models on **RTX 3060 12GB**.

Reference only: hardware, flags, **KV**, prefill (**pp**), generation (**tg**), VRAM. Not a guide or blog.

**X:** [@ItsmeAjayKV](https://x.com/ItsmeAjayKV)

## Machine

| | |
|--|--|
| GPU | RTX 3060 12GB (~11887 MiB) |
| CPU | Ryzen 9 9900X · usually `-t 12` |
| RAM | **64 GB** DDR5 @ 6000 MT/s |
| OS | Ubuntu 24.04 |

Hybrid MoE uses **system RAM** for experts. Always log **`-ctk` / `-ctv`**.  
Details: [HARDWARE.md](HARDWARE.md) · [techniques/kv-cache.md](techniques/kv-cache.md)

## Browse

| | |
|--|--|
| **[RESULTS.md](RESULTS.md)** | All snapshot tables (pp + tg + MTP + image + video) |
| [models/](models/) | Per-model commands and full benches |
| [image/](image/) | Bonsai Image 4B |
| [video/](video/) | MiniMax H3 (video + audio) |
| [runtimes/](runtimes/) | llama.cpp / ik / tq3 / turboquant / benchy |
| [techniques/](techniques/) | KV, MoE offload, MTP, power limit |
| [recipes/](recipes/) | Short copy-paste commands |
| [recipes/threejs-game-qwen-mtp.md](recipes/threejs-game-qwen-mtp.md) | Three.js game dev: Qwen MTP Q6 server + ik_llama |
| [data/underclock/](data/underclock/) | GPU CSVs / plots |

## Reports

HTML cards (GitHub Pages). Listing: [ajay9o9.github.io/runs-on-a-3060](https://ajay9o9.github.io/runs-on-a-3060/).

| Report | |
|--------|--|
| [Ornith 1.5 35B-A3B](https://ajay9o9.github.io/runs-on-a-3060/reports/ornith-1.5-35b/) | RAM floors + t/s · Q6_K / Q4_K_M / AD mixed |

## Models catalog

| Model | Kind | Page |
|-------|------|------|
| Qwen3.6 35B-A3B | MoE LLM | [models/qwen3.6-35b-a3b.md](models/qwen3.6-35b-a3b.md) |
| Qwen3.6 27B MTP | dense LLM + MTP | [models/qwen3.6-27b.md](models/qwen3.6-27b.md) |
| Gemma 4 12B | dense LLM (± QAT / MTP) | [models/gemma4-12b.md](models/gemma4-12b.md) |
| Gemma 4 26B-A4B | MoE LLM | [models/gemma4-26b-a4b.md](models/gemma4-26b-a4b.md) |
| Diffusion Gemma 26B-A4B | text diffusion LLM | [models/diffusion-gemma.md](models/diffusion-gemma.md) |
| Ternary Bonsai 27B | ternary dense LLM | [models/bonsai-ternary-27b.md](models/bonsai-ternary-27b.md) |
| Laguna S-2.1 | large MoE (118B-A8B) | [models/laguna-s-2.1.md](models/laguna-s-2.1.md) |
| Ling-3.0-flash | MoE LLM ([AtomicChat Q4_K_M GGUF](https://huggingface.co/AtomicChat/Ling-3.0-flash-GGUF)) | [models/ling-3.0-flash.md](models/ling-3.0-flash.md) |
| Ornith 1.5 35B-A3B | MoE LLM (Q6_K / Q4_K_M / AD mixed) | [models/ornith-1.5-35b.md](models/ornith-1.5-35b.md) |
| Bonsai Image 4B | image gen | [image/bonsai-image-4b.md](image/bonsai-image-4b.md) |
| MiniMax H3 | video + audio gen | [video/minimax-h3.md](video/minimax-h3.md) |

## Highlights (one line each)

| | pp | tg | Link |
|--|---:|---:|------|
| Qwen 35B **TQ3** (4k) | ~610 | ~**54** | [RESULTS](RESULTS.md#qwen36-35b-a3b) |
| Qwen 35B **TQ3** (128k) | ~515 | ~**59** | same |
| Qwen 35B Q6 hybrid (ik) | ~805 | ~**47** | same |
| Qwen 35B **MTP** n-max 2–3 vs off | — | ~**33** vs ~21 | [RESULTS](RESULTS.md#mtp-qwen-35b-mtp-q6) |
| Qwen 35B MTP Q6 **ik vs llama** prefill | ~**821** vs ~473 | ~46 | [three.js use case](recipes/threejs-game-qwen-mtp.md) |
| Gemma 4 12B **Q5** | ~1100 | ~**32** | [RESULTS](RESULTS.md#gemma-4-12b) |
| Gemma 4 12B **QAT** @ 32k + MTP | ~1011 | ~**40.5** | [RESULTS](RESULTS.md#gemma-qat-mtp-32k--64k) |
| Gemma 4 26B-A4B (ncmoe 20) | ~690 | ~**39** | [RESULTS](RESULTS.md#gemma-4-26b-a4b) |
| Laguna S-2.1 **IQ3_S** hybrid (f16 ~32k) | ~202 | ~**23** | [RESULTS](RESULTS.md#laguna-s-21-ud-iq3_s--ud-q4_k_m) · [models](models/laguna-s-2.1.md) |
| Laguna S-2.1 **Q4_K_M** (q4_0, ncmoe 46, ~164k) | ~54 | ~**12** | same |
| Ling-3.0-flash **Q4_K_M** hybrid (ncmoe 42) | ~87–135* | ~**16–21** | [RESULTS](RESULTS.md#ling-30-flash-q4_k_m) · [models](models/ling-3.0-flash.md) |
| Ornith 1.5 **IQ4_XS-IQ3_S** 16 GB (ncmoe 22, d112k) | — | ~**30.5** | [RESULTS](RESULTS.md#ornith-15-35b-a3b) · [models](models/ornith-1.5-35b.md) |
| Ornith 1.5 **Q4_K_M** 24 GB (ncmoe 24, d112k) | — | ~**30.6** | same |
| Ornith 1.5 **Q6_K** 32 GB (ncmoe 28, d112k) | — | ~**27.6** | same |
| Ternary Bonsai 27B | ~440 @24k | ~**23–25** | [RESULTS](RESULTS.md#other-llm) |
| Bonsai Image 4B | — | ~10 s / ~6.6 GB | [RESULTS](RESULTS.md#image) |
| MiniMax H3 480p/10 s **+ audio** | — | ~**6:10–6:47** | [RESULTS](RESULTS.md#video) · [video](video/minimax-h3.md) |

\*Ling first pp4096 cold (~30 t/s); steadier band is pp8k–32k.

Full rows, KV, offload, and depths → **[RESULTS.md](RESULTS.md)**.

## Runtimes

| | Repo |
|--|------|
| llama.cpp | [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) |
| ik_llama.cpp | [ikawrakow/ik_llama.cpp](https://github.com/ikawrakow/ik_llama.cpp) |
| TQ3 | [turbo-tan/llama.cpp-tq3](https://github.com/turbo-tan/llama.cpp-tq3) |
| TurboQuant (KV) | [TheTom/llama-cpp-turboquant](https://github.com/TheTom/llama-cpp-turboquant) |
| TurboQuant (AtomicChat Ling) | [AtomicBot-ai/atomic-llama-cpp-turboquant](https://github.com/AtomicBot-ai/atomic-llama-cpp-turboquant) |
| Diffusion Gemma | llama.cpp branch `nvidia-diffusion-gemma` ([lnigam/llama.cpp](https://github.com/lnigam/llama.cpp)) |
| llama-benchy | [eugr/llama-benchy](https://github.com/eugr/llama-benchy) |
| Bonsai Image | [PrismML-Eng/Bonsai-image-demo](https://github.com/PrismML-Eng/Bonsai-image-demo) |

Side-by-side Qwen forks: [runtimes/comparison.md](runtimes/comparison.md)

## Log checklist (LLM)

```text
runtime + commit
-ngl / -ncmoe (or exps=CPU) / -t
-ctk / -ctv
pp t/s + prompt size
tg t/s
peak VRAM (+ host RAM if hybrid)
MTP: n-max / draft model if used
```

```bash
export MODEL_DIR=/path/to/ggufs
./build/bin/llama-bench ... -ctk <type> -ctv <type> ...
```

## Names

| Name | Kind |
|------|------|
| Ternary Bonsai 27B | text LLM |
| Bonsai Image 4B | image gen |
| Diffusion Gemma | text diffusion LLM |
| Laguna S-2.1 | large MoE LLM |
| Ling-3.0-flash | MoE LLM |
| Ornith 1.5 35B-A3B | MoE LLM |
| MiniMax H3 | video + audio gen |

## Contributing / license

[CONTRIBUTING.md](CONTRIBUTING.md) · [CC-BY-4.0](LICENSE)
