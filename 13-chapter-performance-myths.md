---
layout: default
title: "Chapter 11: Performance"
nav_order: 14
---

# Chapter 11: Performance: When React is Actually Fast (Sometimes)

"React is fast!" cry the evangelists. "React is slow!" cry the critics. The truth? React is exactly as fast as you let it be, which is usually not very fast at all because you're doing everything wrong.

## The Myths of React Performance

### Myth #1: "The Virtual DOM Makes React Fast"

We covered this lie in Chapter 5, but it bears repeating: the Virtual DOM doesn't make React fast. It makes React possible. There's a difference.

```jsx
// "Fast" React
const List = ({ items }) => (
  <ul>
    {items.map(item => <li key={item.id}>{item.name}</li>)}
  </ul>
);

// Actually fast vanilla JS
list.innerHTML = items.map(item => `<li>${item.name}</li>`).join('');
```

The vanilla version is faster. Always. The Virtual DOM is overhead, not optimization.

### Myth #2: "React Only Updates What Changes"

Technically true, practically false:

```jsx
const Parent = () => {
  const [count, setCount] = useState(0);
  
  console.log('Parent renders');
  
  return (
    <>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      <ExpensiveChild />  {/* Re-renders every time! */}
      <AnotherChild />    {/* This too! */}
      <ThirdChild />      {/* And this! */}
    </>
  );
};
```

Change parent state? Every child re-renders. React updates only what changes in the DOM, but it re-runs ALL component functions to figure out what changed.

### Myth #3: "Hooks Are Faster Than Classes"

```jsx
// Class component
class Counter extends React.Component {
  state = { count: 0 };
  
  increment = () => {
    this.setState({ count: this.state.count + 1 });
  };
  
  render() {
    return <button onClick={this.increment}>{this.state.count}</button>;
  }
}

// Hooks component
const Counter = () => {
  const [count, setCount] = useState(0);
  
  // New function created every render!
  const increment = () => setCount(count + 1);
  
  return <button onClick={increment}>{count}</button>;
};
```

Hooks create new functions every render. Classes reuse methods. "But hooks have smaller bundle size!" Sure, save 2KB on bundle size, lose 10ms on every render.

## React.memo: Memoizing Your Mistakes

React.memo is React admitting that unnecessary re-renders are a problem:

```jsx
// Without memo: Re-renders when parent re-renders
const ExpensiveComponent = ({ data }) => {
  console.log('Expensive calculation running...');
  const result = doExpensiveCalculation(data);
  return <div>{result}</div>;
};

// With memo: Only re-renders when props change
const ExpensiveComponent = React.memo(({ data }) => {
  console.log('Expensive calculation running...');
  const result = doExpensiveCalculation(data);
  return <div>{result}</div>;
});
```

But memo has its own problems:

```jsx
// This breaks memo
const Parent = () => {
  const [count, setCount] = useState(0);
  
  // New object every render!
  const config = { color: 'red', size: 'large' };
  
  return (
    <>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      <MemoizedComponent config={config} /> {/* Re-renders anyway! */}
    </>
  );
};
```

Objects and arrays break memo because JavaScript compares them by reference.

## useMemo and useCallback: Premature Optimization Central

The hooks everyone uses wrong:

```jsx
const OverOptimized = () => {
  // Wrong: Memoizing a primitive
  const name = useMemo(() => 'John', []);
  
  // Wrong: Memoizing cheap calculations
  const doubled = useMemo(() => value * 2, [value]);
  
  // Wrong: Dependencies change every render
  const filtered = useMemo(
    () => items.filter(item => item.active),
    [items] // If items is new every render, useless!
  );
  
  // Wrong: Memoizing inline styles
  const style = useMemo(() => ({ color: 'red' }), []);
  // Just move it outside the component!
};
```

The cost of memoization:
- Memory to store memoized values
- Comparison of dependencies
- Closure creation
- Mental overhead

Often more expensive than just recalculating!

## When Re-renders Actually Matter

Re-renders only matter when:

1. **You have expensive calculations:**
```jsx
const Component = ({ bigData }) => {
  // This is actually expensive
  const processed = useMemo(() => {
    return bigData.map(item => {
      // Complex calculations
      return heavyProcessing(item);
    });
  }, [bigData]);
};
```

2. **You have many components:**
```jsx
// 1000 items re-rendering is noticeable
const List = ({ items }) => (
  <>
    {items.map(item => (
      <ExpensiveItem key={item.id} item={item} />
    ))}
  </>
);
```

