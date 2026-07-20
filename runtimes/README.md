# runtimes/

Builds used for the logs. Not limited to upstream llama.cpp.

| Runtime | Repo | Used for |
|---------|------|----------|
| llama.cpp | [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) | Gemma, general GGUF, server, MTP branches |
| ik_llama.cpp | [ikawrakow/ik_llama.cpp](https://github.com/ikawrakow/ik_llama.cpp) | Qwen3.6 Unsloth Q6, expert offload |
| llama.cpp-tq3 | [turbo-tan/llama.cpp-tq3](https://github.com/turbo-tan/llama.cpp-tq3) | TQ3_4S, `ctv tq3_0` |
| TurboQuant branch | community | `turbo4` / `turbo3` KV on Q4_K_XL |
| llama.cpp MTP | branch / PR builds | Qwen + Gemma MTP |
| llama-diffusion-gemma | diffusion CLI | Diffusion Gemma 26B-A4B |
| llama-benchy | HTTP client | long-context server benches |

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
