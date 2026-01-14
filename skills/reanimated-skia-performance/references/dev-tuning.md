# Dev-mode tuning (sliders)

Use a dev-only tuning panel to iterate on animation configs (spring/timing) and shader uniforms without touching code.

## Rules

- Render tuning UI only in `__DEV__`.
- Keep tuning UI in a separate component so slider state changes do not re-render the animated scene.
- Store tunables in `SharedValue`s; update them from slider callbacks via `.set()`.
- For Skia: feed `SharedValue`s into uniforms via `useDerivedValue` (worklet) so updates stay on the UI thread.
- For Reanimated config: treat slider changes as *config updates* and re-trigger the animation when you want to apply them.

## Minimal building block: `DevSlider`

This is intentionally JS-driven (fine in dev). It updates a `SharedValue<number>`.

```tsx
import { memo, useState } from 'react';
import { Text, View } from 'react-native';
import Slider from '@react-native-community/slider';
import type { SharedValue } from 'react-native-reanimated';

export const DevSlider = memo(function DevSlider(props: {
  label: string;
  value: SharedValue<number>;
  min: number;
  max: number;
  step?: number;
}) {
  const { label, value, min, max, step = 0.01 } = props;
  const [display, setDisplay] = useState(() => value.get());

  return (
    <View style={{ gap: 8 }}>
      <Text>{label}: {display.toFixed(3)}</Text>
      <Slider
        minimumValue={min}
        maximumValue={max}
        step={step}
        value={value.get()}
        onValueChange={(v) => {
          setDisplay(v);
          value.set(v);
        }}
      />
    </View>
  );
});
```

## Tune a spring config

Goal: tune `withSpring` parameters without rewriting code.

Pattern:
- Store tunables in shared values.
- Re-trigger the spring when you want to apply the new config.

```tsx
import { Pressable, View } from 'react-native';
import Animated, { useAnimatedStyle, useSharedValue, withSpring } from 'react-native-reanimated';

const progress = useSharedValue(0);

// Tunables
const damping = useSharedValue(18);
const stiffness = useSharedValue(220);

const start = () => {
  // OK on JS: tunables update rarely and this is dev tooling.
  progress.set(
    withSpring(progress.get() === 0 ? 1 : 0, {
      damping: damping.get(),
      stiffness: stiffness.get(),
    })
  );
};

const style = useAnimatedStyle(() => ({
  transform: [{ translateY: (1 - progress.get()) * 40 }],
}));

return (
  <>
    <Animated.View style={[{ height: 60, width: 60, backgroundColor: 'black' }, style]} />

    {__DEV__ && (
      <View style={{ padding: 12, gap: 12 }}>
        <DevSlider label="damping" value={damping} min={1} max={40} step={1} />
        <DevSlider label="stiffness" value={stiffness} min={50} max={600} step={10} />
        <Pressable onPress={start}><Animated.Text>Restart</Animated.Text></Pressable>
      </View>
    )}
  </>
);
```

Notes:
- If you try to restart the spring on every slider tick, it can feel "sticky"; prefer a dedicated restart button, or use `onSlidingComplete`.

## Tune a shader uniform

Goal: adjust shader parameters continuously and cheaply.

Pattern:
- Keep the runtime effect compiled once (`useMemo`).
- Create `SharedValue` tunables.
- Build `uniforms` via `useDerivedValue` and pass it to `<Shader uniforms={uniforms} />`.

```tsx
import { useMemo } from 'react';
import { View } from 'react-native';
import { Canvas, Fill, Paint, Shader, Skia, type Uniforms } from '@shopify/react-native-skia';
import { useDerivedValue, useSharedValue } from 'react-native-reanimated';

const intensity = useSharedValue(0.6);
const speed = useSharedValue(0.002);

const sksl = `
uniform float2 u_resolution;
uniform float u_time;
uniform float u_intensity;
uniform float u_speed;

half4 main(float2 xy) {
  float2 uv = xy / u_resolution;
  float t = u_time * u_speed;
  float v = 0.5 + 0.5 * sin(t + uv.x * 10.0);
  v = mix(v, v * v, u_intensity);
  return half4(v, v * 0.6, 1.0 - v, 1.0);
}
`;

const effect = useMemo(() => Skia.RuntimeEffect.Make(sksl), []);
if (!effect) return null;

const uniforms = useDerivedValue<Uniforms>(() => ({
  u_resolution: Skia.Point(width, height),
  u_time: performance.now(),
  u_intensity: intensity.get(),
  u_speed: speed.get(),
}));

return (
  <>
    <Canvas style={{ width, height }}>
      <Fill>
        <Paint>
          <Shader source={effect} uniforms={uniforms} />
        </Paint>
      </Fill>
    </Canvas>

    {__DEV__ && (
      <View style={{ padding: 12, gap: 12 }}>
        <DevSlider label="intensity" value={intensity} min={0} max={1} step={0.01} />
        <DevSlider label="speed" value={speed} min={0.0005} max={0.01} step={0.0005} />
      </View>
    )}
  </>
);
```

Notes:
- Prefer a real clock (`useClock()` from Skia externals) over `performance.now()` if you want the shader time to advance on the UI thread.
- Keep uniforms small; prefer numbers/vectors/`Float32Array`.

