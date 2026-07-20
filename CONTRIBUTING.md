# Contributing

Submit measured runs. Keep tone factual (command + numbers). No essays required.

## Required fields

1. GPU model + VRAM  
2. System RAM size  
3. CPU + `-t`  
4. Runtime name (+ commit/build if known)  
5. Exact command  
6. **KV cache: `-ctk` and `-ctv` (or type_k / type_v)** — required for LLM runs  
7. Results: pp/tg (or image time/steps/resolution), peak VRAM if available  
8. Model filename + quant (+ HF link if public)  

LLM results **without** KV types will be marked incomplete or rejected.

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
... -ctk ... -ctv ...
```

### KV
- ctk / type_k:
- ctv / type_v:

### Results
| test | t/s | VRAM |
|------|-----|------|
| pp4096 | | |
| tg128 | | |
```

## Scope

- Prefer RTX 3060 12GB or other 12GB cards; label other GPUs clearly  
- Keep LLM vs image paths separate  
- No secrets, private tokens, or huge binaries  

KV reference: [techniques/kv-cache.md](techniques/kv-cache.md).
