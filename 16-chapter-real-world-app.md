# Chapter 16: Building Something Real: A Todo App (Of Course)

Every React tutorial builds a todo app. It's the "Hello, World" of web frameworks. The ritual sacrifice we offer to the React gods. Today, we join this ancient tradition, but we're going to build it the way real developers do: over-engineered, over-complicated, and over-Reacted.

## The Simple Todo App That Wasn't

Here's how a todo app should be:

```html
<!-- The entire todo app in vanilla JS -->
<input id="input" placeholder="Add todo">
<button onclick="addTodo()">Add</button>
<ul id="list"></ul>

<script>
const todos = [];
function addTodo() {
  const input = document.getElementById('input');
  todos.push(input.value);
  document.getElementById('list').innerHTML = 
    todos.map(t => `<li>${t}</li>`).join('');
  input.value = '';
}
</script>
```

Ten lines. Works perfectly. But that's not the React way.

## The React Way: 500 Lines and Counting

First, let's set up our project:

```bash
npx create-react-app todo-app
cd todo-app
npm install redux react-redux @reduxjs/toolkit
npm install react-router-dom
npm install axios
npm install styled-components
npm install uuid
npm install react-beautiful-dnd
npm install react-hook-form
npm install yup
npm install date-fns
npm install react-query

# 847 packages installed
# 234MB node_modules
# For a todo list
```

## The Component Structure

```
src/
├── components/
│   ├── atoms/
│   │   ├── Button.jsx
│   │   ├── Input.jsx
│   │   └── Checkbox.jsx
│   ├── molecules/
│   │   ├── TodoItem.jsx
│   │   └── TodoForm.jsx
│   ├── organisms/
│   │   ├── TodoList.jsx
│   │   └── TodoFilters.jsx
│   └── templates/
│       └── TodoLayout.jsx
├── hooks/
│   ├── useTodos.js
│   ├── useLocalStorage.js
│   └── useDebounce.js
├── store/
│   ├── index.js
│   ├── todoSlice.js
│   └── filterSlice.js
├── services/
│   └── todoService.js
├── utils/
│   └── constants.js
└── App.js
```

Atomic design for a todo app. Because organization!

## The State Management

```jsx
// store/todoSlice.js
import { createSlice } from '@reduxjs/toolkit';

const todoSlice = createSlice({
  name: 'todos',
  initialState: {
    items: [],
    loading: false,
    error: null,
    filter: 'all',
    sortBy: 'date',
    searchTerm: ''
  },
  reducers: {
    addTodo: (state, action) => {
      state.items.push({
        id: Date.now(),
        text: action.payload,
        completed: false,
        createdAt: new Date().toISOString(),
        priority: 'medium',
        tags: [],
        dueDate: null
      });
    },
    toggleTodo: (state, action) => {
      const todo = state.items.find(t => t.id === action.payload);
      if (todo) todo.completed = !todo.completed;
    },
    // 20 more actions...
  }
});
```

## The Todo Component

```jsx
// components/molecules/TodoItem.jsx
import React, { memo, useCallback, useMemo } from 'react';
import styled from 'styled-components';
import { useDispatch } from 'react-redux';
import { useDrag, useDrop } from 'react-dnd';
import { formatDistanceToNow } from 'date-fns';

const TodoContainer = styled.div`
  display: flex;
  align-items: center;
  padding: ${props => props.theme.spacing.md};
  background: ${props => props.completed ? 
    props.theme.colors.completed : 
    props.theme.colors.pending};
  border-radius: ${props => props.theme.borderRadius};
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  
  &:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  }
`;

const TodoItem = memo(({ todo, index }) => {
  const dispatch = useDispatch();
  
  const [{ isDragging }, drag] = useDrag({
    type: 'TODO',
    item: { id: todo.id, index },
    collect: monitor => ({
      isDragging: monitor.isDragging()
    })
  });
  
  const handleToggle = useCallback(() => {
    dispatch(toggleTodo(todo.id));
  }, [dispatch, todo.id]);
  
  const handleDelete = useCallback(() => {
    dispatch(deleteTodo(todo.id));
  }, [dispatch, todo.id]);
  
  const timeAgo = useMemo(() => {
    return formatDistanceToNow(new Date(todo.createdAt));
  }, [todo.createdAt]);
  
  return (
    <TodoContainer 
      ref={drag}
      completed={todo.completed}
      isDragging={isDragging}
    >
      <Checkbox 
        checked={todo.completed}
        onChange={handleToggle}
      />
      <TodoText completed={todo.completed}>
        {todo.text}
      </TodoText>
      <TodoMeta>
        {timeAgo} ago
      </TodoMeta>
      <DeleteButton onClick={handleDelete}>
        ×
      </DeleteButton>
    </TodoContainer>
  );
});
```

## The Form with Validation

```jsx
// components/molecules/TodoForm.jsx
import { useForm } from 'react-hook-form';
import { yupResolver } from '@hookform/resolvers/yup';
import * as yup from 'yup';

const schema = yup.object({
  text: yup
    .string()
    .required('Todo text is required')
    .min(3, 'Todo must be at least 3 characters')
    .max(100, 'Todo must be less than 100 characters'),
  priority: yup
    .string()
    .oneOf(['low', 'medium', 'high']),
  dueDate: yup
    .date()
    .min(new Date(), 'Due date must be in the future')
});

const TodoForm = () => {
  const { register, handleSubmit, errors, reset } = useForm({
    resolver: yupResolver(schema)
  });
  
  const onSubmit = async (data) => {
    try {
      await dispatch(addTodoAsync(data));
      reset();
      toast.success('Todo added successfully!');
    } catch (error) {
      toast.error('Failed to add todo');
    }
  };
  
  return (
    <Form onSubmit={handleSubmit(onSubmit)}>
      <Input 
        {...register('text')}
        error={errors.text?.message}
        placeholder="What needs to be done?"
      />
      {/* 50 more lines of form fields */}
    </Form>
  );
};
```

