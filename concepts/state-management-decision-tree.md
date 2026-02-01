# State Management: When to Use What

## The Decision Tree

```
Do I need this data to trigger re-renders?
│
├─ NO → Module-level variable or ref
│
└─ YES → Is it only relevant to this component + children?
         │
         ├─ YES → useState / useReducer
         │
         └─ NO → Would props drilling be 3+ levels?
                  │
                  ├─ NO → Lift state up, pass as props
                  │
                  └─ YES → Is the data relatively stable?
                           │
                           ├─ YES → Context API
                           │
                           └─ NO → Consider state management library
```

---

## Component State (useState / useReducer)

**When to use:**
- Data only relevant to that component and its children
- Form inputs, UI toggles, local temporary state
- When props drilling is ≤2 levels deep
- **Default choice unless you have a reason not to**

```javascript
function SearchInput() {
  const [query, setQuery] = useState('');
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      {isOpen && <Dropdown query={query} />}
    </div>
  );
}
```

**Interview tip:** Start here and lift state up as needed. Shows you understand React fundamentals.

---

## Module-Level Variables

**When to use:**
- Constants, configuration that never changes
- Singletons/shared instances
- Derived values that don't need to trigger re-renders

```javascript
// Fine - constants
const API_BASE = 'https://api.example.com';
const CACHE_TTL = 5 * 60 * 1000;

// Fine - singleton instance
const analytics = new AnalyticsClient();

// CAREFUL - mutable state here is usually a bug
let cache = new Map(); // SSR issues, no re-renders
```

**Interview red flag:** Don't use module-level mutable state for reactive data. Interviewers will likely flag this as a bug because:
- No re-renders when it changes
- SSR hydration issues
- Shared between requests in server context

---

## Context API

**When to use:**
- Data needed by many components at different nesting levels
- When props drilling would be 3+ levels deep
- Relatively stable data (not updating frequently)
- When you want to avoid external dependencies

**Classic examples:** Theme, auth/user data, i18n, feature flags

```javascript
const UserContext = createContext(null);

function App() {
  const [user, setUser] = useState(null);

  return (
    <UserContext.Provider value={{ user, setUser }}>
      <Header />    {/* needs user */}
      <Main />
      <Sidebar />   {/* needs user */}
    </UserContext.Provider>
  );
}

function Header() {
  const { user } = useContext(UserContext);
  return <div>Welcome, {user?.name}</div>;
}
```

**Interview tip:** When you reach for Context, explain the props drilling problem you're solving.

**Context gotchas:**
- All consumers re-render when context value changes
- Split contexts by update frequency if needed
- Don't put everything in one giant context

---

## State Management Library

**When to use:**
- Complex async workflows with side effects
- Significant client-side caching needs
- Multiple components updating shared state frequently
- When you need devtools/time-travel debugging

```javascript
// Zustand example - simple and minimal
const useStore = create((set) => ({
  items: [],
  addItem: (item) => set((state) => ({
    items: [...state.items, item]
  })),
}));

// React Query for server state
const { data, isLoading } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
});
```

**Interview tip:** Usually overkill unless the problem is clearly complex enough. But mention a library by name if warranted: "This would be cleaner with Zustand for the atom pattern" or "If this were real, I'd use React Query for the server state."

---

## Quick Reference

| Approach | Use When | Re-renders? | Complexity |
|----------|----------|-------------|------------|
| useState | Local UI state, forms | Yes | Low |
| Module variable | Constants, singletons | No | Low |
| Context | Theme, auth, i18n | Yes (all consumers) | Medium |
| State library | Complex async, shared mutable | Yes (selective) | Higher |

---

## Interview Strategy

1. **Start with component state** and lift it up as needed
2. **Explain your reasoning** - the thought process matters more than the choice
3. **Acknowledge trade-offs** - "Context works here, but if this updated frequently I'd consider..."
4. **Mention libraries by name** when relevant, even if you don't implement them
