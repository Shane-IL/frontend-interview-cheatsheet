# Frontend Interview Pattern Cheat Sheet

## State Management Patterns

### 1. Derived State vs Stored State
**Problem**: Storing both ID and full object when one derives from the other
**Solution**: Use useMemo for derived values

```javascript
// ❌ BAD - Duplicate state
const [selectedDoorId, setSelectedDoorId] = useState(null);
const [selectedDoor, setSelectedDoor] = useState(null);

// ✅ GOOD - Derive it
const [selectedDoorId, setSelectedDoorId] = useState(null);
const selectedDoor = useMemo(
  () => doors.find(d => d.id === selectedDoorId),
  [doors, selectedDoorId]
);
```

### 2. Module-Level Cache (Avoiding React State)
**Problem**: Global cache in React state causes entire tree re-renders
**Solution**: Store cache outside React, force updates only where needed

```javascript
// Outside any component
const cache = new Map();

function useCache(keys) {
  const [, forceUpdate] = useReducer(x => x + 1, 0);
  
  useEffect(() => {
    const missing = keys.filter(k => !cache.has(k));
    if (!missing.length) return;
    
    fetchData(missing).then(data => {
      data.forEach(item => cache.set(item.id, item));
      forceUpdate(); // Only this component re-renders
    });
  }, [keys.join(',')]);
  
  return cache;
}
```

**When to use**:
- Shared data accessed by multiple components
- No prop drilling or Context needed
- Want to avoid cascade re-renders
- Pure React constraint (no external state management)

### 3. Pagination Caching (Map vs Array)
**Problem**: Caching paginated API data for navigation without refetching
**Two approaches**: Map indexed by page number vs flat array with range getter

#### Approach A: Map with Page Keys
```javascript
const [pages, setPages] = useState(new Map()); // Map<pageNumber, Item[]>
const [currentPage, setCurrentPage] = useState(1);
const [isLoading, setIsLoading] = useState(false);

useEffect(() => {
  if (pages.has(currentPage)) return; // Cache hit

  setIsLoading(true);
  fetchPage(currentPage).then(data => {
    setPages(prev => new Map(prev).set(currentPage, data));
    setIsLoading(false);
  });
}, [currentPage]);

// Render current page
const currentItems = pages.get(currentPage) || [];
```

#### Approach B: Flat Array with Range Getter
```javascript
const [items, setItems] = useState([]); // Sparse array, undefined = not loaded
const [totalCount, setTotalCount] = useState(0);

const PAGE_SIZE = 20;

const fetchRange = (start, end) => {
  const needsFetch = items.slice(start, end).some(item => item === undefined);
  if (!needsFetch) return;

  const startPage = Math.floor(start / PAGE_SIZE);
  fetchPage(startPage).then(data => {
    setItems(prev => {
      const next = [...prev];
      data.forEach((item, i) => next[start + i] = item);
      return next;
    });
  });
};

// Getter for visible range
const getRange = (start, end) => items.slice(start, end);
```

**When to use Map**:
- Discrete pagination UI (page 1, 2, 3 buttons)
- Clear cache hit/miss semantics
- Page boundaries matter to the UX

**When to use Array**:
- Infinite scroll
- Virtualized lists (react-window, react-virtual)
- Need to treat data as one continuous list

**Array gotchas**:
- Sparse arrays in JS are weird (holes vs undefined)
- May need to pre-size based on total count
- More complex logic to determine what ranges to fetch

## Tree/Hierarchy Patterns

### 4. Flat List → Tree Structure (Map-Based)
**Problem**: Convert flat list with parent references to nested tree
**Solution**: Two-pass with object lookup

```javascript
function buildTree(items) {
  // Pass 1: Create lookup with children arrays
  const map = {};
  items.forEach(item => {
    map[item.id] = { ...item, children: [] };
  });
  
  // Pass 2: Wire relationships by reference
  const roots = [];
  items.forEach(item => {
    if (item.parentId) {
      map[item.parentId].children.push(map[item.id]);
    } else {
      roots.push(map[item.id]);
    }
  });
  
  return roots;
}
```

