# Contributing

Submit measured runs. Keep tone factual (command + numbers). No essays required.

## Required fields

1. GPU model + VRAM  
2. System RAM size  
3. CPU + `-t`  
4. Runtime name (+ commit/build if known)  
5. Exact command  
6. Results: pp/tg (or image time/steps/resolution), peak VRAM if available  
7. Model filename + quant (+ HF link if public)  

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
```

## Scope

- Prefer RTX 3060 12GB or other 12GB cards; label other GPUs clearly  
- Keep LLM vs image paths separate  
- No secrets, private tokens, or huge binaries
