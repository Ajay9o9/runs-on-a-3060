# GitHub Pages dashboards

This `docs/` folder is the Pages site. One HTML report per model:

```text
docs/
  index.html                 ← listing
  ornith-1.5-35b/index.html  ← this report
  <future-model>/index.html
```

Live (after Pages is on, source **main** / **docs**):

- Listing: https://ajay9o9.github.io/runs-on-a-3060/
- Ornith: https://ajay9o9.github.io/runs-on-a-3060/ornith-1.5-35b/

To add another model: drop `index.html` in `docs/<slug>/` and add a card on `docs/index.html`. Keep the markdown bench log in `models/`.