**Example with employees**:
```javascript
const employees = [
  { id: 1, name: "Alice", managerId: null },
  { id: 2, name: "Bob", managerId: 1 },
  { id: 3, name: "Charlie", managerId: 1 },
  { id: 4, name: "David", managerId: 2 }
];

const tree = buildTree(employees);
// Returns: [{ Alice with children: [Bob with children: [David], Charlie] }]
```

**Rendering**:
```jsx
function TreeNode({ node }) {
  return (
    <div style={{ marginLeft: '20px' }}>
      <div>{node.name}</div>
      {node.children.length > 0 && (
        <div>
          {node.children.map(child => (
            <TreeNode key={child.id} node={child} />
          ))}
        </div>
      )}
    </div>
  );
}

function Tree({ items }) {
  const tree = buildTree(items);
  return (
    <div>
      {tree.map(root => (
        <TreeNode key={root.id} node={root} />
      ))}
    </div>
  );
}
```

**With expand/collapse**:
```jsx
function TreeNode({ node }) {
  const [expanded, setExpanded] = useState(true);
  const hasChildren = node.children.length > 0;
  
  return (
    <div style={{ marginLeft: '20px' }}>
      <div onClick={() => setExpanded(!expanded)} style={{ cursor: 'pointer' }}>
        {hasChildren && (expanded ? '▼ ' : '▶ ')}
        {node.name}
      </div>
      {expanded && hasChildren && (
        <div>
          {node.children.map(child => (
            <TreeNode key={child.id} node={child} />
          ))}
        </div>
      )}
    </div>
  );
}
```

**Complexity**: O(n) - two passes through the data

## Testing Patterns

### 5. Integration Testing (Component + Data Flow)
**Problem**: Testing multiple components working together with async operations
**Solution**: React Testing Library + MSW + real component tree

```javascript
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { setupServer } from 'msw/node';
import { rest } from 'msw';
import { DoorsPage } from './DoorsPage';

// Mock API responses
const server = setupServer(
  rest.get('/api/doors', (req, res, ctx) => {
    return res(ctx.json([
      { id: 'door-1', name: 'Door A', actionIds: ['action-1', 'action-2'] }
    ]));
  }),
  rest.get('/api/action/:id', (req, res, ctx) => {
    const { id } = req.params;
    return res(ctx.json({ id, name: 'Open', description: 'Opens door' }));
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test('selecting door fetches and displays actions', async () => {
  render(<DoorsPage />);
  
  // Wait for doors to load
  await waitFor(() => screen.getByText('Door A'));
  
  // Click door
  await userEvent.click(screen.getByText('Door A'));
  
  // Verify actions loaded and displayed
  await waitFor(() => screen.getByText('Open'));
  expect(screen.getByText('Opens door')).toBeInTheDocument();
});
```

**What you're testing**:
- Interactions between components
- Data fetching and state updates
- UI updates based on async operations
- Loading and error states

**Key differences from unit tests**:
- Render full component tree (not isolated)
- Use real hooks and data fetchers (not mocked)
- Mock only external APIs (with MSW)
- Test user flows, not implementation details

**Alternative: Storybook Integration Tests**:
```javascript
export const DoorSelection = {
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    
    await waitFor(() => canvas.getByText('Door A'));
    await userEvent.click(canvas.getByText('Door A'));
    await expect(canvas.getByText('Open')).toBeInTheDocument();
  },
  parameters: {
    msw: {
      handlers: [
        rest.get('/api/doors', (req, res, ctx) => {
          return res(ctx.json([
            { id: 'door-1', name: 'Door A', actionIds: ['action-1'] }
          ]));
        })
      ]
    }
  }
};
```

**Test scenarios to cover**:
- User interactions trigger correct API calls
- Loading states during async operations
- Error states (API fails, network issues)
- Cache behavior (hitting API vs reading from cache)
- Switching between items updates UI correctly
- Aggregated data displays correctly

## Performance Patterns

### 6. Memoization (useMemo, useCallback, React.memo)
**Problem**: Preventing unnecessary recalculations and re-renders

#### useMemo - Memoize computed values
```javascript
// ✅ GOOD - Expensive computation
const sortedItems = useMemo(
  () => items.slice().sort((a, b) => a.name.localeCompare(b.name)),
  [items]
);

// ❌ BAD - Cheap computation, memo overhead not worth it
const fullName = useMemo(
  () => `${firstName} ${lastName}`,
  [firstName, lastName]
);
// Just do: const fullName = `${firstName} ${lastName}`;
```

