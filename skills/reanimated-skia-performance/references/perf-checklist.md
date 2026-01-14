# Animation / 2D Graphics Performance Checklist

## First-pass questions

- Is the animation driven on the UI thread (worklets / `SharedValue`), or is it using React state?
- Is there any JS↔UI crossing happening per frame (`runOnJS`/`scheduleOnRN`)?
- Are we allocating large objects/arrays/paths/images per frame?
- Are we accidentally triggering layout on every frame?
- Is jank only in dev? Check dev-only tooling (sliders, overlays, logs) and JS thread load.

## Reanimated checklist

- Use `SharedValue` + `useAnimatedStyle` / `useDerivedValue`.
- Prefer transforms (translate/scale/rotate) over layout properties when possible.
- Avoid `runOnJS` in `onUpdate`.
- Batch related writes to shared values.

## Skia checklist

- Memoize `RuntimeEffect`, `Path`, `Paint`, images.
- Keep the scene graph shallow.
- Avoid heavy effects over large areas (blur, masks, layers).
- Prefer animated props / uniforms over React re-renders.
- Use `debug` on Skia views (when available) to see average render times.

## Shader checklist

- Ensure every uniform declared in SkSL is provided.
- Keep uniforms small.
- Normalize coordinates in-shader (`uv = xy / resolution`).
- Avoid branching and unbounded loops.

## Common root causes of jank

- JS thread stalls (network parsing, heavy JSON, logging, synchronous work).
- Too many React renders during gestures.
- Per-frame allocations triggering GC.
- Layout thrash (animating width/height/top/left).
- Overdraw / expensive GPU ops (large blurs, masks, layers).

