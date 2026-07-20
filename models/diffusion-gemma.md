# Diffusion Gemma 26B-A4B on RTX 3060

**Not image generation.** This is a **text** diffusion language model (denoising tokens), run via `llama-diffusion-gemma-cli`.

## Model

| File | Quant |
|------|-------|
| `diffusiongemma-26B-A4B-it-Q4_K_M.gguf` | Q4_K_M |

## Hardware footprint

| Metric | Value |
|--------|-------|
| GPU | RTX 3060 12GB |
| Peak VRAM | **10450 MB** (all step counts, `-ngl 15`) |
| Offload | partial GPU layers (`-ngl 15`) — rest relies on system RAM (**64 GB** host) |

## Command

```bash
./build/bin/llama-diffusion-gemma-cli \
  -m $MODEL_DIR/diffusiongemma-26B-A4B-it-Q4_K_M.gguf \
  -ngl 15 \
  --diffusion-steps N \
  -n 512 \
  -p "Write a Python quicksort implementation"
```

## Step sweep (512 answer tokens target)

| steps | canvas tok/s | wall (512 canvas) | s/step | notes |
|------:|-------------:|------------------:|-------:|-------|
| 12 | **42.4** | 12.08 s | ~0.503 | |
| 18 | 28.6 | 17.93 s | ~0.498 | |
| 24 | 21.2 | 24.16 s | ~0.503 | one run short answers |
| 28 | 18.9 | 27.04 s | ~0.510 | |
| 32 | 16.0 | 32.02 s | ~0.500 | |
| 38 | 13.4 | 38.20 s | ~0.503 | |
| 42 | 12.5 | 40.84 s | ~0.504 | |
| 48 | 13.8 | 37.15 s | ~0.502 | early-stop / shorter answer in log |

Prefill on short prompt stayed ~**32–35 tok/s**.

## Summary

| Metric | Value |
|--------|-------|
| Peak VRAM @ ngl 15 | ~10450 MB |
| s/step (lab) | ~0.5 |

Not image gen. See [Bonsai Image](../image/bonsai-image-4b.md) for image models.
