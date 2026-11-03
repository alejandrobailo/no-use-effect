# no-use-effect

A Skill that enforces a strict no-direct-`useEffect` rule in React and React Native codebases.

Based on [Factory's approach](https://x.com/alvinsng/status/1900587498498195) and React's official [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) guide.

## Install

```bash
npx skills add alejandrobailo/no-use-effect
```

## The 6 Rules

1. **Derive state, don't sync it** — Compute inline (`useMemo` only for expensive work without React Compiler)
2. **Use data-fetching libraries** — React Query, SWR, etc., or `use()` + Suspense in React 19+, instead of fetch-in-effect
3. **Event handlers, not effects** — User actions belong in handlers
4. **`useMountEffect` for mount-time sync** — Named wrapper for `useEffect(..., [])`
5. **Reset with `key`** — Use React's remount semantics instead of reset effects
6. **Never patch a broken effect with a ref** — Remove the effect instead of guarding it with `hasRun`/`isMounted` refs

## What it does

When active, this skill prevents Claude from writing `useEffect` directly in components. Instead, it guides toward the correct primitive for each case: derived state, event handlers, `useSyncExternalStore`, data-fetching libraries, key-based resets, or `useMountEffect`.

## React Compiler

The skill detects React Compiler (`reactCompiler: true` in `next.config.*` or `babel-plugin-react-compiler` in `package.json`). When it's on, the skill never suggests `useMemo` or `useCallback`, and project rules about memoization take precedence.
