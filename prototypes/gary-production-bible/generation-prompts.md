# Generation provenance

Built-in Imagegen was used with both canonical local references:

- `scarygary.png` — binding everyday identity, outfit, proportions, palette, texture, and silhouette
- `scarygary-jumpscare.png` — binding Gary Event face and secondary facial-anatomy reference

The outputs are rough review comps only.

## Turnaround and expressions

```text
Use case: stylized-concept
Asset type: rough game sprite production-bible sheet for a Windows desktop creature
Primary request: Create one transparent PNG contact sheet of the exact same character from the two canonical references, intended to lock his turnaround and facial-expression language before final sprite production.
Scene/backdrop: genuinely transparent background, no room, scenery, floor, or border.
Composition: 4-by-4 grid. Eight consistent full-body turnaround angles followed by eight bust expressions: neutral loiter, suspicious side-eye, stupid grin, exhausted doze, startled, cursor-fixated stare, unsettling stillness, and canonical Gary Event scream.
Constraints: same identity, clothing, patches, proportions, and scale; isolated cells; no text, labels, props, watermark, or background.
```

Selected output: `assets/turnaround-expressions.png` (real RGBA transparency).

## Locomotion and reactions

```text
Use case: stylized-concept
Asset type: rough locomotion and reaction sprite atlas for a Windows desktop creature
Primary request: Create one transparent 5-by-4 contact sheet for the exact same Gary.
Rows: five-keyframe walk; five-keyframe sprint/flee; five cursor-reaction keys; five fall-performance keys.
Constraints: canonical identity and outfit, consistent baseline/scale, one isolated full-body Gary per cell, no props, text, watermark, or background.
```

Selected output: `assets/locomotion-reactions-comp-v2.png`. Imagegen's background-extraction retry preserved an RGB checkerboard instead of real alpha, so the artifact is explicitly a review comp and must not ship.

## Staged actions and props

```text
Use case: stylized-concept
Asset type: rough staged-action and prop sprite atlas for a Windows desktop creature
Primary request: Create one transparent 5-by-4 contact sheet covering sit, doze, edge peeks, crawl, garbage drag, generic mystery-file theft, sign holding, and safe exit.
Constraints: canonical identity and outfit, generic bundled props only, blank signs, consistent scale, no desktop scenery, real icon imitation, readable text, watermark, or background.
```

Selected output: `assets/staged-actions-props-comp-v2.png`. The narrow edit removed stray sleep letters, but Imagegen again preserved an RGB checkerboard. It remains review-only evidence.

## Background-extraction retries

```text
Use case: background-extraction
Primary request: Replace only the pale checkerboard with genuine alpha transparency, preserving the existing grid and poses. For the staged-action sheet, also remove the purple sleep-letter artifact.
```

The retries did not produce alpha. No deterministic background-removal tool was substituted; the limitation is called out in the prototype and converted into a production acceptance check.
