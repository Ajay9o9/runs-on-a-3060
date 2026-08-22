# MiniMax H3 on RTX 3060 12GB

Text-to-**video with synchronised audio in one pass** (not llama.cpp). 24 fps
native, 32 kHz stereo. Runs under ComfyUI.

**Workflows / setup:** [Ajay9o9/minimax-h3-30series](https://github.com/Ajay9o9/minimax-h3-30series)
**Stack:** ComfyUI + [ComfyUI-MultiGPU](https://github.com/pollockjj/ComfyUI-MultiGPU) (DisTorch2) + [ComfyUI-KJNodes](https://github.com/kijai/ComfyUI-KJNodes) + [ComfyUI-MiniMax-H3-Turbo](https://github.com/Larryvrh/ComfyUI-MiniMax-H3-Turbo)

## Models

Comfy-Org pruned **INT8 ConvRot** builds — already quantized, not fp16.

| File | Repo | Size |
|------|------|-----:|
| `diffusion_models/minimax_h3_fl2va_pruned_int8_convrot.safetensors` | [Comfy-Org/MiniMax-H3](https://huggingface.co/Comfy-Org/MiniMax-H3) | 19.53 GiB |
| `text_encoders/qwen3vl_32b_minimax_h3_int8_convrot.safetensors` | same | 25.28 GiB |
| `vae/minimax_h3_video_vae_fp16.safetensors` | same | 4.85 GiB |
| `vae/minimax_h3_audio_vae_fp32.safetensors` | same | 577 MiB |
| `loras/minimax_h3_turbo_4step_ema_ckpt850.safetensors` | [larryvrh/MiniMax-H3-Turbo-Lora](https://huggingface.co/larryvrh/MiniMax-H3-Turbo-Lora) | 744 MiB |

**~51 GiB total.** 45 GiB of that is DiT + text encoder, against 11.6 GiB of
usable VRAM.

## Hardware context

| Spec | Value |
|------|-------|
| GPU | RTX 3060 12GB (~11.6 GiB usable, ~0.5 GiB to desktop) |
| System RAM | **64 GB** — this is the binding constraint, not VRAM |
| PCIe | 4.0 x16 |
| ComfyUI | 0.31.0 · torch 2.13.0+cu130 · driver 595.84 |
| Sampler | Turbo LoRA, 4 steps, cfg 1.0 (distilled — aborts above 1.0) |

The text encoder (25.3 GiB) can never fit a 12 GB card and lives in system RAM
permanently. Smaller cards do **not** need less RAM — they stream more of the
same files, so the RAM requirement goes *up* as VRAM goes down.

## The one knob

DisTorch2 byte-expert allocation — a **per-model quota**, not a device capacity:

```text
DiT loader           cuda:0,5gb;cpu,*
text encoder loader  cuda:0,8gb;cpu,*
```

`5gb` resident on the 3060 (vs `13gb` on a 3090). Everything above that streams
from system RAM each step. All compute is on `cuda:0`; MultiGPU is a weight
shelf, not tensor parallelism.

**Failure datum** — running the 3090 preset (`cuda:0,13gb`, 1344×768/124) on the
3060 OOMs at `video_patch_proj`:

```text
Currently allocated: 10.25 GiB · Requested: 4.27 GiB · Device limit: 11.61 GiB
```

That 4.27 GiB is a rare true *activation demand* reading — on a 24 GB card
ComfyUI's allocator expands to fill free VRAM, so peak-used tells you nothing.

## Timed runs (lab log)

Shape 864×480 · 243 frames · 24 fps · 10.125 s · 32 kHz stereo. Times are
ComfyUI's own `Prompt executed in`.

| Run | Wall |
|-----|-----:|
| Baseline single run | **6:47** |
| 5-clip batch (anime / gameplay / FOV / cyberpunk / third-person) | ~31.6 min total |
| — per clip average | ~**6:10** |
| — `03-fov` | 6:16 |

All five verified by ffprobe at 864×480, 24/1 fps, 10.125 s, audio 32 kHz stereo.

**Compare shapes, not cards.** 480p/10 s on this 3060 (6:47) beats 1344×768/10 s
on a 3090 (9:31) — it is doing a third of the pixel-frames. The card decides your
resolution, not whether the model runs.

## Not tested here

- Full resolution (1344×768) on 12 GB — the tuner refuses it without `--force`
- 15 s clips (362 frames)
- SeedVR2 upscale on 12 GB — wants ~15 GiB, does not fit
- 16 GB or 32 GB system RAM (64 GB is what was measured)
- GGUF quants ([unsloth/MiniMax-H3-GGUF](https://huggingface.co/unsloth/MiniMax-H3-GGUF)) — would cut the RAM floor, unverified

## Summary

| Metric | Value |
|--------|------:|
| Shape | 864×480 · 10.125 s · 24 fps + audio |
| Resident VRAM quota | 5 GiB |
| Wall time | ~6:10–6:47 |
| System RAM floor | 64 GB |
