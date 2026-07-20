# MTP / speculative decoding on RTX 3060

## What we tried

- llama.cpp **MTP / draft-mtp** branches  
- ik_llama MTP experiments  
- Models: Qwen3.6 35B MTP GGUFs, Gemma 4 12B + MTP draft GGUF  

## Cost / benefit (Qwen MTP Q6, short prompts)

| Mode | Gen t/s (approx) | VRAM |
|------|------------------|------|
| no MTP | ~21 | ~5.9–6.0 GB (partial GPU experts) |
| n-max 2–3 | **~31–36** | ~8.0–10.5 GB depending on ngl/ncmoe |

**n-max too high** (5–6) often **hurt** accept rate and tok/s.

## Command pattern

```bash
./build/bin/llama-cli \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-MTP-UD-Q6_K_XL.gguf \
  -p "..." -n 128 \
  -ngl 5 -ncmoe 32 \
  -fa 1 -t 12 \
  -ctk q8_0 -ctv q8_0 \
  --spec-type mtp --spec-draft-n-max 3
```

Gemma draft-MTP server:

```bash
./build/bin/llama-server \
  -m $MODEL_DIR/gemma-4-12b-it-UD-Q5_K_XL.gguf \
  --model-draft $MODEL_DIR/gemma-4-12B-it-MTP-Q8_0.gguf \
  ... \
  --spec-type draft-mtp --spec-draft-n-max 4
```

## Caveats from lab

- Extra VRAM for draft path is real — leave headroom under 12GB.  
- Host **RAM** still holds most MoE weights.  
- Some runs saw crashes (`double free`) on experimental MTP builds — pin a known-good commit before relying on it.