## The Custom Hooks

```jsx
// hooks/useTodos.js
const useTodos = (filter = 'all') => {
  const todos = useSelector(state => state.todos.items);
  const searchTerm = useSelector(state => state.todos.searchTerm);
  const sortBy = useSelector(state => state.todos.sortBy);
  
  const filteredTodos = useMemo(() => {
    return todos
      .filter(todo => {
        if (filter === 'completed') return todo.completed;
        if (filter === 'active') return !todo.completed;
        return true;
      })
      .filter(todo => 
        todo.text.toLowerCase().includes(searchTerm.toLowerCase())
      )
      .sort((a, b) => {
        switch(sortBy) {
          case 'date':
            return new Date(b.createdAt) - new Date(a.createdAt);
          case 'priority':
            return priorityOrder[b.priority] - priorityOrder[a.priority];
          case 'alphabetical':
            return a.text.localeCompare(b.text);
          default:
            return 0;
        }
      });
  }, [todos, filter, searchTerm, sortBy]);
  
  return {
    todos: filteredTodos,
    totalCount: todos.length,
    completedCount: todos.filter(t => t.completed).length
  };
};
```

## The API Layer (For Local Storage)

```jsx
// services/todoService.js
class TodoService {
  async getTodos() {
    // Simulate API delay
    await new Promise(resolve => setTimeout(resolve, 500));
    
    const todos = localStorage.getItem('todos');
    if (!todos) throw new Error('Failed to fetch todos');
    
    return JSON.parse(todos);
  }
  
  async saveTodo(todo) {
    // More fake delay
    await new Promise(resolve => setTimeout(resolve, 300));
    
    const todos = await this.getTodos();
    todos.push(todo);
    localStorage.setItem('todos', JSON.stringify(todos));
    
    return todo;
  }
  
  // 10 more methods...
}
```

## The Tests (That Nobody Runs)

```jsx
// TodoItem.test.js
import { render, fireEvent, waitFor } from '@testing-library/react';
import { Provider } from 'react-redux';
import { store } from '../store';

describe('TodoItem', () => {
  it('should render todo text', () => {
    const todo = { id: 1, text: 'Test todo', completed: false };
    const { getByText } = render(
      <Provider store={store}>
        <TodoItem todo={todo} />
      </Provider>
    );
    
    expect(getByText('Test todo')).toBeInTheDocument();
  });
  
  // 50 more tests that test React more than your code
});
```

## The Performance "Optimizations"

```jsx
// App.js
const TodoList = lazy(() => import('./components/TodoList'));

function App() {
  return (
    <ErrorBoundary>
      <Suspense fallback={<Spinner />}>
        <Provider store={store}>
          <QueryClientProvider client={queryClient}>
            <ThemeProvider theme={theme}>
              <Router>
                <Routes>
                  <Route path="/" element={<TodoList />} />
                  <Route path="/todo/:id" element={<TodoDetail />} />
                  <Route path="/settings" element={<Settings />} />
                </Routes>
              </Router>
            </ThemeProvider>
          </QueryClientProvider>
        </Provider>
      </Suspense>
    </ErrorBoundary>
  );
}
```

## The Final Product

```
Bundle Size: 423KB (minified + gzipped)
Load Time: 2.3 seconds
First Contentful Paint: 1.8 seconds
Time to Interactive: 3.2 seconds
Lighthouse Score: 62

Features:
- Add todos ✓
- Delete todos ✓
- Mark complete ✓
- Drag and drop ✓
- Filters ✓
- Search ✓
- Sorting ✓
- Persistence ✓
- Animations ✓
- Dark mode ✓
- Keyboard shortcuts ✓
- Undo/Redo ✓
- Real-time sync ✓
- PWA support ✓
- i18n ✓

Bugs:
- Crashes on iOS Safari
- Search doesn't work with special characters
- Drag and drop breaks after filter change
- Memory leak in useEffect
- Race condition in API calls
```

## The Reality Check

We built a todo app with:
- 15 components
- 8 custom hooks
- 3 context providers
- Redux store
- 20+ dependencies
- 500+ lines of code
- 2MB bundle

The vanilla version was 10 lines.

But hey, ours has:
- Scalability (for your massive todo empire)
- Maintainability (by developers who understand Redux)
- Testability (tests we'll never write)
- Type safety (TypeScript we didn't add)
- Performance optimizations (that made it slower)

## The Lessons Learned

1. **Simple problems don't need complex solutions**
2. **But complex solutions get you hired**
3. **Over-engineering is the default**
4. **The ecosystem encourages complexity**
5. **Nobody needs a 2MB todo app**
6. **But everyone builds one anyway**

## The Path Forward

You're going to build a todo app. It's going to be over-engineered. You're going to add features nobody asked for. You're going to optimize performance that doesn't need optimizing. You're going to use patterns you don't understand for problems you don't have.

And it's going to work. Sort of. Slowly. With bugs. But it'll be "production ready" and "enterprise grade" and "following best practices."

In the next chapter, we'll look at alternatives to React - the grass that seems greener until you realize it's just different grass with different problems.

But first, appreciate the absurdity: we turned a 10-line problem into a 500-line solution and called it progress.

Welcome to modern web development.