# GitHub Pages reports

`docs/` is the Pages root (Settings → Pages → `main` / `/docs`). Reports live under `reports/`:

```text
docs/
  index.html                           ← listing
  reports/ornith-1.5-35b/index.html    ← this report
  reports/<future-model>/index.html
```

Live:

- Listing: https://ajay9o9.github.io/runs-on-a-3060/
- Ornith: https://ajay9o9.github.io/runs-on-a-3060/reports/ornith-1.5-35b/

To add another model: put `index.html` in `docs/reports/<slug>/` and add a card on `docs/index.html`. Keep the markdown bench log in `models/`.
