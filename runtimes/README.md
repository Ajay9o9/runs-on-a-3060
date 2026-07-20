# Runtimes tried on RTX 3060

This lab is not “stock llama.cpp only.” Different forks unlock different quants, KV types, and speeds.

## Quick map

| Runtime | Repo | What we used it for | Standout on 3060 |
|---------|------|---------------------|------------------|
| **llama.cpp** (upstream / mainline) | [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) | Gemma 4, general GGUF, server, MTP branches | Solid baseline; Q6 MoE sometimes weaker than ik |
| **ik_llama.cpp** | [ikawrakow/ik_llama.cpp](https://github.com/ikawrakow/ik_llama.cpp) | Qwen3.6 Unsloth Q6, expert offload, long PP | Strong PP/TG on Q6; Q6 worked when mainline struggled |
| **llama.cpp-tq3** | [turbo-tan/llama.cpp-tq3](https://github.com/turbo-tan/llama.cpp-tq3) | TQ3_4S weights + `tq3_0` V cache | **Fastest Qwen gen** seen here (~54–60 t/s) |
| **TurboQuant / TheTom branch** | community TurboQuant llama.cpp branch | `turbo4` / `turbo3` KV on Q4_K_XL | Early daily-driver server (~30–43 t/s hybrid) |
| **llama.cpp MTP** (e.g. PR / draft-mtp) | speculative MTP branches | Qwen + Gemma MTP | Gen ~20 → ~33+ t/s with VRAM tax |
| **llama-diffusion-gemma** | diffusion-gemma fork/cli | Diffusion Gemma 26B-A4B | Text diffusion, ~0.5 s/step @ ngl 15 |
| **llama-benchy** | HTTP client vs `llama-server` | Server-side PP/TG at filled context | Gemma long-context, Bonsai 27B |

## How to read results

Every serious result should list:

1. **Runtime name** + rough build/commit if known  
2. **Command** (flags)  
3. **Offload story** (`-ngl`, `-ncmoe` / `-ot exps=CPU`, threads)  
4. **pp t/s** and **tg t/s**  
5. **Peak VRAM** when measured  
6. Reminder: host has **64 GB RAM** — hybrid MoE dumps experts into system RAM  

Full side-by-side numbers: [comparison.md](comparison.md).

## Build tip (from lab notes)

When building CUDA llama.cpp variants for flexible KV types:

```text
GGML_CUDA_FA_ALL_QUANTS
```

Lab note: helps avoid KV paths falling back in ways that hurt GPU FA for some quant mixes.
