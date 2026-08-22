# video/

Video generation logs. Same machine: RTX 3060 12GB · 9900X · 64 GB RAM.

| Project | Page | Stack / repo |
|---------|------|--------------|
| MiniMax H3 (video + audio) | [minimax-h3.md](minimax-h3.md) | ComfyUI + [ComfyUI-MultiGPU](https://github.com/pollockjj/ComfyUI-MultiGPU) · workflows: [minimax-h3-30series](https://github.com/Ajay9o9/minimax-h3-30series) |

Unlike the LLM paths, weight streaming here is driven by **DisTorch2**
(ComfyUI-MultiGPU), not `-ncmoe`. Same idea, different knob — see
[techniques/moe-offload.md](../techniques/moe-offload.md) for the LLM equivalent.