#### useCallback - Memoize functions (for referential equality)
```javascript
// ✅ GOOD - Passed to memoized child or in dependency array
const handleClick = useCallback((id) => {
  setSelectedId(id);
}, []);

// ❌ BAD - No memoized consumers, wasted overhead
const handleClick = useCallback(() => {
  console.log('clicked');
}, []);
```

#### React.memo - Memoize components
```javascript
// Only re-renders if props change (shallow compare)
const ExpensiveList = React.memo(function ExpensiveList({ items, onSelect }) {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id} onClick={() => onSelect(item.id)}>
          {item.name}
        </li>
      ))}
    </ul>
  );
});

// Parent must stabilize props for memo to help
function Parent() {
  const [items] = useState(initialItems);
  const handleSelect = useCallback((id) => { /* ... */ }, []);

  return <ExpensiveList items={items} onSelect={handleSelect} />;
}
```

**Memoization gotchas**:

```javascript
// ❌ GOTCHA: Object/array literals break memoization
<MemoizedChild style={{ color: 'red' }} />  // New object every render
<MemoizedChild items={items.filter(x => x.active)} />  // New array every render

// ✅ FIX: Memoize or lift to stable reference
const style = useMemo(() => ({ color: 'red' }), []);
const activeItems = useMemo(() => items.filter(x => x.active), [items]);

// ❌ GOTCHA: Missing dependencies
const handleSubmit = useCallback(() => {
  submitForm(formData);  // formData not in deps - stale closure!
}, []);

// ✅ FIX: Include all dependencies
const handleSubmit = useCallback(() => {
  submitForm(formData);
}, [formData]);

// ❌ GOTCHA: Memoizing everything "just in case"
// Adds complexity, memory overhead, and often no benefit
// Only memoize when you have evidence of perf issues
```

