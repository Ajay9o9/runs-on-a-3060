# Bonsai Image 4B (PrismML) on RTX 3060

Local **text-to-image** model (not llama.cpp). Ternary (~1.58-bit) and binary (1-bit) variants via [Bonsai-image-demo](https://github.com/PrismML-Eng/Bonsai-image-demo) + gemlite GPU backend.

## Models

| HF / local name | Role |
|-----------------|------|
| `prism-ml/bonsai-image-ternary-4B-gemlite-2bit` | Recommended quality (ternary) |
| `prism-ml/bonsai-image-binary-4B-gemlite-1bit` | Smaller / lighter |

Local studio layout (after download):

```text
models/bonsai-image-4B-ternary-gemlite/
models/bonsai-image-4B-binary-gemlite/
```

## Hardware context

| Spec | Value |
|------|-------|
| GPU | RTX 3060 12GB |
| System RAM | **64 GB** (studio + OS; image path is mostly VRAM bound) |
| Lab peak VRAM (generate) | **~6585 MB** |
| Lab gen time (example prompts) | **~9.7 s** total, **~2.4 s/step** |

Plenty of VRAM headroom on 12GB at the resolutions tested (~768×512 class defaults in tooling).

## How we ran it

1. Clone + setup studio:
   ```bash
   git clone https://github.com/PrismML-Eng/Bonsai-image-demo.git
   cd Bonsai-image-demo
   ./setup.sh
   ./scripts/download_model.sh ternary   # or binary
   ./scripts/serve.sh                    # backend + frontend
   ```

2. HTTP generate (local skill / CLI pattern):

```bash
# defaults used in lab tooling
# backend: bonsai-ternary-gemlite
# port: 8001
python bonsai_generate.py \
  "thick forest at night with fireflies" \
  --steps 8 --width 768 --height 512 \
  --backend bonsai-ternary-gemlite
```

API shape:

```http
POST http://127.0.0.1:8001/generate
{
  "prompt": "...",
  "steps": 6,
  "width": 768,
  "height": 512,
  "backend": "bonsai-ternary-gemlite"
}
```

Hermes `/img` style parameters also used: `steps`, `width`, `height`, `guidance`, `seed`, `backend=ternary|binary`.

## Timed runs (lab log)

| Prompt style | Total time | Avg step | Peak VRAM |
|--------------|------------|----------|-----------|
| Elden Ring style | **9.7 s** | **2.4 s** | **6585 MB** |
| Breath of the Wild style | **9.7 s** | **2.4 s** | **6585 MB** |

(Same timing/VRAM across those two prompts in the log.)

## Compare ternary vs binary

Studio/frontend supports backend swap; `/compare` style workflow hits both for side-by-side quality vs speed/size.

## Summary

| Metric | Value |
|--------|-------|
| Peak VRAM (lab gens) | ~6585 MB |
| Example wall time | ~9.7 s |

Not the same as [Ternary Bonsai 27B LLM](../models/bonsai-ternary-27b.md).
