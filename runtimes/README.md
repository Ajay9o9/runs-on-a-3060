# runtimes/

Builds used for the logs.

| Runtime | Repo / branch | Used for |
|---------|---------------|----------|
| llama.cpp | [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) | Gemma, general GGUF, server, MTP branches |
| ik_llama.cpp | [ikawrakow/ik_llama.cpp](https://github.com/ikawrakow/ik_llama.cpp) | Qwen3.6 Unsloth Q6, expert offload |
| llama.cpp-tq3 | [turbo-tan/llama.cpp-tq3](https://github.com/turbo-tan/llama.cpp-tq3) | TQ3_4S, `ctv tq3_0` |
| llama-cpp-turboquant | [TheTom/llama-cpp-turboquant](https://github.com/TheTom/llama-cpp-turboquant) | `turbo4` / `turbo3` KV on Q4_K_XL |
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

## Build note

Some CUDA builds used:

```text
GGML_CUDA_FA_ALL_QUANTS
```
