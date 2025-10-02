---
layout: default
title: "Chapter 9: Props Drilling"
nav_order: 12
---

# Chapter 9: Props Drilling: Pass It Down, Pass It Down, Pass It Down...

Props drilling is what happens when your great-grandmother needs to pass down a family heirloom to you, but instead of giving it directly, she has to give it to your grandmother, who gives it to your mother, who gives it to you. Except your grandmother and mother don't care about the heirloom at all. They're just the middlemen in this generational game of hot potato.

Welcome to React's data flow, where "unidirectional" is a fancy word for "inconvenient."

## The Christmas Tree of Props

Here's how data flows in React:

```jsx
const App = () => {
  const [user, setUser] = useState({ name: 'John', id: 1 });
  
  return <Dashboard user={user} setUser={setUser} />;
};

const Dashboard = ({ user, setUser }) => {
  return <Profile user={user} setUser={setUser} />;
};

const Profile = ({ user, setUser }) => {
  return <Settings user={user} setUser={setUser} />;
};

const Settings = ({ user, setUser }) => {
  return <AccountDetails user={user} setUser={setUser} />;
};

const AccountDetails = ({ user, setUser }) => {
  return <NameEditor user={user} setUser={setUser} />;
};

const NameEditor = ({ user, setUser }) => {
  // FINALLY! We actually use the props!
  return (
    <input 
      value={user.name}
      onChange={(e) => setUser({ ...user, name: e.target.value })}
    />
  );
};
```

Six components. Five of them don't care about `user` or `setUser`. But they all have to handle them because React said so.

## When Your Component Needs Its Grandparent's Data

The worst part about props drilling isn't the extra code. It's the coupling. Every component in the chain now depends on props it doesn't use:

```jsx
const Middle = ({ 
  // Props I actually use
  title,
  onClose,
  
  // Props I'm just passing through
  user,
  setUser,
  theme,
  setTheme,
  locale,
  setLocale,
  permissions,
  notifications,
  socket,
  analytics,
  // ... 20 more props for child components
}) => {
  return (
    <div>
      <h1>{title}</h1>
      <button onClick={onClose}>Close</button>
      
      <DeepChild 
        user={user}
        setUser={setUser}
        theme={theme}
        setTheme={setTheme}
        locale={locale}
        setLocale={setLocale}
        permissions={permissions}
        notifications={notifications}
        socket={socket}
        analytics={analytics}
        // Pass them all down!
      />
    </div>
  );
};
```

This component is now a glorified postal service, delivering packages it never opens.

## The Spread Operator "Solution"

Developers got tired of typing, so they created this pattern:

```jsx
const Middle = (props) => {
  const { title, onClose, ...restProps } = props;
  
  return (
    <div>
      <h1>{title}</h1>
      <button onClick={onClose}>Close</button>
      <DeepChild {...restProps} />
    </div>
  );
};
```

"Problem solved!" you think. Until you realize:
- You have no idea what props are being passed
- TypeScript can't help you
- Adding a new prop might break things downstream
- Removing a prop might break things you don't know about
- You've created an invisible API

## Context to the Rescue (Sort Of)

React heard our cries and gave us Context API:

```jsx
const UserContext = createContext();

const App = () => {
  const [user, setUser] = useState({ name: 'John', id: 1 });
  
  return (
    <UserContext.Provider value={{ user, setUser }}>
      <Dashboard />
    </UserContext.Provider>
  );
};

// Skip all the middle components!

const NameEditor = () => {
  const { user, setUser } = useContext(UserContext);
  
  return (
    <input 
      value={user.name}
      onChange={(e) => setUser({ ...user, name: e.target.value })}
    />
  );
};
```

Great! Now we have:
- Invisible dependencies
- Re-render cascades when context changes
- No way to track where data is used
- Provider hell at the top of our app
- The need to create a context for every piece of shared data

We solved props drilling by creating context confusion.

## The Component Prop Explosion

As components grow, so do their props:

```jsx
const Button = ({ 
  // Visual props
  variant,
  size,
  color,
  fullWidth,
  rounded,
  shadow,
  gradient,
  
  // State props
  disabled,
  loading,
  active,
  selected,
  
  // Behavior props
  onClick,
  onHover,
  onFocus,
  onBlur,
  
  // Content props
  children,
  icon,
  iconPosition,
  badge,
  badgeColor,
  
  // DOM props
  className,
  style,
  id,
  role,
  ariaLabel,
  tabIndex,
  
  // Feature flags
  enableRipple,
  enableTooltip,
  enableHapticFeedback,
  
  // Kitchen sink
  ...rest
}) => {
  // 100 lines of prop processing
};
```

This button component has more configuration options than a space shuttle.

## The Prop Naming Convention Wars

Teams spend hours debating prop names:

```jsx
// Option 1: Boolean prefix
<Modal isOpen={true} isClosable={true} hasOverlay={true} />

// Option 2: No prefix
<Modal open={true} closable={true} overlay={true} />

// Option 3: Mixed because different developers
<Modal isOpen={true} closable={true} showOverlay={true} />

// Option 4: The "handler" debate
<Button onClick={handleClick} />
<Button handleClick={handleClick} />
<Button onPress={handleClick} />

// Option 5: The "callback" pattern
<Form onSubmit={handleSubmit} />
<Form onFormSubmit={handleSubmit} />
<Form handleSubmit={handleSubmit} />
<Form submitHandler={handleSubmit} />
```

