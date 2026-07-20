# Image generation on RTX 3060

Separate from the LLM section. Same test box: **RTX 3060 12GB** · 9900X · **64 GB RAM**.

| Project | Page | Stack | Notes |
|---------|------|-------|-------|
| **Bonsai Image 4B** (PrismML) | [bonsai-image-4b.md](bonsai-image-4b.md) | gemlite / studio, not llama.cpp | Ternary + binary; ~**6.6 GB** VRAM in lab gens |

## Not image gen (lives under models/)

- [Diffusion Gemma](../models/diffusion-gemma.md) — **text** diffusion  
- [Ternary Bonsai 27B](../models/bonsai-ternary-27b.md) — **text** LLM  

## Name collision warning

**Bonsai Image 4B ternary** ≠ **Ternary Bonsai 27B** LLM. Always say “image” or “27B LLM” in writeups / X posts.
