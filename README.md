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

### LLM (tg ≈ tokens/s generation)

| Model | Quant / setup | Runtime | Offload | KV (`ctk`/`ctv`) | tg | Notes |
|-------|---------------|---------|---------|------------------|---:|-------|
| Qwen3.6 35B-A3B | TQ3_4S | llama.cpp-tq3 | ngl 99, ncmoe 16–32 | **q4_0 / tq3_0** | ~54–60 | hybrid |
| Qwen3.6 35B-A3B | Q5 Unsloth | mainline | ncmoe 32 | **q8_0 / q8_0** | ~48 | large host RAM |
| Qwen3.6 35B-A3B | Q5 Unsloth | ik_llama | exps=CPU | **q4_0 / q4_0** | ~46 | large host RAM |
| Qwen3.6 35B-A3B | Q6 Unsloth | ik_llama | ncmoe 32 | **q8_0 / q8_0** | ~47 | large host RAM |
| Qwen3.6 35B-A3B | Q4_K_XL | [TheTom turboquant](https://github.com/TheTom/llama-cpp-turboquant) | ngl 99, ncmoe 48 | **turbo4 / turbo3** | ~43 | vs ~30 @ ngl 20 same KV |
| Qwen3.6 35B MTP | MTP Q6 | llama.cpp MTP branch | n-max 2–3 | **q8_0 / q8_0** | ~31–36 | higher VRAM than no-MTP |
| Gemma 4 12B | Q5_K_XL | llama.cpp | full GPU | **q8_0 / q8_0** | ~30–33 | |
| Gemma 4 12B | Q6_K_XL | llama.cpp | full GPU | **q8_0 / q8_0** | ~26 | peak VRAM ~11.3 GB |
| Gemma 4 26B-A4B | Q6 | llama.cpp | ncmoe 20–32 | **q8_0 / q8_0** | ~33–39 | VRAM ~3.8–10.1 GB |
| Ternary Bonsai 27B | Q2_0 | llama-server | ngl 999 | **not recorded** | ~23–25 | LLM, not image |
| Diffusion Gemma 26B | Q4 | llama.cpp `nvidia-diffusion-gemma` | ngl 15 | **not recorded** | ~0.5 s/step | text diffusion; peak ~10450 MB |

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

**Logged fields (LLM):** runtime, `-ngl`, `-ncmoe` / expert offload, `-t`, **`-ctk` / `-ctv`**, pp t/s, tg t/s, peak VRAM, host RAM context.

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
