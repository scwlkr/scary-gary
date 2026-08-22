# Gary sprite and sound production bible prototype

> PROTOTYPE ONLY — review material for GitHub issue 7. Do not ship these generated contact sheets.

This standalone prototype tests one recommendation through three review lenses:

- `?variant=identity` — identity, turnaround, silhouette, expression, and palette
- `?variant=coverage` — complete behavior, prop, sign, and Gary Event asset coverage
- `?variant=production` — frame budgets, scale, alpha/export, naming, atlas, and sound rules

Open `index.html` directly in a browser. Use the bottom switcher or the left/right arrow keys to change lenses. No server, package install, or network connection is required.

The rough contact sheets were generated from the two canonical PNG references. Only `assets/turnaround-expressions.png` currently has real alpha. The two behavior sheets deliberately carry a visible `REVIEW COMP — RGB CHECKERBOARD` warning because the background-extraction attempts still baked the checkerboard into RGB. That failure is evidence for the hard production gate: final routine sprites must be genuine RGBA PNGs and pass black, white, and magenta matte checks.

## Recommendation under review

Keep canonical Gary's identity and hand-painted, pixel-edged texture; author at 4x; render everyday Gary at 205 logical pixels; use one screen-relative horizontal mirror rule for directional routines; keep signs and the Gary Event unmirrored; ship routine art as trimmed RGBA sprites plus pivot metadata; and reserve the canonical extreme face for the single Gary Event.

The generated sheets are visual prompts, not production sprites. A human artist should redraw the accepted direction into stable per-frame sources before implementation.
