# Chapter 12: Patterns and Anti-Patterns: The Good, The Bad, The React

React patterns are like fashion trends. What's hot today is tomorrow's code smell. What was an anti-pattern last year is this year's best practice. And just like fashion, everyone pretends they always knew bell-bottoms would come back.

## Container/Presentational: Separation That Isn't

Remember when this was the gold standard?

```jsx
// Container Component (Smart)
class UserListContainer extends React.Component {
  state = { users: [] };
  
  componentDidMount() {
    fetch('/api/users')
      .then(res => res.json())
      .then(users => this.setState({ users }));
  }
  
  render() {
    return <UserList users={this.state.users} />;
  }
}

// Presentational Component (Dumb)
const UserList = ({ users }) => (
  <ul>
    {users.map(user => (
      <li key={user.id}>{user.name}</li>
    ))}
  </ul>
);
```

"Separation of concerns!" we said. "Smart components handle logic, dumb components handle presentation!"

Then hooks came along:

```jsx
const UserList = () => {
  const [users, setUsers] = useState([]);
  
  useEffect(() => {
    fetch('/api/users')
      .then(res => res.json())
      .then(setUsers);
  }, []);
  
  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
};
```

Suddenly, the pattern we swore by for years was an anti-pattern. Separation of concerns became "why are you creating two components for no reason?"

## Higher-Order Components: Inception for Code

HOCs were React's way of sharing logic before hooks. It's a function that takes a component and returns a component. If that sounds confusing, wait until you see it in action:

```jsx
// The HOC
const withAuth = (WrappedComponent) => {
  return class extends React.Component {
    state = { user: null };
    
    componentDidMount() {
      // Auth logic
      this.setState({ user: getCurrentUser() });
    }
    
    render() {
      return <WrappedComponent {...this.props} user={this.state.user} />;
    }
  };
};

// Usage
const ProfilePage = ({ user }) => <div>Welcome, {user.name}!</div>;
const AuthProfilePage = withAuth(ProfilePage);

// Or with multiple HOCs (HOC hell)
export default withAuth(
  withRouter(
    withTheme(
      withLocale(
        withAnalytics(
          withErrorBoundary(
            ProfilePage
          )
        )
      )
    )
  )
);
```

That's six levels of wrapping. Your component is now a Russian doll of functions returning functions returning components.

## Render Props: Functions Returning Functions Returning JSX

When HOCs weren't confusing enough, we invented render props:

```jsx
class Mouse extends React.Component {
  state = { x: 0, y: 0 };
  
  handleMouseMove = (e) => {
    this.setState({ x: e.clientX, y: e.clientY });
  };
  
  render() {
    return (
      <div onMouseMove={this.handleMouseMove}>
        {this.props.render(this.state)}
      </div>
    );
  }
}

// Usage
<Mouse render={({ x, y }) => (
  <div>The mouse is at ({x}, {y})</div>
)} />
```

We're passing functions as props that return JSX. It's like callback hell and component composition had a baby.

## Compound Components: When Simple Gets Complicated

Compound components seem elegant:

```jsx
<Tabs>
  <TabList>
    <Tab>Tab 1</Tab>
    <Tab>Tab 2</Tab>
  </TabList>
  <TabPanels>
    <TabPanel>Panel 1</TabPanel>
    <TabPanel>Panel 2</TabPanel>
  </TabPanels>
</Tabs>
```

But implementing them requires:
- React.Children.map
- Context API
- cloneElement
- Implicit parent-child contracts
- Prayer that someone doesn't rearrange the children

```jsx
const Tabs = ({ children }) => {
  const [activeIndex, setActiveIndex] = useState(0);
  
  return (
    <TabContext.Provider value={{ activeIndex, setActiveIndex }}>
      {React.Children.map(children, (child, index) =>
        React.cloneElement(child, { index })
      )}
    </TabContext.Provider>
  );
};
```

Congratulations, you've made tabs complicated.

## The useReducer Pattern: Redux Envy

"useState is too simple!" said nobody ever. But React gave us useReducer anyway:

```jsx
const formReducer = (state, action) => {
  switch (action.type) {
    case 'UPDATE_FIELD':
      return { ...state, [action.field]: action.value };
    case 'RESET':
      return initialState;
    case 'SUBMIT':
      return { ...state, isSubmitting: true };
    default:
      return state;
  }
};

const Form = () => {
  const [state, dispatch] = useReducer(formReducer, initialState);
  
  return (
    <form onSubmit={() => dispatch({ type: 'SUBMIT' })}>
      <input
        value={state.name}
        onChange={(e) => dispatch({
          type: 'UPDATE_FIELD',
          field: 'name',
          value: e.target.value
        })}
      />
    </form>
  );
};
```

