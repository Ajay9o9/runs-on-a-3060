# KV cache (`-ctk` / `-ctv`)

KV cache type is part of every LLM result. Same model + offload with different K/V quants is **not** the same run.

llama.cpp flags:

| Flag | Meaning |
|------|---------|
| `-ctk TYPE` | K cache type |
| `-ctv TYPE` | V cache type |

Bench tables often print these as `type_k` / `type_v`.

## Types used in this lab

| type_k (`-ctk`) | type_v (`-ctv`) | Where used |
|-----------------|-----------------|------------|
| `q4_0` | `tq3_0` | Qwen TQ3_4S on **llama.cpp-tq3** |
| `turbo4` | `turbo3` | Qwen Q4_K_XL on [TheTom/llama-cpp-turboquant](https://github.com/TheTom/llama-cpp-turboquant) |
| `q8_0` | `q8_0` | Most Gemma 4 runs; many Qwen Q5/Q6 mainline/ik horizontal runs; MTP short-prompt sweeps |
| `q4_0` | `q4_0` | Some ik_llama Q5/Q6 with `-ot exps=CPU`; early MTP bench lines |

## Why it matters on 12GB

- Lower-bit / special KV (turbo, tq3) reduces **context VRAM**, so more room for layers/experts or longer `-c`.
- `q8_0`/`q8_0` is common and comparable across models, but hungrier at long context.
- Do not compare tg numbers across rows unless **KV matches** (or the difference is the point of the row).

## Required log fields

Every new LLM entry should include:

```text
-ctk <type> -ctv <type>
```

If a historical log omitted KV, the page marks it **not recorded**.

## Related

- [../runtimes/comparison.md](../runtimes/comparison.md) — same model, different runtime/KV
- [moe-offload.md](moe-offload.md) — offload + KV both affect VRAM
