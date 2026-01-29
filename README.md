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

## Tree/Hierarchy Patterns

### 3. Flat List → Tree Structure (Map-Based)
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

### 4. Integration Testing (Component + Data Flow)
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
| Map-Based Tree | Flat list → tree structure | O(n) |
| Integration Testing | Testing component interactions + async | N/A |

## Additional Resources

- React Testing Library: Focus on testing user behavior, not implementation
- MSW (Mock Service Worker): Mock APIs at network level
- Storybook: Visual development + interaction testing
- React DevTools Profiler: Identify unnecessary re-renders
