---
layout: default
title: "Chapter 14: Testing"
nav_order: 15
---

# Chapter 14: Testing React: Because Your Components Need Therapy Too

Testing React applications is like trying to test a house of cards in a wind tunnel. You're not testing your code so much as testing React's abstractions of your code's representation of what might eventually become DOM elements. If that sounds complicated, wait until you see the actual tests.

## Unit Testing Components: Mocking the Universe

To test a simple component, you first need to mock everything it touches:

```jsx
// The component (5 lines)
const UserGreeting = ({ userId }) => {
  const user = useUser(userId);
  return <div>Hello, {user?.name}!</div>;
};

// The test (50 lines)
import { render, screen, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from 'react-query';
import { Provider } from 'react-redux';
import { MemoryRouter } from 'react-router-dom';
import { ThemeProvider } from 'styled-components';
import { createStore } from './test-utils';
import { mockUser } from './mocks';

jest.mock('../hooks/useUser', () => ({
  useUser: jest.fn()
}));

describe('UserGreeting', () => {
  let store;
  let queryClient;
  
  beforeEach(() => {
    store = createStore();
    queryClient = new QueryClient({
      defaultOptions: {
        queries: { retry: false }
      }
    });
    jest.clearAllMocks();
  });
  
  it('displays user name', async () => {
    useUser.mockReturnValue({ name: 'John' });
    
    render(
      <QueryClientProvider client={queryClient}>
        <Provider store={store}>
          <MemoryRouter>
            <ThemeProvider theme={theme}>
              <UserGreeting userId={1} />
            </ThemeProvider>
          </MemoryRouter>
        </Provider>
      </QueryClientProvider>
    );
    
    await waitFor(() => {
      expect(screen.getByText('Hello, John!')).toBeInTheDocument();
    });
  });
});
```

Fifty lines to test five lines. This is progress.

## React Testing Library: Not Testing Implementation Details (Allegedly)

React Testing Library's philosophy: "Test your components like users would use them!"

Reality: Users don't use components. They click buttons and read text. But here we are:

```jsx
// "Testing like a user"
const { container } = render(<Button />);
const button = container.querySelector('button');
fireEvent.click(button);

// Actual user
// *clicks button with mouse*
// *doesn't know what a querySelector is*
```

The motto should be: "Test your components like a user who knows JavaScript, the DOM API, and has access to your source code would use them."

## The Act Warning Nightmare

```jsx
// Your test
it('updates on click', () => {
  const { getByRole } = render(<Counter />);
  const button = getByRole('button');
  fireEvent.click(button);
  expect(button).toHaveTextContent('1');
});

// Console
Warning: An update to Counter inside a test was not wrapped in act(...).
```

The solution? Wrap everything in `act()`:

```jsx
import { act } from '@testing-library/react';

it('updates on click', async () => {
  const { getByRole } = render(<Counter />);
  const button = getByRole('button');
  
  await act(async () => {
    fireEvent.click(button);
  });
  
  expect(button).toHaveTextContent('1');
});
```

But wait, `fireEvent` is already wrapped in `act`! But sometimes it's not enough! Welcome to React testing, where the rules are made up and the warnings don't matter.

## Snapshot Testing: The Lie Detector That Lies

Snapshot testing was supposed to make testing easy:

```jsx
it('renders correctly', () => {
  const tree = renderer.create(<Component />).toJSON();
  expect(tree).toMatchSnapshot();
});
```

What actually happens:
1. Test fails
2. "Did the component change?" 
3. "I don't know, maybe?"
4. `npm test -- -u` (update all snapshots)
5. All tests pass
6. Ship it

Snapshot tests are like that friend who agrees with everything you say. Not helpful, but makes you feel good.

## Testing Hooks: The Abstract Abstraction

Hooks can't be tested directly because they're not components. So we test them with a special utility:

```jsx
import { renderHook, act } from '@testing-library/react-hooks';

// The hook
const useCounter = (initial = 0) => {
  const [count, setCount] = useState(initial);
  const increment = () => setCount(c => c + 1);
  return { count, increment };
};

// The test
it('increments counter', () => {
  const { result } = renderHook(() => useCounter());
  
  expect(result.current.count).toBe(0);
  
  act(() => {
    result.current.increment();
  });
  
  expect(result.current.count).toBe(1);
});
```

We're testing a function that returns an object by rendering it in a fake component that doesn't exist. Make sense? No? Perfect, you're ready for React testing.

## Async Testing: Wait For It...

Testing async operations in React is an exercise in patience:

```jsx
it('loads user data', async () => {
  render(<UserProfile userId={1} />);
  
  // Wait for loading to appear
  expect(screen.getByText('Loading...')).toBeInTheDocument();
  
  // Wait for loading to disappear
  await waitForElementToBeRemoved(() => screen.queryByText('Loading...'));
  
  // Wait for data to appear
  await waitFor(() => {
    expect(screen.getByText('John Doe')).toBeInTheDocument();
  }, {
    timeout: 3000,
    onTimeout: () => console.log('Still waiting...'),
  });
  
  // Wait for potential re-renders
  await waitFor(() => {
    expect(screen.queryByText('Loading...')).not.toBeInTheDocument();
  });
  
  // Just to be safe
  await new Promise(resolve => setTimeout(resolve, 100));
});
```

