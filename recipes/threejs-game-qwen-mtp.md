# Use case: Three.js game development (Qwen 35B MTP Q6)

Local coding / game-dev sessions for a **Three.js** project on the lab **RTX 3060 12GB**.  
Model: `Qwen3.6-35B-A3B-MTP-UD-Q6_K_XL.gguf`  
**KV:** `-ctk q8_0 -ctv q8_0`  
**Context:** `-c 132768`  
**Host:** 64 GB RAM (hybrid MoE experts on CPU)

## llama.cpp-style server (logged command)

Runtime family: llama.cpp with MTP (`--spec-stage mtp:n_max=4` in this log).

```bash
./build/bin/llama-server \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-MTP-UD-Q6_K_XL.gguf \
  -c 132768 \
  -ngl 99 --n-cpu-moe 36 \
  -fa 1 -t 12 --no-mmap \
  -ctk q8_0 -ctv q8_0 \
  --jinja \
  --chat-template-kwargs '{"preserve_thinking":true}' \
  --temp 0.6 --top-p 0.95 --top-k 20 --min-p 0.0 \
  --presence-penalty 0.0 --repeat-penalty 1.0 \
  --spec-stage mtp:n_max=4 \
  --metrics
```

| Flag | Role in this setup |
|------|---------------------|
| `-c 132768` | long context for code + game state / multi-file work |
| `-ngl 99 --n-cpu-moe 36` | hybrid MoE; many experts on CPU/RAM |
| `-ctk q8_0 -ctv q8_0` | KV cache |
| `--no-mmap` | as logged |
| `--spec-stage mtp:n_max=4` | MTP draft depth 4 |
| `--jinja` + `preserve_thinking` | chat template / thinking behavior |
| sampling flags | as used for interactive coding |

Source vault note: `three js game.md`.

## Same model on ik_llama.cpp (better lab numbers)

Same GGUF was also run on **[ik_llama.cpp](https://github.com/ikawrakow/ik_llama.cpp)**.  
Lab bench comparison (not the full game server line, but same weights + similar hybrid offload + same KV):

**Shared flags:** `-ngl 99 --n-cpu-moe 32 -fa 1 -t 12 -ctk q8_0 -ctv q8_0`, model MTP-UD-Q6_K_XL.

| Runtime | test | t/s |
|---------|------|----:|
| **ik_llama** | pp512 | **821.16 ± 31.39** |
| llama.cpp (e-mtp build) | pp512 | 472.57 ± 7.47 |
| **ik_llama** | tg256 | **46.44 ± 1.05** |
| llama.cpp (e-mtp build) | tg256 | 45.33 ± 0.24 |
| **ik_llama** | pp65536 | **713.26 ± 4.78** |
| llama.cpp | pp65536 | 412.56 ± 3.81 |
| **ik_llama** | tg256 (w/ 64k prefill path) | **47.41 ± 0.13** |
| llama.cpp | tg256 (w/ 64k) | 45.74 ± 0.32 |

Prefill is where ik_llama pulled ahead hard on this model; gen was slightly higher too.

### ik_llama server line (related log)

Different offload/MTP flag spelling than the Three.js command, same model + 132k context class:

```bash
./build/bin/llama-server \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-MTP-UD-Q6_K_XL.gguf \
  -c 132768 \
  -ngl 5 --n-cpu-moe 32 \
  -fa 1 -t 12 --no-mmap \
  -ctk q8_0 -ctv q8_0 \
  --jinja \
  --chat-template-kwargs '{"preserve_thinking":true}' \
  --temp 0.6 --top-p 0.95 --top-k 20 --min-p 0.0 \
  --presence-penalty 0.0 --repeat-penalty 1.0 \
  -mtp --draft-max 3 --draft-p-min 0.0
```

Lab practice for the game: **same server command shape on ik_llama when possible** → better interactive feel / prefill vs stock llama.cpp on this Q6 MTP GGUF (see table above).

## Related

- [../models/qwen3.6-35b-a3b.md](../models/qwen3.6-35b-a3b.md) — Qwen 35B benches + MTP sweeps  
- [../runtimes/comparison.md](../runtimes/comparison.md) — runtime side-by-side  
- [../techniques/mtp.md](../techniques/mtp.md)  
- [../RESULTS.md](../RESULTS.md)
