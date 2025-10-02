---
layout: default
title: "Chapter 9: Hooks"
nav_order: 10
---

# Chapter 9: Hooks: The Magic That Makes You Miss Classes

In 2018, React said, "You know those class components we've been telling you to use for the past five years? Yeah, forget about them. Functions are the future now. Here's a bunch of magic functions that start with 'use' to make it work."

Thus, hooks were born, and thousands of React developers had to relearn React. Again.

## useState: Because Functions Need Memory Too

Functions are supposed to be pure. Same input, same output. No memory, no side effects. React looked at this beautiful simplicity and said, "But what if functions could remember things?"

```jsx
const Counter = () => {
  const [count, setCount] = useState(0);
  
  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
};
```

This looks simple, but what's actually happening is dark magic. That function runs every render, yet somehow `count` persists. How? React maintains a secret array of state values tied to each component instance, accessed by the order in which hooks are called.

Yes, the ORDER. Which brings us to...

## The Rules of Hooks: Arbitrary but Mandatory

React has two rules for hooks, and breaking them causes your app to explode:

1. **Only call hooks at the top level** (not in loops, conditions, or nested functions)
2. **Only call hooks from React functions** (components or custom hooks)

This means you can't do:

```jsx
// DON'T DO THIS - React will cry
const Component = ({ shouldUseState }) => {
  if (shouldUseState) {
    const [state, setState] = useState(0); // BOOM!
  }
};

// OR THIS
const Component = () => {
  for (let i = 0; i < 3; i++) {
    const [state, setState] = useState(i); // EXPLOSION!
  }
};

// OR THIS
const Component = () => {
  if (Math.random() > 0.5) {
    return <div>Early return</div>;
  }
  const [state, setState] = useState(0); // CONDITIONAL EXECUTION = DEATH
};
```

Why? Because React tracks hooks by their call order. Skip a hook call, and React gets confused about which state belongs to which hook. It's like musical chairs, but if someone leaves, everyone after them sits in the wrong chair and the universe implodes.

## Custom Hooks: Abstracting Your Abstractions

Not content with just built-in hooks, React encourages you to create custom hooks. The rule? If it starts with "use", it's a hook!

```jsx
// A "custom hook" (aka a function that starts with "use")
const useCounter = (initialValue = 0) => {
  const [count, setCount] = useState(initialValue);
  
  const increment = () => setCount(count + 1);
  const decrement = () => setCount(count - 1);
  const reset = () => setCount(initialValue);
  
  return { count, increment, decrement, reset };
};

// Usage
const Counter = () => {
  const { count, increment, decrement, reset } = useCounter(0);
  // ...
};
```

Congratulations, you've created a function that calls other functions. Revolutionary!

The best part? You can't call your custom hook conditionally either:

```jsx
// Still broken
const Component = ({ shouldCount }) => {
  if (shouldCount) {
    const { count } = useCounter(); // NOPE!
  }
};
```

## Why Everything Starts with "use"

React's naming convention is that all hooks must start with "use". Why? So the linter knows it's a hook and can yell at you when you break the rules.

```jsx
// This is a hook (linter will check the rules)
const useMyThing = () => {
  useState(); // OK
};

// This is not a hook (linter ignores it)
const getMyThing = () => {
  useState(); // Linter won't catch this error!
};
```

It's like a secret handshake that tells React, "This function is special and fragile, handle with care."

## The useState Closure Trap

Here's a fun bug that will ruin your afternoon:

```jsx
const Timer = () => {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const interval = setInterval(() => {
      setCount(count + 1); // This will always set count to 1!
    }, 1000);
    
    return () => clearInterval(interval);
  }, []); // Empty deps array
  
  return <div>Count: {count}</div>;
};
```

The interval captures `count` when it's 0, and it stays 0 forever in that closure. The fix?

```jsx
setCount(prevCount => prevCount + 1); // Functional update
```

Or update the dependencies array and create a new interval every time count changes (inefficient but works):

```jsx
useEffect(() => {
  const interval = setInterval(() => {
    setCount(count + 1);
  }, 1000);
  
  return () => clearInterval(interval);
}, [count]); // Now it works but creates new interval each second
```

Closures: turning simple counters into computer science problems since 2018.

## useReducer: Redux at Home

When useState isn't complicated enough, there's useReducer:

```jsx
const initialState = { count: 0 };

function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };
    case 'decrement':
      return { count: state.count - 1 };
    default:
      throw new Error();
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  return (
    <>
      Count: {state.count}
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
    </>
  );
}
```

That's 20 lines to make a counter. But hey, it's "more predictable" than `setCount(count + 1)`.

## useMemo and useCallback: Premature Optimization Central

React gives you hooks to optimize performance before you even have a performance problem:

```jsx
const ExpensiveComponent = ({ data, id }) => {
  // "Optimize" a calculation
  const processedData = useMemo(() => {
    return data.map(item => item * 2); // "Expensive" calculation
  }, [data]);
  
  // "Optimize" a function
  const handleClick = useCallback(() => {
    console.log(id);
  }, [id]);
  
  return <div onClick={handleClick}>{processedData}</div>;
};
```

