# Results snapshot

All numbers from one machine: **RTX 3060 12GB · Ryzen 9 9900X · 64 GB RAM · Ubuntu 24.04**.  
Full commands and flags: [models/](models/) · [runtimes/comparison.md](runtimes/comparison.md).

## Metrics

| Label | Meaning |
|-------|---------|
| **pp** | prefill / prompt processing (tok/s) |
| **tg** | generation / decode (tok/s) |
| **KV** | `-ctk` / `-ctv` (see [techniques/kv-cache.md](techniques/kv-cache.md)) |

Do not compare a row’s **tg** to another row’s **pp**. Prefill size is noted (e.g. pp4096).

---

## Qwen3.6 35B-A3B

| Setup | Runtime | Offload | KV | pp | tg |
|-------|---------|---------|-----|----|----|
| TQ3_4S, 4k | llama.cpp-tq3 | ngl 99, ncmoe 32 | q4_0 / tq3_0 | pp4096 ~600–620 | ~54 |
| TQ3_4S, 128k | llama.cpp-tq3 | ngl 99, ncmoe 16–24 | q4_0 / tq3_0 | pp131072 ~500–530 | ~58–60 |
| Q5 Unsloth | mainline | ncmoe 32 | q8_0 / q8_0 | pp4096 ~500 | ~48 |
| Q5 Unsloth | ik_llama | exps=CPU | q4_0 / q4_0 | pp4096 ~825 | ~46 |
| Q6 Unsloth | ik_llama | ncmoe 32 | q8_0 / q8_0 | pp4096 ~800–810 | ~47 |
| Q4_K_XL horizontal | [turboquant](https://github.com/TheTom/llama-cpp-turboquant) | ngl 99, ncmoe 48 | turbo4 / turbo3 | pp4096 ~480 | ~43 |
| Q4_K_XL vertical | same | ngl 20 | turbo4 / turbo3 | pp4096 ~560 | ~30 |

Page: [models/qwen3.6-35b-a3b.md](models/qwen3.6-35b-a3b.md) · forks: [runtimes/comparison.md](runtimes/comparison.md)

### MTP (Qwen 35B MTP Q6)

Short prompts, KV `q8_0` / `q8_0`, hybrid MoE (e.g. ngl 5–6, ncmoe 24–32). See [techniques/mtp.md](techniques/mtp.md).

| Mode | tg | VRAM (approx) |
|------|---:|---------------|
| no MTP | ~20–22 | ~5.9–8.0 GB |
| MTP n-max 2–3 | ~31–36 | ~8.0–10.5 GB |
| MTP n-max 5–6 | often worse than 2–3 | ~8–10.5 GB |

---

## Gemma 4 12B

| Setup | Runtime | KV | pp | tg | VRAM note |
|-------|---------|-----|----|----|-----------|
| Q5_K_XL | llama.cpp | q8_0 / q8_0 | pp4096 ~1070–1150 | ~30–33 | full GPU |
| Q6_K_XL | llama.cpp | q8_0 / q8_0 | pp4096 ~1110 | ~26 | peak ~11.3 GB |
| Q8, ngl 40 | llama.cpp | q8_0 / q8_0 | pp4096 ~986 | ~15 | peak ~11.2 GB |
| Unsloth Q4 non-QAT @ 32k depth | llama.cpp + [llama-benchy](https://github.com/eugr/llama-benchy) | q8_0 / q8_0 | pp4096@d32k ~820 | tg256 ~37 | ~9980 MB |
| Unsloth **QAT** Q4 @ 32k, no MTP | same | q8_0 / q8_0 | pp4096@d32k ~880–1000 | tg256 ~34–39 | ~9380 MB |
| Unsloth **QAT** Q4 @ 32k, MTP n-max 4 | same + draft | q8_0 / q8_0 | pp4096@d32k ~1011 | tg256 ~40.5 | |

Long-context QAT ± MTP (32k–256k): [models/gemma4-12b.md](models/gemma4-12b.md)

### Gemma QAT MTP (32k / 64k)

| Mode | depth | pp4096 | tg256 |
|------|------:|-------:|------:|
| QAT, no MTP | 32k | ~1000 | ~39 |
| QAT + MTP n-max 4 | 32k | ~1011 | ~40.5 |
| QAT, no MTP | 64k | ~810 | ~39 |
| QAT + MTP n-max 4 | 64k | ~808 | ~35 |

---

## Gemma 4 26B-A4B

| Setup | Offload | KV | pp | tg | Peak VRAM |
|-------|---------|-----|----|----|-----------|
| Q6 | ngl 99, ncmoe 32 | q8_0 / q8_0 | pp4096 ~540 | ~34 | ~3.8 GB |
| Q6 | ngl 99, ncmoe 20 | q8_0 / q8_0 | pp4096 ~690 | ~39 | ~10.1 GB |

Page: [models/gemma4-26b-a4b.md](models/gemma4-26b-a4b.md)

---

## Other LLM

| Model | Setup | pp | tg / other | Notes |
|-------|-------|----|------------|-------|
| Ternary Bonsai 27B | ngl 999, KV not recorded | pp24576 ~440 | tg256 ~23–25 | [models/bonsai-ternary-27b.md](models/bonsai-ternary-27b.md) |
| Diffusion Gemma 26B | llama.cpp branch `nvidia-diffusion-gemma`, ngl 15 | short prefill ~32–35 | ~0.5 s/step | text diffusion; peak ~10450 MB — [models/diffusion-gemma.md](models/diffusion-gemma.md) |
| Qwen 27B MTP | commands | — | — | [models/qwen3.6-27b.md](models/qwen3.6-27b.md) |

---

## Image

| Model | Stack | Time | Peak VRAM |
|-------|-------|-----:|----------:|
| Bonsai Image 4B ternary | [Bonsai-image-demo](https://github.com/PrismML-Eng/Bonsai-image-demo) | ~9.7 s (~2.4 s/step) | ~6585 MB |

Page: [image/bonsai-image-4b.md](image/bonsai-image-4b.md)
