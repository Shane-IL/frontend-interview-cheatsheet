# Refs Beyond DOM Elements

## The Mental Model

Refs are for **"side effect infrastructure"** - things that exist outside React's render cycle but need to persist across renders.

**Key insight:** If changing it should cause a re-render, it's not a ref - it's state.

---

## When to Use Refs for Non-DOM Values

### 1. Storing mutable values that shouldn't trigger re-renders

```javascript
// Tracking if component is mounted
const isMountedRef = useRef(true);

useEffect(() => {
  return () => { isMountedRef.current = false; };
}, []);

// Avoid setState on unmounted component
fetchData().then(data => {
  if (isMountedRef.current) {
    setData(data);
  }
});
```

### 2. Storing timers/intervals/animation frames

```javascript
const timerRef = useRef(null);

const startTimer = () => {
  timerRef.current = setInterval(() => {
    // do something
  }, 1000);
};

const stopTimer = () => {
  clearInterval(timerRef.current);
};

// Always clean up
useEffect(() => {
  return () => clearInterval(timerRef.current);
}, []);
```

### 3. Storing instances that need cleanup

```javascript
const wsRef = useRef(null);

useEffect(() => {
  wsRef.current = new WebSocket(url);

  wsRef.current.onmessage = (event) => {
    // handle message
  };

  return () => wsRef.current?.close();
}, [url]);
```

### 4. Storing previous values

```javascript
function usePrevious(value) {
  const ref = useRef();

  useEffect(() => {
    ref.current = value;
  });

  return ref.current;
}

// Usage: compare current vs previous
function Counter({ count }) {
  const prevCount = usePrevious(count);

  return (
    <div>
      Now: {count}, Before: {prevCount}
    </div>
  );
}
```

### 5. Storing callbacks to avoid stale closures

```javascript
// Pattern: latest callback ref
const callbackRef = useRef(onSomething);

// Keep it updated
useEffect(() => {
  callbackRef.current = onSomething;
});

// Use in effect with empty deps - always calls latest callback
useEffect(() => {
  const handler = () => callbackRef.current();
  window.addEventListener('resize', handler);
  return () => window.removeEventListener('resize', handler);
}, []); // Empty deps - handler always calls latest callback
```

---

## The Golden Rule: Never Mutate Refs During Render

**Only mutate refs in effects and event handlers.**

```javascript
// ❌ BAD - mutating during render
function BadComponent() {
  const countRef = useRef(0);
  countRef.current++; // WRONG - inconsistent behavior
  return <div>{countRef.current}</div>;
}

// ✅ GOOD - mutating in effect
function GoodComponent() {
  const renderCountRef = useRef(0);

  useEffect(() => {
    renderCountRef.current++;
  });

  return <div>...</div>;
}

// ✅ GOOD - mutating in event handler
function EventExample() {
  const timerRef = useRef(null);

  const handleClick = () => {
    timerRef.current = setTimeout(() => {
      // do something
    }, 1000);
  };

  return <button onClick={handleClick}>Start</button>;
}
```

### Why this matters

In Strict Mode and with Concurrent React, renders can be:
- Called multiple times
- Discarded/retried
- Paused and resumed

Mutating during render means:
- Inconsistent state between renders
- Different behavior in dev vs production
- Race conditions with concurrent features

### Safe places to mutate refs

- `useEffect` / `useLayoutEffect`
- Event handlers (`onClick`, `onChange`, etc.)
- Callbacks (`setTimeout`, promises, etc.)
- Cleanup functions

---

## When NOT to Use Refs

| Don't Do This | Do This Instead |
|---------------|-----------------|
| Cache expensive calculations | `useMemo` |
| Store derived state | `useMemo` |
| Store values you need in dependency arrays | `useState` |
| Bypass normal React data flow | Lift state, use context |
| Use as a "global variable" workaround | Module-level const or state management |

---

## Interview Red Flags to Avoid

1. **Using refs to cache expensive calculations** - that's `useMemo`'s job
2. **Using refs as a "global variable" workaround** - smells like you're fighting React
3. **Mutating ref.current during render** - causes inconsistency, shows misunderstanding of React model
4. **Storing values that should trigger re-renders** - if UI depends on it, it's state

---

## Quick Reference

| Use Case | Ref? | Why |
|----------|------|-----|
| DOM element | Yes | Classic use case |
| Timer ID | Yes | Needs cleanup, no re-render needed |
| WebSocket instance | Yes | Persists across renders, needs cleanup |
| Previous value | Yes | Classic pattern |
| Stable callback | Yes | Avoids stale closures |
| Expensive computation | No | Use `useMemo` |
| Value that affects UI | No | Use `useState` |