The irony? The optimization often costs more than the thing you're optimizing. It's like taking a helicopter to avoid a speed bump.

## useRef: The Escape Hatch

When you need to break free from React's paradigm and touch the real DOM:

```jsx
const TextInput = () => {
  const inputRef = useRef(null);
  
  const focusInput = () => {
    inputRef.current.focus(); // Direct DOM manipulation! Scandal!
  };
  
  return (
    <>
      <input ref={inputRef} />
      <button onClick={focusInput}>Focus</button>
    </>
  );
};
```

useRef is React admitting that sometimes, just sometimes, you need to do things the normal way.

## useContext: Global State with Extra Steps

```jsx
const ThemeContext = createContext();

const ThemeProvider = ({ children }) => {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
};

const ThemedButton = () => {
  const { theme } = useContext(ThemeContext);
  
  return <button className={theme}>Click me</button>;
};
```

It's like global variables, but with more boilerplate and the potential for re-render cascades!

## useLayoutEffect: useEffect's Impatient Sibling

When useEffect isn't confusing enough, there's useLayoutEffect:

```jsx
useLayoutEffect(() => {
  // Runs synchronously after DOM mutations
  // Before the browser paints
  // Blocks visual updates
  // Use sparingly or users will notice jank
}, []);

useEffect(() => {
  // Runs asynchronously after paint
  // Doesn't block visual updates
  // Use this 99% of the time
}, []);
```

Two hooks that do almost the same thing but not quite. Because one timing API wasn't enough.

## The Custom Hook Obsession

Once developers discovered custom hooks, everything became a hook:

```jsx
const useMousePosition = () => {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handleMove = (e) => setPosition({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', handleMove);
    return () => window.removeEventListener('mousemove', handleMove);
  }, []);
  
  return position;
};

const useLocalStorage = (key, initialValue) => {
  // 30 lines of code to save a value
};

const useDebounce = (value, delay) => {
  // 15 lines to delay a value
};

const usePrevious = (value) => {
  // 5 lines to remember the last value
};

const useWhyDidYouUpdate = (props) => {
  // 20 lines to debug your own code
};
```

We've created hooks to do things that were one-liners before React.

## The Hooks Dependency Array: A Footgun Collection

Every hook with a dependency array is a potential bug:

```jsx
useEffect(() => {
  // Runs once (maybe)
}, []);

useEffect(() => {
  // Runs every render (performance killer)
});

useEffect(() => {
  // Runs when 'value' changes (or does it?)
}, [value]);

useEffect(() => {
  // Oops, forgot a dependency
  // Eslint will yell at you
  // But the bug might be subtle
  doSomething(importantValue);
}, []); // Missing importantValue!
```

The exhaustive-deps ESLint rule exists because developers can't be trusted to track their own dependencies.

## The Migration Nightmare

Remember when hooks came out and you had to migrate your class components?

```jsx
// Before: Class component (2013-2018)
class Counter extends React.Component {
  state = { count: 0 };
  
  increment = () => {
    this.setState({ count: this.state.count + 1 });
  };
  
  render() {
    return (
      <button onClick={this.increment}>
        Count: {this.state.count}
      </button>
    );
  }
}

// After: Function component with hooks (2018-present)
const Counter = () => {
  const [count, setCount] = useState(0);
  
  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
};
```

Thousands of components rewritten. Millions of developer hours spent. To achieve the exact same functionality with different syntax.

## In Defense of Hooks (Through Gritted Teeth)

Hooks do have some benefits:

1. **Smaller bundle size** (no class component overhead)
2. **Easier to share logic** (custom hooks vs HOCs/render props)
3. **Easier to test** (they're just functions... sort of)
4. **Better minification** (function names compress better)
5. **No more `this` binding** (one less footgun)

But we traded these problems for new ones: dependency arrays, closure stale values, rules of hooks, and the great relearning of 2018.

## The Reality Check

Hooks are React admitting that class components were a mistake, but instead of going back to simple functions, they invented a complex system of magic functions that must be called in a specific order and can't be used conditionally.

It's like burning down your house because you don't like the kitchen, then building a new house where the kitchen is in the basement and you need a special key to open the fridge.

## The Path Forward

You're going to use hooks. You're going to forget dependencies. You're going to create infinite loops. You're going to wonder why your effect runs twice. You're going to debug stale closures.

And eventually, you'll memorize the rules, internalize the patterns, and forget there was ever a simpler way.

In the next chapter, we'll dive deep into useEffect - the hook that causes more bugs than any other, yet somehow became the cornerstone of React development.

But first, take a moment to appreciate the simplicity we lost when functions became stateful. Remember when functions were just functions? When they took inputs and returned outputs? When you could reason about them without considering call order and dependency arrays?

Those were the days. Now excuse me while I debug why my custom hook is causing an infinite re-render loop. It's probably a missing dependency. Or an extra one. Or React is running in strict mode. Or the moon is in the wrong phase.

With hooks, who knows?