We've turned `setState({ name: value })` into a 30-line state machine. Because Redux worked so well, right?

## The Custom Hook Everything Pattern

Once developers discovered custom hooks, everything became a hook:

```jsx
// Before hooks: Simple function
function formatDate(date) {
  return date.toLocaleDateString();
}

// After hooks: Everything must be a hook!
function useFormattedDate(date) {
  const [formatted, setFormatted] = useState('');
  
  useEffect(() => {
    setFormatted(date.toLocaleDateString());
  }, [date]);
  
  return formatted;
}
```

Congratulations, you've made date formatting stateful and asynchronous.

## The Over-Memoization Anti-Pattern

"Performance optimization" gone wrong:

```jsx
const OverOptimized = () => {
  // Memoizing a constant
  const title = useMemo(() => 'Hello World', []);
  
  // Memoizing a simple calculation
  const doubled = useMemo(() => 2 * 2, []);
  
  // Memoizing everything
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);
  
  const style = useMemo(() => ({
    color: 'red'
  }), []);
  
  return (
    <div style={style} onClick={handleClick}>
      {title} - {doubled}
    </div>
  );
};
```

The memoization costs more than the computation it's preventing.

## The God Component Anti-Pattern

When one component does everything:

```jsx
const App = () => {
  // 50 state variables
  const [user, setUser] = useState();
  const [posts, setPosts] = useState([]);
  const [comments, setComments] = useState([]);
  const [likes, setLikes] = useState([]);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);
  // ... 44 more
  
  // 30 useEffects
  useEffect(() => { /* fetch user */ }, []);
  useEffect(() => { /* fetch posts */ }, [user]);
  useEffect(() => { /* fetch comments */ }, [posts]);
  // ... 27 more
  
  // 40 handler functions
  const handleLogin = () => { /* 50 lines */ };
  const handleLogout = () => { /* 30 lines */ };
  const handlePostCreate = () => { /* 100 lines */ };
  // ... 37 more
  
  // 500 lines of JSX
  return (
    <div>
      {/* Everything renders here */}
    </div>
  );
};
```

One component to rule them all, and in the darkness bind them.

## The Prop Spreading Catastrophe

```jsx
const DangerousComponent = (props) => {
  return (
    <div {...props}> {/* What could go wrong? */}
      <button {...props.buttonProps}>
        <span {...props.spanProps}>
          {props.children}
        </span>
      </button>
    </div>
  );
};

// Usage
<DangerousComponent 
  onClick={handleDivClick}  // Goes to div
  buttonProps={{ onClick: handleButtonClick }} // Goes to button
  spanProps={{ onClick: handleSpanClick }} // Goes to span
  // Which handler actually fires? All of them!
/>
```

## The Early Return Maze

```jsx
const Component = ({ user, posts, permissions }) => {
  if (!user) return <Login />;
  if (!permissions) return <NoPermissions />;
  if (!posts) return <Loading />;
  if (posts.length === 0) return <NoPosts />;
  if (user.banned) return <Banned />;
  if (!user.emailVerified) return <VerifyEmail />;
  if (maintenanceMode) return <Maintenance />;
  
  // The actual component, 7 guards deep
  return <div>Content</div>;
};
```

Your component is more security checkpoint than component.

## The State Initialization Function Anti-Pattern

```jsx
// Wrong: Function runs every render
const Component = () => {
  const [data, setData] = useState(expensiveComputation());
};

// Right: Function runs once
const Component = () => {
  const [data, setData] = useState(() => expensiveComputation());
};

// But people do this anyway
const Component = () => {
  const [data, setData] = useState(() => {
    // 100 lines of initialization logic
    // Making useState do way too much
    return complexData;
  });
};
```

## In Defense of Patterns (Some of Them)

Some patterns are actually useful:

1. **Composition over Inheritance**: Actually good advice
2. **Single Responsibility**: When actually followed
3. **Custom Hooks**: For truly reusable logic
4. **Error Boundaries**: Because errors happen

But we've taken every pattern to its extreme, turning guidelines into gospel.

## The Path Forward

You're going to use these patterns. You're going to create HOCs that you'll refactor into hooks. You're going to over-memoize, then under-memoize, then give up and use a state management library.

And eventually, you'll realize that the best pattern is often no pattern - just simple, readable code that does what it needs to do.

In the next chapter, we'll explore React performance myths - why your app is slow, and why it's probably not for the reasons you think.

But first, appreciate the irony that we've created patterns to manage the complexity that the patterns themselves create. It's patterns all the way down, and confusion all the way up.