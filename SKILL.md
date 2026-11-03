---
name: no-use-effect
description: "ALWAYS ACTIVE. Enforces a strict no-direct-useEffect rule in React and React Native code. Never write useEffect directly — use derived state, event handlers, useSyncExternalStore, data-fetching libraries or use(), key-based resets, or useMountEffect instead. React Compiler aware: no useMemo/useCallback when the compiler is on. Triggers on all React component code: writing new components, editing existing ones, reviewing code, fixing bugs, or refactoring. Also triggers when user mentions useEffect, side effects, data fetching, state sync, or component lifecycle."
---

# No Direct useEffect

Never call `useEffect` directly. For the rare case of syncing with an external system on mount, use `useMountEffect()`.

```tsx
export function useMountEffect(effect: () => void | (() => void)) {
  /* eslint-disable react-hooks/exhaustive-deps, no-restricted-syntax */
  useEffect(effect, []);
}
```

The only place `useEffect` may appear directly is inside reusable custom hooks (like `useMountEffect` itself, or a `useData` hook when no fetching library is available). Components must never import or call `useEffect`.

## React Compiler Projects

Before reaching for manual memoization, check whether the project compiles with React Compiler: `reactCompiler: true` in `next.config.*`, `babel-plugin-react-compiler` in `package.json`, or project rules that ban `useMemo`/`useCallback`.

When the compiler is on:

- Never write `useMemo` or `useCallback`. The compiler memoizes derived values and functions for you.
- Derive expensive values inline (Rule 1) and write callback refs as plain functions.
- Project rules about memoization win over the examples in this skill.

Without the compiler, the `useMemo` and `useCallback` examples below apply as written.

## The 6 Rules

### Rule 1: Derive state, do not sync it

If you can calculate it from existing state/props, compute it inline.

```tsx
// BAD: two render cycles
const [filteredProducts, setFilteredProducts] = useState([]);
useEffect(() => {
  setFilteredProducts(products.filter((p) => p.inStock));
}, [products]);

// GOOD: one render
const filteredProducts = products.filter((p) => p.inStock);
```

For expensive calculations without React Compiler, use `useMemo`:

```tsx
const visibleTodos = useMemo(
  () => getFilteredTodos(todos, filter),
  [todos, filter]
);
```

With React Compiler, keep it inline. The compiler caches it:

```tsx
const visibleTodos = getFilteredTodos(todos, filter);
```

**Smell test:** You're about to write `useEffect(() => setX(f(y)), [y])` or state that only mirrors other state/props.

### Rule 2: Use data-fetching libraries

Effect-based fetching creates race conditions and reinvents caching.

```tsx
// BAD: race condition
useEffect(() => {
  fetchProduct(productId).then(setProduct);
}, [productId]);

// GOOD: library handles cancellation/caching/staleness
const { data: product } = useQuery(
  ['product', productId],
  () => fetchProduct(productId)
);
```

In React 19+, use the `use()` hook with `<Suspense>` to unwrap promises without `useEffect` or `useState`:

```tsx
// GOOD: use() + Suspense (React 19+)
function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);
  return <h1>{user.name}</h1>;
}
```

**Smell test:** Your effect does `fetch()` then `setState()`, or you're reimplementing caching/retries/cancellation.

### Rule 3: Event handlers, not effects

If a user action triggers it, do the work in the handler. Procedural logic (validate → transform → submit) belongs in handlers. Reconstructing procedural flow through ref+effect chains is a sign the logic was event-driven from the start.

```tsx
// BAD: flag-relay through effect
const [liked, setLiked] = useState(false);
useEffect(() => {
  if (liked) { postLike(); setLiked(false); }
}, [liked]);

// GOOD: direct
<button onClick={() => postLike()}>Like</button>
```

For shared logic between handlers, extract a function — not an effect:

```tsx
function buyProduct() {
  addToCart(product);
  showNotification(`Added ${product.name}`);
}
```

**Smell test:** State used as a flag so an effect can do the real action, or "set flag -> effect runs -> reset flag" mechanics.

### Rule 4: useMountEffect for one-time external sync

Only for true mount-time external system setup: DOM integration, third-party widgets, browser API subscriptions.

```tsx
// BAD: guard inside effect
useEffect(() => {
  if (!isLoading) playVideo();
}, [isLoading]);

// GOOD: conditional mounting
function VideoPlayerWrapper({ isLoading }) {
  if (isLoading) return <LoadingScreen />;
  return <VideoPlayer />;
}
function VideoPlayer() {
  useMountEffect(() => playVideo());
}
```

**Smell test:** Behavior is naturally "setup on mount, cleanup on unmount" with an external system.

### Rule 5: Reset with key, not dependency choreography

If you need "start fresh when ID changes," use React's remount semantics.

```tsx
// BAD: effect resets on ID change
useEffect(() => { loadVideo(videoId); }, [videoId]);

// GOOD: key forces clean remount
<VideoPlayer key={videoId} videoId={videoId} />

function VideoPlayer({ videoId }) {
  useMountEffect(() => { loadVideo(videoId); });
}
```

This also applies to resetting form state, clearing selections, etc. Use `key` on the component instead of an effect that sets state to initial values.

**Smell test:** Effect's only job is to reset local state when an ID/prop changes.

### Rule 6: Never patch a broken effect with a ref

If you need a `useRef` to stop an effect from double-firing, looping, or accessing stale state, the effect itself is the problem. Eliminate the root effect — don't bandage it with a ref.

```tsx
// BAD: ref guard hides the real problem
const hasRun = useRef(false);
useEffect(() => {
  if (hasRun.current) return;
  hasRun.current = true;
  showWelcomeToast(userId);
}, [userId]);

// GOOD: useMountEffect for true one-time setup
useMountEffect(() => {
  showWelcomeToast(userId);
});
```

