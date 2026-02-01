# Unit, Integration, and E2E Tests

## The Testing Pyramid

```
        /\
       /  \      E2E (few)
      /----\     - Slow, expensive, brittle
     /      \    - Test full user journeys
    /--------\
   /          \  Integration (some)
  /            \ - Test components working together
 /--------------\
/                \ Unit (many)
 ----------------  - Fast, cheap, isolated
```

The idea: have lots of fast unit tests at the bottom, fewer integration tests in the middle, and a small number of E2E tests at the top.

---

## Unit Tests

**What they are**: Tests for a single function, hook, or component in complete isolation.

**Key characteristic**: If it breaks, you know exactly what's wrong.

### What "isolation" means

You mock/stub everything external:
- API calls
- Other modules
- Browser APIs
- Even child components sometimes

### Example: Testing a utility function

```javascript
// utils/formatPrice.js
export function formatPrice(cents) {
  if (cents < 0) return '$0.00';
  return `$${(cents / 100).toFixed(2)}`;
}

// utils/formatPrice.test.js
import { formatPrice } from './formatPrice';

test('formats cents to dollars', () => {
  expect(formatPrice(1999)).toBe('$19.99');
});

test('handles zero', () => {
  expect(formatPrice(0)).toBe('$0.00');
});

test('handles negative (edge case)', () => {
  expect(formatPrice(-500)).toBe('$0.00');
});
```

### Example: Testing a custom hook

```javascript
// hooks/useCounter.js
export function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  const increment = () => setCount(c => c + 1);
  const decrement = () => setCount(c => c - 1);
  return { count, increment, decrement };
}

// hooks/useCounter.test.js
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

test('starts at initial value', () => {
  const { result } = renderHook(() => useCounter(5));
  expect(result.current.count).toBe(5);
});

test('increments', () => {
  const { result } = renderHook(() => useCounter(0));
  act(() => result.current.increment());
  expect(result.current.count).toBe(1);
});
```

### When to use unit tests

- Pure functions (formatters, validators, calculators)
- Custom hooks with logic
- Complex algorithms
- Anything where you need to test many edge cases quickly

### Limitations

- A unit test passing doesn't mean things work together
- You can have 100% unit test coverage and still have a broken app

---

## Integration Tests

**What they are**: Tests for multiple units working together, but not the whole system.

**Key characteristic**: Tests realistic user interactions within a bounded part of your app.

### What "integration" means in frontend

Usually: rendering a component tree and testing that:
- Child components render correctly
- State flows between components
- API calls happen and UI updates
- User interactions trigger the right behavior

### The key difference from unit tests

You're NOT mocking your own code. You mock the boundaries:
- API calls (with MSW or similar)
- Browser APIs if needed
- Maybe third-party libraries

But your components, hooks, and utilities are real.

### Example: Testing a search feature

```javascript
// Components: SearchPage -> SearchInput + ResultsList
// This tests them working together

import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';
import { SearchPage } from './SearchPage';

// Mock the API at the network level
const server = setupServer(
  http.get('/api/search', ({ request }) => {
    const url = new URL(request.url);
    const query = url.searchParams.get('q');

    if (query === 'react') {
      return HttpResponse.json([
        { id: 1, title: 'React Basics' },
        { id: 2, title: 'React Hooks' }
      ]);
    }
    return HttpResponse.json([]);
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test('searching shows results', async () => {
  render(<SearchPage />);

  // Type in the search input
  await userEvent.type(screen.getByRole('textbox'), 'react');
  await userEvent.click(screen.getByRole('button', { name: /search/i }));

  // Results appear
  await waitFor(() => {
    expect(screen.getByText('React Basics')).toBeInTheDocument();
    expect(screen.getByText('React Hooks')).toBeInTheDocument();
  });
});

test('no results shows empty state', async () => {
  render(<SearchPage />);

  await userEvent.type(screen.getByRole('textbox'), 'xyznotfound');
  await userEvent.click(screen.getByRole('button', { name: /search/i }));

  await waitFor(() => {
    expect(screen.getByText(/no results/i)).toBeInTheDocument();
  });
});
```

### What makes this an integration test?

- `SearchPage`, `SearchInput`, and `ResultsList` are all real
- State management between them is real
- Only the HTTP layer is mocked
- We're testing a user flow, not implementation details

### When to use integration tests

