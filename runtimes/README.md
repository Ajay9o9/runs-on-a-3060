# runtimes/

Builds used for the logs.

| Runtime | Repo / branch | Used for |
|---------|---------------|----------|
| llama.cpp | [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) | Gemma, general GGUF, server, MTP branches |
| ik_llama.cpp | [ikawrakow/ik_llama.cpp](https://github.com/ikawrakow/ik_llama.cpp) | Qwen3.6 Unsloth Q6, expert offload |
| llama.cpp-tq3 | [turbo-tan/llama.cpp-tq3](https://github.com/turbo-tan/llama.cpp-tq3) | TQ3_4S, `ctv tq3_0` |
| llama-cpp-turboquant | [TheTom/llama-cpp-turboquant](https://github.com/TheTom/llama-cpp-turboquant) | `turbo4` / `turbo3` KV on Q4_K_XL |
| atomic-llama-cpp-turboquant | [AtomicBot-ai/atomic-llama-cpp-turboquant](https://github.com/AtomicBot-ai/atomic-llama-cpp-turboquant) — prebuilt release **b10269-1.5.0+** | **Required** for AtomicChat Ling-3.0-flash GGUFs — see [below](#atomic-llama-cpp-turboquant-atomicchat-ling) |
| llama.cpp diffusion | **Branch `nvidia-diffusion-gemma`** (llama.cpp tree; lab remote [lnigam/llama.cpp](https://github.com/lnigam/llama.cpp) tracking `lnigam/nvidia-diffusion-gemma`) | `llama-diffusion-gemma-cli` for Diffusion Gemma — **not a separate product**, still llama.cpp |
| llama-benchy | [eugr/llama-benchy](https://github.com/eugr/llama-benchy) | long-context server benches over HTTP |

## Logged fields

When present in a log entry:

1. Runtime name (+ build/commit if recorded)
2. Command and flags
3. Offload: `-ngl`, `-ncmoe` / `-ot exps=CPU`, `-t`
4. **KV cache: `-ctk` / `-ctv` (type_k / type_v)** — required for LLM
5. pp t/s, tg t/s
6. Peak VRAM if measured
7. Host RAM context: this machine has **64 GB** (experts often on host)

Commands and tables: [comparison.md](comparison.md).  
KV types used in this lab: [../techniques/kv-cache.md](../techniques/kv-cache.md).

## atomic-llama-cpp-turboquant (AtomicChat Ling)

Fork used for [AtomicChat/Ling-3.0-flash-GGUF](https://huggingface.co/AtomicChat/Ling-3.0-flash-GGUF).
**Stock llama.cpp will not run these files** — the quantizations need a TurboQuant-enabled build, and stock
`llama-quantize` cannot produce them either. Not the same project as
[TheTom/llama-cpp-turboquant](https://github.com/TheTom/llama-cpp-turboquant) (that one is the `turbo4`/`turbo3`
**KV cache** fork); this one is about the **weight** format.

No compilation needed — prebuilt archives from release **b10269-1.5.0 or newer**:

```bash
wget https://github.com/AtomicBot-ai/atomic-llama-cpp-turboquant/releases/download/b10269-1.5.0/llama-turboquant-linux-x64-cuda-13.3.tar.gz
tar xzf llama-turboquant-linux-x64-cuda-13.3.tar.gz && cd llama-turboquant-*
```

Archives: Linux x64 CUDA 13.3 (this lab), macOS arm64, Windows x64 CUDA 13.3, plus AMD/Vulkan/CPU-only.
Intel SYCL needs a source build. NVFP4 quants need Blackwell for native FP4 (dequant fallback otherwise).

Upstream run shape (chat template ships inside the GGUF, thinking + tool calling included):

```bash
./llama-cli -m AD-Q5_K_M/Ling-3.0-flash-AD-Q5_K_M-00001-of-00002.gguf \
  --jinja -ngl 99 -c 32768
```

- Author sampling: **temp 0.6 · top_p 0.95 · top_k 20**.
- **`AD-*` files are the production rungs**; `*_STOCK` / `*_FLAT` are control files published to show what the
  AD strategy buys, *not* meant for real use. 64 GB rung is `AD-IQ3_M` (62.2 GB, KLD 0.0481).
- Known upstream gap: the multi-token prediction block is not wired up.

3060 hybrid log: [../models/ling-3.0-flash.md](../models/ling-3.0-flash.md).

## Build note

Some CUDA builds used:

```text
GGML_CUDA_FA_ALL_QUANTS
```