```tsx
// BAD: ref to capture latest callback, dodging the dependency array
const onMessageRef = useRef(onMessage);
onMessageRef.current = onMessage;
useEffect(() => {
  const conn = createConnection(roomId);
  conn.on('message', (msg) => onMessageRef.current(msg));
  return () => conn.disconnect();
}, [roomId]);

// GOOD: useEffectEvent (React 19.2+) captures latest values automatically
const onMsg = useEffectEvent((msg) => {
  onMessage(msg);
});
useEffect(() => {
  const conn = createConnection(roomId);
  conn.on('message', onMsg);
  return () => conn.disconnect();
}, [roomId]);
```

The React docs explicitly warn: *"The right question isn't 'how to run an Effect once', but 'how to fix my Effect so that it works after remounting'."*

**Smell test:** You're adding `hasRun.current`, `isMounted.current`, or a ref whose sole purpose is controlling when/if an effect runs. Or you're storing a callback in a ref to avoid listing it as a dependency.

## Additional Patterns

### Notifying parents about state changes

Do not use an effect to call `onChange` — update both in the event handler, or lift state up:

```tsx
// BAD
useEffect(() => { onChange(isOn); }, [isOn, onChange]);

// GOOD: update both during the event
function updateToggle(nextIsOn) {
  setIsOn(nextIsOn);
  onChange(nextIsOn);
}
```

### useEffectEvent for latest values (React 19.2+)

When an effect needs to read the latest value of a prop/state without re-running when it changes, use `useEffectEvent` instead of a ref workaround:

```tsx
// BAD: ref to read latest theme without adding it as dependency
const themeRef = useRef(theme);
themeRef.current = theme;
useEffect(() => {
  logVisit(url, themeRef.current);
}, [url]);

// GOOD: useEffectEvent reads latest values automatically
const onVisit = useEffectEvent(() => {
  logVisit(url, theme);
});
useEffect(() => {
  onVisit();
}, [url]);
```

Call the function `useEffectEvent` returns only from inside effects. Never call it during render or pass it to other components.

Before React 19.2, `useEffectEvent` isn't available: prefer moving logic to event handlers or `useMountEffect`. If neither fits, isolate the ref workaround in a custom hook — never in a component.

### Callback refs for DOM side effects

For DOM measurement or setup, a callback ref is more reliable than `useRef` + `useMountEffect` — it fires exactly when the node attaches or detaches:

```tsx
// BAD: ref.current may be null on first render
const ref = useRef(null);
useMountEffect(() => {
  setHeight(ref.current.getBoundingClientRect().height);
});
return <div ref={ref}>Content</div>;

// GOOD: callback ref fires when node is attached
const measuredRef = useCallback((node: HTMLDivElement | null) => {
  if (node !== null) {
    setHeight(node.getBoundingClientRect().height);
  }
}, []);
return <div ref={measuredRef}>Content</div>;
```

With React Compiler, write the callback ref as a plain function (`const measuredRef = (node: HTMLDivElement | null) => { ... }`); the compiler keeps its identity stable. Without the compiler, keep `useCallback`: a new function on every render makes React detach and re-attach the ref, running its cleanup each time.

In React 19+, callback refs support cleanup return values. For earlier versions, store cleanup logic in a ref inside a custom hook.

### Subscribing to external stores

Use `useSyncExternalStore` instead of manual subscription effects:

```tsx
const isOnline = useSyncExternalStore(
  subscribe,
  () => navigator.onLine,
  () => true
);
```

### App initialization

Run once at module level, not in an effect:

```tsx
if (typeof window !== 'undefined') {
  checkAuthToken();
  loadDataFromLocalStorage();
}
```

### Chains of computations

Never chain effects that trigger each other. Derive values inline and batch state updates in the event handler:

```tsx
// BAD: 3 effects chaining state -> 3 extra renders
useEffect(() => { if (card?.gold) setGoldCardCount(c => c + 1); }, [card]);
useEffect(() => { if (goldCardCount > 3) { setRound(r => r + 1); setGoldCardCount(0); } }, [goldCardCount]);
useEffect(() => { if (round > 5) setIsGameOver(true); }, [round]);

// GOOD: derive + handle in event
const isGameOver = round > 5;

function handlePlaceCard(nextCard) {
  setCard(nextCard);
  if (nextCard.gold) {
    if (goldCardCount < 3) setGoldCardCount(goldCardCount + 1);
    else { setGoldCardCount(0); setRound(round + 1); }
  }
}
```

## Decision Checklist

Before writing any effect, answer these questions:

1. **Can I compute it during render?** -> Derive it inline (`useMemo` only for expensive work, and only without React Compiler)
2. **Is it triggered by a user action?** -> Event handler
3. **Am I fetching data?** -> Data-fetching library (React Query, SWR, etc.) or `use()` + Suspense in React 19+
4. **Am I subscribing to an external store?** -> `useSyncExternalStore`
5. **Do I need to reset state when a prop changes?** -> `key` prop
6. **Is it true mount-time external system sync?** -> `useMountEffect`
7. **Am I using refs to control when/if an effect runs?** -> Remove the refs; move logic to an event handler, derive it, or use `useEffectEvent`
8. **Do I need a DOM side effect when a node mounts?** -> Callback ref (a plain function with React Compiler)

If none of the above apply and you still think you need `useEffect`, see [references/patterns.md](references/patterns.md) for detailed examples.

## Failure Modes

- **useMountEffect failures** are binary and loud: it ran once, or not at all
- **Direct useEffect failures** degrade gradually as flaky behavior, perf issues, or loops before a hard crash

Choose the bug that's easy to find.