Every team has their convention. Every team member ignores it.

## The Children Props Special Case

React has a special prop called `children` that makes composition possible:

```jsx
const Container = ({ children, ...props }) => {
  return <div {...props}>{children}</div>;
};

// But wait, you can also pass children as a prop!
<Container children={<div>Content</div>} />

// Or use the special syntax
<Container>
  <div>Content</div>
</Container>

// Or both! (please don't)
<Container children="Prop children">
  Tag children
</Container>
// Guess which one wins?
```

## The Render Props Pattern (More Props!)

When regular props aren't confusing enough, there are render props:

```jsx
const DataProvider = ({ render }) => {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    fetchData().then(setData);
  }, []);
  
  return render(data);
};

// Usage
<DataProvider 
  render={(data) => (
    data ? <DisplayData data={data} /> : <Loading />
  )}
/>
```

We're passing functions that return JSX as props. It's props all the way down, and functions all the way up.

## The Default Props Graveyard

Remember defaultProps? React does, barely:

```jsx
// Class components (deprecated)
class Button extends React.Component {
  static defaultProps = {
    variant: 'primary',
    size: 'medium'
  };
}

// Function components (also deprecated)
Button.defaultProps = {
  variant: 'primary',
  size: 'medium'
};

// The "modern" way
const Button = ({ 
  variant = 'primary',
  size = 'medium',
  ...props 
}) => {
  // ...
};
```

Three ways to do the same thing, two deprecated, all confusing.

## The TypeScript Props Interface Hell

With TypeScript, props become even more fun:

```typescript
interface ButtonProps extends 
  React.ButtonHTMLAttributes<HTMLButtonElement>,
  Pick<IconProps, 'icon' | 'iconColor'>,
  Omit<BaseComponentProps, 'children'> {
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'small' | 'medium' | 'large';
  loading?: boolean;
  loadingText?: string;
  leftIcon?: React.ReactElement;
  rightIcon?: React.ReactElement;
  children?: React.ReactNode;
  as?: React.ElementType;
  // 50 more lines...
}
```

Your prop types are now more complex than your actual component.

## The Props Validation Theater

PropTypes were React's attempt at runtime type checking:

```jsx
Button.propTypes = {
  variant: PropTypes.oneOf(['primary', 'secondary']),
  size: PropTypes.oneOf(['small', 'medium', 'large']),
  onClick: PropTypes.func.isRequired,
  children: PropTypes.node,
  custom: PropTypes.shape({
    nested: PropTypes.string,
    deep: PropTypes.shape({
      deeper: PropTypes.number
    })
  })
};
```

They only work in development, they're verbose, and TypeScript made them obsolete. But some teams still use them because "belt and suspenders."

## The Prop Drilling Workarounds

Developers have created numerous patterns to avoid prop drilling:

**1. Component Composition:**
```jsx
// Instead of drilling props
<Parent user={user}>
  <Child user={user}>
    <GrandChild user={user} />
  </Child>
</Parent>

// Compose at the top
<Parent>
  <Child>
    <GrandChild user={user} />
  </Child>
</Parent>
```

**2. Compound Components:**
```jsx
<Form>
  <Form.Input name="email" />
  <Form.Submit />
</Form>
```

**3. Custom Hooks:**
```jsx
const useUser = () => {
  // Get user from context/store/somewhere
  return user;
};
```

Each workaround creates new problems while solving old ones.

## The Performance Cost

Every prop passed is:
- Memory allocated
- Comparison on re-render
- Potential re-render trigger
- TypeScript checking overhead
- DevTools tracking

When you're passing 30 props through 5 components, that adds up.

## In Defense of Props (Barely)

Props do have benefits:

1. **Explicit data flow** - You can see what data components receive
2. **Component isolation** - Components are self-contained
3. **Type safety** - With TypeScript, props can be typed
4. **Testability** - Easy to test with different props

But we've taken it too far. Not everything needs to be a prop. Not every component needs 50 configuration options.

## The Reality Check

Props drilling is what happens when you force a unidirectional data flow onto a naturally interconnected system. It's like making everyone in an office communicate only through their immediate manager. Sure, there's a clear hierarchy, but getting anything done takes forever.

## The Path Forward

You're going to drill props. You're going to pass data through components that don't care about it. You're going to create context providers, then realize you're just drilling different props. You're going to use state management libraries that are just fancy prop drilling with extra steps.

And eventually, you'll accept that in React, data doesn't flow - it trickles down through layers of components like water through coffee grounds, getting more bitter with each level.

In the next chapter, we'll explore React patterns and anti-patterns - the "best practices" that everyone follows until they become the worst practices that everyone avoids.

But first, take a moment to remember when data was just variables you could access. No props, no drilling, no context. Just data, existing in the scope where you needed it.

Those were simpler times. Now excuse me while I pass these 47 props through 12 components to update a single boolean five levels deep.

It's props all the way down, and suffering all the way up.