Half your test is just waiting for React to finish doing React things.

## Mocking Everything

React tests require mocking everything:

```jsx
// Mock fetch
global.fetch = jest.fn();

// Mock timers
jest.useFakeTimers();

// Mock router
jest.mock('react-router-dom', () => ({
  ...jest.requireActual('react-router-dom'),
  useNavigate: () => jest.fn(),
  useLocation: () => ({ pathname: '/' }),
}));

// Mock local storage
const localStorageMock = {
  getItem: jest.fn(),
  setItem: jest.fn(),
  clear: jest.fn()
};
global.localStorage = localStorageMock;

// Mock window
delete window.location;
window.location = { reload: jest.fn() };

// Mock console to hide errors you don't care about
jest.spyOn(console, 'error').mockImplementation(() => {});
```

By the time you're done mocking, you're not testing React anymore. You're testing your mocks.

## E2E Testing: When You've Given Up on Unit Tests

Eventually, you give up and use Cypress or Playwright:

```javascript
// Cypress test
it('user can add a todo', () => {
  cy.visit('/');
  cy.get('[data-testid="todo-input"]').type('New todo');
  cy.get('[data-testid="add-button"]').click();
  cy.contains('New todo').should('be.visible');
});
```

This actually tests what users do! But now you need:
- A running server
- A test database
- Seed data
- Cleanup scripts
- CI/CD pipeline changes
- 10x longer test runs

## The Coverage Lie

```bash
--------------------------|---------|----------|---------|---------|
File                      | % Stmts | % Branch | % Funcs | % Lines |
--------------------------|---------|----------|---------|---------|
All files                 |   95.43 |    88.24 |   92.31 |   95.43 |
--------------------------|---------|----------|---------|---------|

✨ 95% code coverage!
```

What this actually means:
- Your code was executed
- Not that it works correctly
- Not that it handles edge cases
- Not that users can use it
- Just that JavaScript ran without throwing

## Testing State Management

Testing Redux is its own special hell:

```jsx
// Test the action creator
it('creates ADD_TODO action', () => {
  const action = addTodo('Test');
  expect(action).toEqual({
    type: 'ADD_TODO',
    payload: 'Test'
  });
});

// Test the reducer
it('handles ADD_TODO', () => {
  const newState = reducer(initialState, addTodo('Test'));
  expect(newState.todos).toHaveLength(1);
});

// Test the selector
it('selects todos', () => {
  const state = { todos: ['Test'] };
  expect(selectTodos(state)).toEqual(['Test']);
});

// Test the thunk
it('dispatches async action', async () => {
  const dispatch = jest.fn();
  await fetchTodos()(dispatch);
  expect(dispatch).toHaveBeenCalledWith(setTodos(expect.any(Array)));
});

// Test the connected component
it('connects to store', () => {
  // ... 50 more lines
});
```

You're testing Redux more than your actual application.

## The Debug Experience

When tests fail, the error messages are helpful:

```
Expected: "Hello, John!"
Received: <div>Hello, undefined!</div>

  at Object.<anonymous> (UserGreeting.test.js:45:28)
  at processTicksAndRejections (internal/process/task_queues.js:97:5)

● UserGreeting › displays user name

  Unable to find an element with the text: Hello, John!. 
  This could be because the text is broken up by multiple elements. 
  In this case, you can provide a function for your text matcher to make your matcher more flexible.

  <body>
    <div>
      <div>
        Hello, 
        undefined
        !
      </div>
    </div>
  </body>
```

So helpful. So clear. So actionable.

## The Testing Pyramid (More Like Testing Jenga)

The ideal:
```
        /\        E2E (few)
       /  \
      /    \      Integration (some)
     /      \
    /________\    Unit (many)
```

The reality:
```
    |\      /|    Snapshot tests (thousands)
    | \    / |
    |  \  /  |    Unit tests that test React (hundreds)
    |   \/   |
    |   /\   |    Integration tests that fail randomly (dozens)
    |  /  \  |
    | /    \ |    E2E tests that work locally (three)
    |/______\|    Manual testing (everything important)
```

## In Defense of Testing (There's Not Much)

Testing does have benefits:
1. **Refactoring confidence** (if tests aren't too coupled)
2. **Documentation** (if tests are readable)
3. **Regression prevention** (if tests are maintained)
4. **CI/CD requirement** (checkbox driven development)

But React makes testing harder than it needs to be.

## The Path Forward

You're going to write tests. They're going to be brittle. They're going to test implementation details despite your best efforts. They're going to break when you upgrade React. They're going to give you false confidence.

But you'll write them anyway because:
- Your team requires them
- Your CI/CD pipeline demands them
- That green checkmark feels good
- 95% coverage looks impressive

Just remember: the best test is a user actually using your application. Everything else is theater.

In the next chapter, we'll explore the React ecosystem - the 500+ libraries you'll need to build a simple application.

But first, take a moment to appreciate the irony: we're testing code that tests code that represents code that might become HTML. It's abstraction all the way down, and confusion all the way up.