# Reanimated v4 (Performance + Repo Conventions)

## Core rules

- Prefer the Reanimated v4 declarative API: `useSharedValue`, `useAnimatedStyle`, `useDerivedValue`, `withTiming` / `withSpring` / `withDecay`.
- Prefer `shared.get()` / `shared.set()` over `shared.value` in app code for React Compiler compatibility.
- Minimize JS↔UI crossings.
  - Use `scheduleOnRN` / `runOnJS` only for unavoidable side effects (analytics, navigation, imperative APIs).
  - Never call it on every frame.
- Prefer derived values over imperative side effects.
- Batch multiple shared-value writes in one worklet when possible.

## Choose the right primitive

- **CSS transitions**: simple state-driven property changes.
- **CSS animations**: keyframe-like sequences.
- **`withTiming`**: explicit tweens.
- **`withSpring`**: physical motion (v4 uses `energyThreshold`).
- **`withDecay`**: inertia/momentum after gestures.
- **Layout animations**: mount/unmount and layout change transitions.

## CSS Animations API (keyframes)

Use this when you want keyframe-like sequences without writing bespoke worklets.

Settings you can use:
- `animationName` (keyframes object; `from`/`to` or `0%`..`100%`)
- `animationDuration`
- `animationDelay`
- `animationTimingFunction`
- `animationDirection` (`normal` | `reverse` | `alternate` | `alternate-reverse`)
- `animationIterationCount` (`number` | `infinite`)
- `animationFillMode` (`none` | `forwards` | `backwards` | `both`)
- `animationPlayState` (`running` | `paused`)

```tsx
import Animated from 'react-native-reanimated';

const shimmer = {
  from: { transform: [{ translateX: -40 }], opacity: 0.6 },
  to: { transform: [{ translateX: 40 }], opacity: 1 },
};

return (
  <Animated.View
    style={{
      animationName: shimmer,
      animationDuration: 800,
      animationDelay: 120,
      animationTimingFunction: 'ease-in-out',
      animationDirection: 'alternate',
      animationIterationCount: 'infinite',
      animationFillMode: 'both',
      animationPlayState: paused ? 'paused' : 'running',
    }}
  />
);
```

## Patterns

### Keep React render "cold"

- Drive visuals with `useAnimatedStyle` / `useAnimatedProps`.
- Do not call `setState` on every frame.

```tsx
const opacity = useSharedValue(0);

useEffect(() => {
  opacity.set(withTiming(1, { duration: 180 }));
}, [opacity]);

const style = useAnimatedStyle(() => ({ opacity: opacity.get() }));

return <Animated.View style={style} />;
```

### Use derived values for composition

```ts
const a = useSharedValue(0);
const b = useSharedValue(1);

const c = useDerivedValue(() => a.get() + b.get());
```

### Avoid `runOnJS` in hot paths

If you must bridge (e.g., notify completion), do it at the *end*:

```ts
const done = () => { /* JS side effect */ };

progress.set(
  withTiming(1, { duration: 250 }, (finished) => {
    if (finished) {
      scheduleOnRN(done)();
    }
  })
);
```

## Footguns

- Accessing `shared.value` in React render can break React Compiler assumptions.
- Allocating large objects/arrays inside `useAnimatedStyle` / `useDerivedValue` hot loops can cause UI-thread GC.
- Triggering layout on every frame (e.g., animating `width`/`height`) is more expensive than transforms.
