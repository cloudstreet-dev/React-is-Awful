---
layout: default
title: "Chapter 10: useEffect"
nav_order: 11
---

# Chapter 10: useEffect: The Footgun You'll Shoot Yourself With

useEffect is React's answer to the question, "How do we do side effects in functional components?" The answer, apparently, is "Confusingly, with lots of bugs, and in a way that makes developers question their sanity."

If React hooks were a family, useEffect would be the troubled teenager who means well but keeps setting the house on fire.

## ComponentDidMount's Evil Twin

In the before times, we had lifecycle methods that made sense:

```jsx
class OldComponent extends React.Component {
  componentDidMount() {
    // Runs once when component mounts
    console.log('I mounted!');
  }
  
  componentDidUpdate(prevProps) {
    // Runs when props change
    if (prevProps.id !== this.props.id) {
      console.log('ID changed!');
    }
  }
  
  componentWillUnmount() {
    // Runs once when component unmounts
    console.log('Goodbye!');
  }
}
```

Clear, explicit, predictable. React looked at this and said, "What if we combined all of these into one confusing function?"

```jsx
const NewComponent = ({ id }) => {
  useEffect(() => {
    console.log('I... mounted? Updated? Both? WHO KNOWS!');
    
    return () => {
      console.log('Cleanup! Or unmounting! Or both! MYSTERY!');
    };
  }, [id]); // When id changes... or on mount... or...
};
```

## The Dependency Array of Doom

The second argument to useEffect is an array that determines when the effect runs. Sounds simple. It's not.

```jsx
// Runs after every render (usually bad)
useEffect(() => {
  console.log('Every. Single. Render.');
});

// Runs once on mount (lies, runs twice in StrictMode)
useEffect(() => {
  console.log('Once!');
}, []);

// Runs when dependencies change (or do they?)
useEffect(() => {
  console.log('Value changed!');
}, [value]);

// The footgun special
useEffect(() => {
  setCount(count + 1); // Infinite loop, here we come!
}, [count]);
```

The dependency array is like a promise that React makes: "I'll only run this when these values change." It's a promise React loves to break.

## Cleanup Functions: Forgetting Them Since 2019

useEffect can return a cleanup function. You'll forget to add it. Every. Single. Time.

```jsx
// Memory leak waiting to happen
useEffect(() => {
  const timer = setInterval(() => {
    console.log('Tick');
  }, 1000);
  
  // Oops, forgot to return cleanup!
  // This interval will run FOREVER
}, []);

// The fix you'll add after debugging for 2 hours
useEffect(() => {
  const timer = setInterval(() => {
    console.log('Tick');
  }, 1000);
  
  return () => clearInterval(timer); // Cleanup!
}, []);
```

But wait, there's more! The cleanup runs:
- Before the effect runs again
- When the component unmounts
- When dependencies change
- Whenever React feels like it

## The Infinite Loop Trap

Want to crash a browser? useEffect makes it easy!

```jsx
const InfiniteLoop = () => {
  const [count, setCount] = useState(0);
  
  // DON'T DO THIS
  useEffect(() => {
    setCount(count + 1); // Changes count
  }); // No dependency array = runs every render
  
  // OR THIS
  useEffect(() => {
    setCount(count + 1);
  }, [count]); // Runs when count changes, which it just did
  
  // OR THIS
  const [data, setData] = useState({});
  useEffect(() => {
    setData({ ...data, updated: true }); // New object every time
  }, [data]); // Object comparison by reference = always different
};
```

React: "Why is your CPU at 100%?"
You: "useEffect."
React: "Ah, carry on then."

## The Fetch Fiasco

Here's everyone's first useEffect data fetching attempt:

```jsx
const UserProfile = ({ userId }) => {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => setUser(data));
  }, [userId]);
  
  return <div>{user?.name}</div>;
};
```

Looks good, right? WRONG! This has:
- No error handling
- No loading state
- No cleanup for race conditions
- No handling for unmount during fetch

Here's what you actually need:

```jsx
const UserProfile = ({ userId }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    
    setLoading(true);
    setError(null);
    
    fetch(`/api/users/${userId}`)
      .then(res => {
        if (!res.ok) throw new Error('Failed to fetch');
        return res.json();
      })
      .then(data => {
        if (!cancelled) {
          setUser(data);
          setLoading(false);
        }
      })
      .catch(err => {
        if (!cancelled) {
          setError(err.message);
          setLoading(false);
        }
      });
    
    return () => {
      cancelled = true;
    };
  }, [userId]);
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  return <div>{user?.name}</div>;
};
```

That's 30 lines for a simple fetch. "Just use React Query," they'll say. Another library to solve problems React created.

## The Double Execution Surprise

React 18 introduced StrictMode that runs effects twice in development. For "safety."

```jsx
useEffect(() => {
  console.log('Mount!');
  
  return () => console.log('Unmount!');
}, []);

// In development with StrictMode:
// Mount!
// Unmount!
// Mount!
```

Your "run once" effect now runs twice. Your API calls happen twice. Your analytics fire twice. But only in development! Production is different!

React: "This helps you find bugs!"
Developers: "You ARE the bug!"

## The Stale Closure Problem

useEffect creates closures over values, leading to stale data bugs:

```jsx
const Timer = () => {
  const [seconds, setSeconds] = useState(0);
  
  useEffect(() => {
    const interval = setInterval(() => {
      console.log(seconds); // Always logs 0!
      setSeconds(seconds + 1); // Always sets to 1!
    }, 1000);
    
    return () => clearInterval(interval);
  }, []); // Empty deps = closure over initial value
  
  return <div>Seconds: {seconds}</div>;
};
```