**When to memoize**:
- Expensive calculations (sorting, filtering large lists)
- Referential equality matters (deps arrays, memo'd children)
- Profiler shows component re-rendering unnecessarily

**When NOT to memoize**:
- Cheap operations (string concat, simple math)
- Component always re-renders anyway (props always change)
- No measured performance problem

### 7. Debouncing and Throttling
**Problem**: Rate-limiting expensive operations (API calls, heavy computations)

**Debounce**: Wait until user stops, then fire once
**Throttle**: Fire at most once per interval

#### Debounced Search Input
```javascript
function SearchInput({ onSearch }) {
  const [value, setValue] = useState('');

  // Stable debounced function
  const debouncedSearch = useMemo(
    () => debounce((query) => onSearch(query), 300),
    [onSearch]
  );

  // Cleanup on unmount
  useEffect(() => {
    return () => debouncedSearch.cancel();
  }, [debouncedSearch]);

  const handleChange = (e) => {
    const newValue = e.target.value;
    setValue(newValue);        // Update input immediately
    debouncedSearch(newValue); // Debounce the search
  };

  return <input value={value} onChange={handleChange} />;
}
```

#### useDebouncedValue Hook
```javascript
function useDebouncedValue(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Usage
function Search() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebouncedValue(query, 300);

  useEffect(() => {
    if (debouncedQuery) {
      fetchResults(debouncedQuery);
    }
  }, [debouncedQuery]);

  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

#### Throttled Scroll Handler
```javascript
function useThrottledCallback(callback, delay) {
  const lastRun = useRef(Date.now());

  return useCallback((...args) => {
    if (Date.now() - lastRun.current >= delay) {
      callback(...args);
      lastRun.current = Date.now();
    }
  }, [callback, delay]);
}

// Usage
function InfiniteList() {
  const handleScroll = useThrottledCallback((e) => {
    const { scrollTop, scrollHeight, clientHeight } = e.target;
    if (scrollTop + clientHeight >= scrollHeight - 100) {
      loadMore();
    }
  }, 200);

  return <div onScroll={handleScroll}>...</div>;
}
```

**Debounce/Throttle gotchas**:

```javascript
// ❌ GOTCHA: Creating new debounced function every render
const handleSearch = debounce((q) => search(q), 300); // New function each render!

// ✅ FIX: Memoize the debounced function
const handleSearch = useMemo(
  () => debounce((q) => search(q), 300),
  []
);

// ❌ GOTCHA: Stale closure in debounced callback
const [count, setCount] = useState(0);
const debouncedLog = useMemo(
  () => debounce(() => console.log(count), 300), // count is stale!
  []
);

// ✅ FIX: Use ref for latest value
const countRef = useRef(count);
countRef.current = count;
const debouncedLog = useMemo(
  () => debounce(() => console.log(countRef.current), 300),
  []
);

// ❌ GOTCHA: Not canceling on unmount (memory leak, state update on unmounted)
// ✅ FIX: Always cleanup
useEffect(() => () => debouncedFn.cancel(), [debouncedFn]);
```

**Debounce vs Throttle**:
| Use Case | Pattern |
|----------|---------|
| Search input | Debounce - wait for user to stop typing |
| Form validation | Debounce - validate after user pauses |
| Scroll position | Throttle - update at consistent intervals |
| Window resize | Throttle - recalculate layout periodically |
| Button spam prevention | Throttle - allow first click, ignore rapid repeats |

## Common Pitfalls

### Over-formatting responses
- **Problem**: Using bullet points, headers, bold for simple questions
- **Solution**: Use natural prose, save formatting for complex multi-part answers

### Mutating input data
- **Problem**: Modifying arrays/objects passed as arguments
- **Solution**: Clone data or use immutable patterns

```javascript
// ❌ BAD
function processItems(items) {
  items.forEach(item => item.processed = true);
  return items;
}

// ✅ GOOD
function processItems(items) {
  return items.map(item => ({ ...item, processed: true }));
}
```

### Premature optimization
- **Problem**: Adding complexity before it's needed
- **Solution**: Start simple, optimize when you have data showing it's needed

### Context switching overhead
- **Problem**: Struggling to recall patterns you know when under interview pressure
- **Solution**: Review cheat sheet before interviews, practice whiteboarding

## Interview Meta-Skills

### Pattern Recognition Under Pressure
- Keep a mental checklist of common patterns
- If stuck, think: "Have I seen something similar before?"
- Don't be afraid to think out loud and backtrack
- Remember: you know more than you think, retrieval is the hard part

### Clarifying Requirements
- Ask about scale (100 items vs 10,000 items)
- Ask about constraints (pure React? Can I use libraries?)
- Ask about performance requirements
- Ask about testing requirements upfront

### Showing Your Work
- Explain trade-offs ("This is O(n²) but clearer than...")
- Mention alternatives ("Could also use recursion, but iteration is safer")
- Acknowledge what you don't know
- Think about edge cases (empty lists, null values, circular references)

### Structuring Answers
For testing questions, cover:
1. **What** you're testing (integration points, user flows)
2. **Tool choices** (RTL, MSW, Vitest/Jest)
3. **Key scenarios** (happy path, errors, edge cases)
4. **Setup approach** (mocking strategy, test structure)

For architecture questions:
1. **Requirements** (restate to confirm understanding)
2. **Constraints** (scalability, maintainability, pure React?)
3. **Approach** (whiteboard/sketch components and data flow)
4. **Trade-offs** (why this over alternatives)

## Quick Reference

| Pattern | When to Use | Complexity |
|---------|-------------|------------|
| Derived State | Any time state can be computed from other state | O(1) per compute |
| Module-Level Cache | Shared data, no Context, avoid re-renders | O(1) lookups |
| Pagination (Map) | Discrete page navigation, clear cache semantics | O(1) lookups |
| Pagination (Array) | Infinite scroll, virtualized lists | O(1) lookups |
| Map-Based Tree | Flat list → tree structure | O(n) |
| useMemo/useCallback | Expensive calcs, referential equality for memo'd children | O(1) lookup |
| React.memo | Prevent re-renders when props unchanged | O(n) shallow compare |
| Debounce | Wait for pause (search, validation) | O(1) |
| Throttle | Rate-limit continuous events (scroll, resize) | O(1) |
| Integration Testing | Testing component interactions + async | N/A |

## Additional Resources

- React Testing Library: Focus on testing user behavior, not implementation
- MSW (Mock Service Worker): Mock APIs at network level
- Storybook: Visual development + interaction testing
- React DevTools Profiler: Identify unnecessary re-renders