- Features that involve multiple components
- Data fetching flows
- Form submissions
- Any user flow contained within a page or feature

### Limitations

- Slower than unit tests
- Don't catch issues with real APIs, browsers, or infrastructure

---

## E2E (End-to-End) Tests

**What they are**: Tests that run against your real, deployed application in a real browser.

**Key characteristic**: Nothing is mocked. Real browser, real API, real database.

### What you're testing

The entire system from the user's perspective:
- Frontend code
- Backend APIs
- Database
- Authentication
- Third-party services
- Browser behavior

### Example: Testing login flow with Playwright

```javascript
// tests/login.spec.js
import { test, expect } from '@playwright/test';

test('user can log in and see dashboard', async ({ page }) => {
  // Go to real app
  await page.goto('https://staging.myapp.com');

  // Fill real login form
  await page.fill('[name="email"]', 'test@example.com');
  await page.fill('[name="password"]', 'testpassword123');
  await page.click('button[type="submit"]');

  // Verify real redirect and content
  await expect(page).toHaveURL(/.*dashboard/);
  await expect(page.getByText('Welcome back')).toBeVisible();
});

test('invalid login shows error', async ({ page }) => {
  await page.goto('https://staging.myapp.com');

  await page.fill('[name="email"]', 'wrong@example.com');
  await page.fill('[name="password"]', 'wrongpassword');
  await page.click('button[type="submit"]');

  await expect(page.getByText(/invalid credentials/i)).toBeVisible();
});
```

### Example: Testing a purchase flow

```javascript
test('user can complete checkout', async ({ page }) => {
  // Login first
  await page.goto('https://staging.myapp.com/login');
  await page.fill('[name="email"]', 'test@example.com');
  await page.fill('[name="password"]', 'testpassword123');
  await page.click('button[type="submit"]');

  // Add item to cart
  await page.goto('https://staging.myapp.com/products/widget');
  await page.click('button:has-text("Add to Cart")');

  // Go to checkout
  await page.click('a:has-text("Cart")');
  await page.click('button:has-text("Checkout")');

  // Fill payment (using test card)
  await page.fill('[name="cardNumber"]', '4242424242424242');
  await page.fill('[name="expiry"]', '12/25');
  await page.fill('[name="cvc"]', '123');
  await page.click('button:has-text("Pay")');

  // Verify success
  await expect(page.getByText(/order confirmed/i)).toBeVisible();
});
```

### When to use E2E tests

- Critical user paths (login, checkout, signup)
- Flows that touch multiple systems
- Smoke tests to verify deployment worked
- Anything where you need confidence the whole system works

### Limitations

- Slow (seconds to minutes per test)
- Flaky (network issues, timing, external services)
- Expensive to maintain
- Hard to test edge cases

---

## Quick Comparison

| Aspect | Unit | Integration | E2E |
|--------|------|-------------|-----|
| Speed | Milliseconds | Seconds | Seconds to minutes |
| What's real | Just the unit | Your code | Everything |
| What's mocked | Everything else | External APIs | Nothing |
| Confidence | Low (isolated) | Medium | High |
| Debugging | Easy | Medium | Hard |
| Flakiness | Rare | Sometimes | Common |
| Best for | Logic, utils, hooks | Features, flows | Critical paths |

---

## The Mental Model

Think of it like testing a car:

- **Unit test**: Test that the spark plug sparks, the fuel injector injects, the piston moves
- **Integration test**: Test that the engine runs when you turn the key
- **E2E test**: Drive the car from point A to point B

Each level gives you different confidence. You need all three, but in different proportions.

---

## Common Interview Questions

**"How do you decide what type of test to write?"**

Start by asking: what am I trying to verify?
- Specific logic with edge cases → unit test
- Components working together → integration test
- Critical user journey works end-to-end → E2E test

**"What's the difference between mocking in unit vs integration tests?"**

- Unit: mock everything external to the unit
- Integration: only mock system boundaries (APIs, browser APIs)
- E2E: mock nothing (or very rarely, external services you don't control)

**"Why not just write E2E tests for everything?"**

- Too slow for rapid feedback during development
- Too flaky for CI (random failures waste time)
- Too hard to cover edge cases
- Too expensive to maintain

**"Why not just write unit tests for everything?"**

- Doesn't catch integration bugs
- Components can work in isolation but break together
- False confidence from high coverage numbers
