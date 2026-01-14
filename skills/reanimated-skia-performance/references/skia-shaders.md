# Skia (Canvas + Shaders) Performance Patterns

## Principles

- Prefer passing animated values to Skia props over re-rendering React every frame.
  - Skia prop types support `T | { value: T }` (Reanimated `SharedValue<T>` matches this shape).
  - Pass the `SharedValue` object itself (do not pass `shared.value`).
- Memoize heavy Skia objects (`Path`, `Paint`, `RuntimeEffect`, images) with `useMemo`.
- Separate static vs dynamic content.
  - Keep static drawings as a `SkPicture` (record once) or render them into a texture.
  - Animate only the minimal dynamic layer.
- Avoid expensive effects in large areas (blur, masks, `saveLayer`) unless necessary.

## Runtime effects (SkSL shaders)

### Compile once

```ts
const effect = useMemo(() => Skia.RuntimeEffect.Make(sksl), []);
if (!effect) return null; // handle compile failures
```

### Keep uniforms complete and stable

- Every `uniform` declared in SkSL must exist in the `uniforms` object.
- Prefer small uniform payloads.
- For big numeric payloads, prefer `Float32Array` and update it in-place when possible.

### Animate uniforms on the UI thread

- Drive uniforms with Reanimated (`useDerivedValue`) or Skia's `useClock()` helper.
- Treat `useClock()` as milliseconds since first frame.

```tsx
const clock = useClock();

const uniforms = useDerivedValue<Uniforms>(() => ({
  u_time: clock.get(),
  u_resolution: Skia.Point(width, height),
  u_intensity: intensity.get(),
}));

return <Shader source={effect} uniforms={uniforms} />;
```

### Prefer normalization inside the shader

- Keep shader inputs unitless when possible (`uv = xy / resolution`).
- Avoid branching and unbounded loops.

## Canvas scene performance

- Keep the node tree shallow.
  - Prefer grouping transforms with `<Group transform={...}>` instead of many tiny nodes.
- Avoid per-frame path reconstruction.
  - Create the path once and mutate/transform via animated props when possible.
- Avoid allocating large arrays per frame.
  - For many points/particles, use Skia buffer helpers (e.g., `usePointBuffer`, `useRSXformBuffer`) to write into a mutable array.

## Interpolation helpers

- Use `usePathInterpolation()` when morphing between compatible paths.
- Use `useVectorInterpolation()` for point/anchor interpolation.

These helpers are in `@shopify/react-native-skia` reanimated externals and return `SharedValue`s suitable for Skia props.