3. **You have frequent updates:**
```jsx
// Mouse move events = many updates
const MouseTracker = () => {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handleMove = (e) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    window.addEventListener('mousemove', handleMove);
    return () => window.removeEventListener('mousemove', handleMove);
  }, []);
  
  // This renders on every mouse move!
  return <ExpensiveVisualization position={position} />;
};
```

## The Real Performance Killers

### 1. Bundle Size

```javascript
// Your "simple" React app
import React from 'react';           // 6KB
import ReactDOM from 'react-dom';   // 120KB
import { Routes } from 'react-router'; // 50KB
import { Provider } from 'react-redux'; // 60KB
import { ThemeProvider } from 'styled-components'; // 40KB
// ... 50 more imports

// Total: 2MB before you write a line of code
```

### 2. Too Many Components

```jsx
// Why is this a component?
const Spacer = () => <div style={{ height: '10px' }} />;

// And this?
const Text = ({ children }) => <span>{children}</span>;

// And this??
const Wrapper = ({ children }) => <div>{children}</div>;
```

Each component is overhead. Function calls, Virtual DOM nodes, reconciliation.

### 3. Effect Waterfalls

```jsx
const WaterfallComponent = () => {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);
  const [comments, setComments] = useState([]);
  
  // First effect
  useEffect(() => {
    fetchUser().then(setUser);
  }, []);
  
  // Waits for user
  useEffect(() => {
    if (user) {
      fetchPosts(user.id).then(setPosts);
    }
  }, [user]);
  
  // Waits for posts
  useEffect(() => {
    if (posts.length > 0) {
      fetchComments(posts[0].id).then(setComments);
    }
  }, [posts]);
  
  // Serial loading = slow!
};
```

### 4. Inline Functions and Objects

```jsx
const Parent = () => {
  return (
    <Child
      // New function every render
      onClick={() => console.log('clicked')}
      // New object every render
      style={{ color: 'red' }}
      // New array every render
      items={['a', 'b', 'c']}
    />
  );
};
```

## When React is Actually Fast

React CAN be fast when:

1. **Components are properly memoized:**
```jsx
const OptimizedChild = React.memo(({ data }) => {
  return <div>{data}</div>;
}, (prevProps, nextProps) => {
  // Custom comparison
  return prevProps.data.id === nextProps.data.id;
});
```

2. **State is properly structured:**
```jsx
// Bad: One big state object
const [state, setState] = useState({
  user: {},
  posts: [],
  comments: [],
  // ... everything
});

// Good: Separate concerns
const [user, setUser] = useState({});
const [posts, setPosts] = useState([]);
const [comments, setComments] = useState([]);
```

3. **Lists use proper keys:**
```jsx
// Bad: Index as key
{items.map((item, index) => <Item key={index} />)}

// Good: Stable, unique ID
{items.map(item => <Item key={item.id} />)}
```

4. **Code splitting is used:**
```jsx
const HeavyComponent = lazy(() => import('./HeavyComponent'));

const App = () => (
  <Suspense fallback={<Loading />}>
    <HeavyComponent />
  </Suspense>
);
```

## The Optimization Checklist Nobody Follows

1. **Measure first** - Use React DevTools Profiler
2. **Bundle size** - It's probably too big
3. **Network requests** - They're probably serial
4. **Re-renders** - They're probably unnecessary
5. **Component depth** - It's probably too deep
6. **State location** - It's probably too high
7. **Effect dependencies** - They're probably wrong

## The Truth About React Performance

React isn't slow. React isn't fast. React is a complexity engine that turns simple UI updates into a chain of function calls, object creations, tree diffing, and DOM patches.

The performance you get depends entirely on how well you understand and work around React's limitations. It's like driving a car with square wheels - you can make it work, but you're fighting physics the whole way.

## In Defense of React Performance

React's performance is usually "good enough" for most applications. Users won't notice 16ms vs 32ms render times. They will notice 2MB bundles and serial data loading.

Focus on:
1. Bundle size (biggest impact)
2. Data fetching (second biggest)
3. Actual bottlenecks (not imagined ones)

## The Path Forward

You're going to optimize prematurely. You're going to memo components that render once. You're going to useCallback on functions that never change. You're going to spend hours optimizing a component that takes 2ms to render.

And then your app will still be slow because you're loading 2MB of JavaScript for a todo list.

In the next chapter, we'll explore testing React applications - because if you're going to build something slow and complex, you might as well test that it's consistently slow and complex.

But first, remember: the fastest React component is the one that doesn't exist. Every component you don't create is a performance optimization.

Now excuse me while I memo this paragraph to prevent unnecessary re-reads.