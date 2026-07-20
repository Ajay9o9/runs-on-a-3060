# Contributing results

We want **reproducible** 12GB-class numbers, not vibes.

## Minimum for a PR / issue

1. **Hardware**
   - GPU model + VRAM  
   - **System RAM size + speed if known**  
   - CPU model + threads used (`-t`)  
2. **Runtime**
   - llama.cpp / ik_llama / tq3 / other + commit or build number if possible  
3. **Exact command** (copy-paste)  
4. **Results**
   - pp t/s, tg t/s (or image: seconds, steps, resolution)  
   - Peak VRAM (`nvidia-smi` or llama memory breakdown)  
5. **Model filename** + quant + HF link if public  

## Template

```markdown
### Hardware
- GPU:
- RAM:
- CPU:
- OS:

### Runtime
- name / commit:

### Command
```bash
...
```

### Results
| test | t/s | VRAM |
|------|-----|------|
| pp4096 | | |
| tg128 | | |

### Notes
```

## Scope

- Prefer **RTX 3060 12GB** or other **12GB** cards.  
- Other VRAM sizes welcome if labeled clearly.  
- Keep **LLM** vs **image** sections separate.  
- Do not commit private tokens, local absolute home paths with usernames, or huge binaries.