The fix? Functional updates or adding dependencies (creating new intervals):

```jsx
// Option 1: Functional update
setSeconds(s => s + 1);

// Option 2: Add dependency (less efficient)
useEffect(() => {
  const interval = setInterval(() => {
    setSeconds(seconds + 1);
  }, 1000);
  
  return () => clearInterval(interval);
}, [seconds]); // New interval every second!
```

## Event Listeners: The Memory Leak Factory

```jsx
const MouseTracker = () => {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handleMove = (e) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    
    // Add listener
    window.addEventListener('mousemove', handleMove);
    
    // Forget cleanup? Memory leak!
    // Component unmounts, listener stays
    // Page gets slower and slower
    
    return () => {
      window.removeEventListener('mousemove', handleMove);
    };
  }, []); // At least we remembered the cleanup this time
};
```

## The Dependencies Linting Drama

ESLint's exhaustive-deps rule exists because humans can't track dependencies:

```jsx
const Component = ({ userId, teamId }) => {
  useEffect(() => {
    fetchUserData(userId);
    fetchTeamData(teamId); // Uses teamId
  }, [userId]); // Missing teamId! ESLint angry!
};
```

ESLint: "Add teamId to dependencies!"
You: "But I don't want it to run when teamId changes!"
ESLint: "Add it anyway!"
You: "Fine!" *adds eslint-disable-next-line*
ESLint: "You monster."

## The useEffect Chain Reaction

When effects trigger other effects:

```jsx
const CascadeOfDoom = () => {
  const [step1, setStep1] = useState(false);
  const [step2, setStep2] = useState(false);
  const [step3, setStep3] = useState(false);
  
  useEffect(() => {
    setStep1(true);
  }, []);
  
  useEffect(() => {
    if (step1) {
      setStep2(true);
    }
  }, [step1]);
  
  useEffect(() => {
    if (step2) {
      setStep3(true);
    }
  }, [step2]);
  
  // Waterfall of renders!
  // Mount -> Effect 1 -> Render -> Effect 2 -> Render -> Effect 3 -> Render
};
```

## The Async Function Trap

Want to use async/await in useEffect? Too bad!

```jsx
// This doesn't work
useEffect(async () => {
  const data = await fetchData(); // ERROR!
  setData(data);
}, []);

// You need this monstrosity
useEffect(() => {
  const fetchData = async () => {
    const data = await fetch('/api/data');
    setData(data);
  };
  
  fetchData();
}, []);

// Or this IIFE pattern
useEffect(() => {
  (async () => {
    const data = await fetch('/api/data');
    setData(data);
  })();
}, []);
```

Why? Because async functions return promises, and useEffect doesn't know what to do with promises. Logic!

## The Custom Hook Effect Multiplication

When custom hooks use useEffect:

```jsx
const useInterval = (callback, delay) => {
  useEffect(() => {
    const id = setInterval(callback, delay);
    return () => clearInterval(id);
  }, [callback, delay]);
};

const useTimeout = (callback, delay) => {
  useEffect(() => {
    const id = setTimeout(callback, delay);
    return () => clearTimeout(id);
  }, [callback, delay]);
};

const Component = () => {
  useInterval(() => console.log('Interval'), 1000);
  useTimeout(() => console.log('Timeout'), 2000);
  useEffect(() => console.log('Component effect'), []);
  
  // 3 different effects, all fighting for attention
};
```

## The Solutions That Aren't

React's solutions to useEffect problems:

1. **"Just use a data fetching library!"** - Adding complexity to avoid complexity
2. **"Effects should be idempotent!"** - They should be, but they're not
3. **"Use the rules of hooks ESLint plugin!"** - Linting our way out of bad design
4. **"Read the documentation!"** - It's 50 pages for one hook
5. **"Use React Query/SWR!"** - More libraries to fix React's problems

## In Defense of useEffect (It Hurts)

useEffect does enable some patterns:

1. **Declarative side effects** - You declare what should happen
2. **Cleanup is colocated** - The cleanup is right there with the effect
3. **Dependency tracking** - You can see what triggers the effect
4. **Composition** - Custom hooks can encapsulate effects

But these benefits come at the cost of complexity, bugs, and developer sanity.

## The Reality Check

useEffect is what happens when you try to shoehorn imperative operations into a declarative paradigm. It's React admitting that side effects exist while pretending they don't.

Before React:
```javascript
// Fetch data when needed
fetchUser(id).then(user => updateUI(user));

// Add event listener
element.addEventListener('click', handler);

// Set up timer
setInterval(tick, 1000);
```

Simple, imperative, obvious.

After React:
```jsx
useEffect(() => {
  let cancelled = false;
  
  (async () => {
    try {
      if (!cancelled) {
        const user = await fetchUser(id);
        if (!cancelled) {
          setUser(user);
        }
      }
    } catch (error) {
      if (!cancelled) {
        setError(error);
      }
    }
  })();
  
  return () => {
    cancelled = true;
  };
}, [id]);
```

Complex, confusing, error-prone.

## The Path Forward

You're going to use useEffect. You're going to create infinite loops. You're going to forget cleanup functions. You're going to have stale closures. You're going to wonder why your effect runs twice.

And eventually, you'll internalize all the footguns, work around them automatically, and forget that side effects used to be simple.

In the next chapter, we'll explore props drilling - React's method of making data sharing as painful as possible.

But first, take a moment to mourn the simplicity of `element.addEventListener`. No dependencies, no cleanup functions, no re-renders. Just an element, listening for events, like nature intended.

Now excuse me while I debug why my useEffect cleanup function is running before my effect. Or is it after? Or both? 

With useEffect, certainty is dead, and we killed it.