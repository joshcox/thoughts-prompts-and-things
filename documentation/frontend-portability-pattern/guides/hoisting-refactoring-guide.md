# Hoisting Refactoring Guide

This guide provides step-by-step instructions for refactoring components using the hoisting algorithm to achieve layer separation and portability.

- [Hoisting Refactoring Guide](#hoisting-refactoring-guide)
  - [Prerequisites](#prerequisites)
  - [The Three-Step Process](#the-three-step-process)
    - [Step 1: Identify External Dependencies](#step-1-identify-external-dependencies)
      - [What to Look For](#what-to-look-for)
      - [What to Keep Internal](#what-to-keep-internal)
      - [Decision Tree](#decision-tree)
    - [Step 2: Convert Dependencies to Props Interface](#step-2-convert-dependencies-to-props-interface)
      - [Pattern 2a: State → Data Props](#pattern-2a-state--data-props)
      - [Pattern 2b: Data Fetching → Data Props + Loading States](#pattern-2b-data-fetching--data-props--loading-states)
      - [Pattern 2c: Handlers → Callback Props](#pattern-2c-handlers--callback-props)
      - [Pattern 2d: Design the Complete Props Interface](#pattern-2d-design-the-complete-props-interface)
    - [Step 3: Move Ownership to Domain Layer](#step-3-move-ownership-to-domain-layer)
      - [Option A: Domain Hook](#option-a-domain-hook)
      - [Option B: Domain Container Component](#option-b-domain-container-component)
      - [Option C: Inline Container](#option-c-inline-container)
  - [Common Refactoring Patterns](#common-refactoring-patterns)
    - [Pattern 1: Form with API Submission](#pattern-1-form-with-api-submission)
      - [Before: Mixed Concerns](#before-mixed-concerns)
      - [After: Presentation Component](#after-presentation-component)
      - [After: Domain Container](#after-domain-container)
    - [Pattern 2: List with Filtering and Actions](#pattern-2-list-with-filtering-and-actions)
      - [Before: Mixed Concerns](#before-mixed-concerns-1)
      - [After: Presentation Component](#after-presentation-component-1)
      - [After: Domain Container](#after-domain-container-1)
    - [Pattern 3: Component with Navigation](#pattern-3-component-with-navigation)
      - [Before: Mixed Concerns](#before-mixed-concerns-2)
      - [After: Presentation Component](#after-presentation-component-2)
      - [After: Domain Container](#after-domain-container-2)
  - [Complete Example: Task Manager](#complete-example-task-manager)
    - [Before: Tightly Coupled Component](#before-tightly-coupled-component)
    - [After: Clean Presentation Component](#after-clean-presentation-component)
    - [After: Domain Container](#after-domain-container-3)
    - [After: Domain Hook](#after-domain-hook)
    - [After: Data Layer](#after-data-layer)
  - [Testing Hoisted Components](#testing-hoisted-components)
    - [Testing Presentation Components](#testing-presentation-components)
    - [Testing Domain Logic](#testing-domain-logic)
  - [Troubleshooting](#troubleshooting)
    - ["Too Many Props Syndrome"](#too-many-props-syndrome)
    - ["Circular Dependencies"](#circular-dependencies)
    - ["Where Does Validation Go?"](#where-does-validation-go)
    - ["Component Feels Too Dumb"](#component-feels-too-dumb)
  - [Next Steps](#next-steps)
  - [Additional Resources](#additional-resources)


## Prerequisites

Before starting, familiarize yourself with:
- [Architecture: Three-Layer Pattern](../architecture.md#core-pattern-three-layer-architecture)
- [Technical Strategy: Hoisting Algorithm](../technical-strategy/hoisting-algorithm.md)

## The Three-Step Process

### Step 1: Identify External Dependencies

Scan the component for any dependency that reaches outside the component's local scope.

#### What to Look For

**Non-Local State:**
- Context consumers: `useContext`, `useSelector`, `useRecoilValue`
- Global state hooks: `useRecoilState`, `useAtom`, `useStore`
- Router hooks: `useParams`, `useNavigate`, `useLocation`, `useSearchParams`
- Authentication hooks: `useAuth`, `useUser`, `useSession`

**Data Fetching:**
- API calls: `fetch`, `axios.get`, `api.post`
- GraphQL: `useQuery`, `useMutation`, `useLazyQuery`
- Network hooks: `useSWR`, `useQuery` (React Query)
- Data loading side effects in `useEffect`

**Side-Effectful Handlers:**
- Handlers that modify non-local state
- Handlers that trigger API calls
- Handlers that navigate or manipulate the router
- Handlers that interact with external services (analytics, logging)

#### What to Keep Internal

These concerns should remain in the component:
- Local UI state: collapsed/expanded, selected tab, input focus, hover states
- UI-only side effects: scroll position, animation triggers
- Derived values calculated from props
- Event handlers that only manipulate local state

#### Decision Tree

```
Is this state/data/handler...
├─ Used only for UI presentation? → Keep internal
├─ Affects other components? → Hoist
├─ Fetched from network? → Hoist
├─ From global state/context? → Hoist
├─ Triggers navigation? → Hoist
└─ Local form input? → Keep internal (until submission)
```

### Step 2: Convert Dependencies to Props Interface

Transform each identified external dependency into a prop that will be injected from the parent.

#### Pattern 2a: State → Data Props

```typescript
// Before: Direct context access
const { user } = useAuth();
const { items } = useSelector(state => state.items);

// After: Props interface
interface ComponentProps {
  user: User;
  items: Item[];
}
```

#### Pattern 2b: Data Fetching → Data Props + Loading States

```typescript
// Before: Direct fetching
const { data, loading, error } = useQuery(GET_ITEMS);

// After: Props interface
interface ComponentProps {
  items: Item[];
  isLoading: boolean;
  error?: Error;
}
```

#### Pattern 2c: Handlers → Callback Props

```typescript
// Before: Direct state manipulation
const dispatch = useDispatch();
const handleDelete = (id: string) => {
  dispatch(deleteItem(id));
};

// After: Props interface
interface ComponentProps {
  onDeleteItem: (id: string) => void;
}
```

#### Pattern 2d: Design the Complete Props Interface

Combine all identified dependencies into a cohesive interface:

```typescript
interface ComponentProps {
  // Data
  items: Item[];
  user: User;
  
  // Loading states
  isLoading: boolean;
  error?: Error;
  
  // Callbacks
  onDeleteItem: (id: string) => void;
  onUpdateItem: (id: string, updates: Partial<Item>) => void;
  onRefresh: () => void;
}
```

### Step 3: Move Ownership to Domain Layer

Create a domain container (hook or component) that owns the hoisted concerns and provides them to the presentation component.

#### Option A: Domain Hook

```typescript
// domain/useItemManager.ts
export function useItemManager() {
  const { user } = useAuth();
  const dispatch = useDispatch();
  const items = useSelector(state => state.items);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<Error>();

  const handleDeleteItem = useCallback(async (id: string) => {
    setIsLoading(true);
    try {
      await deleteItem(id); // Data layer call
      dispatch(removeItem(id));
    } catch (err) {
      setError(err);
    } finally {
      setIsLoading(false);
    }
  }, [dispatch]);

  // ... more handlers

  return {
    items,
    user,
    isLoading,
    error,
    onDeleteItem: handleDeleteItem,
    // ... more props
  };
}
```

#### Option B: Domain Container Component

```typescript
// domain/ItemManagerContainer.tsx
export function ItemManagerContainer() {
  const domainProps = useItemManager();
  
  return <ItemList {...domainProps} />;
}
```

#### Option C: Inline Container

```typescript
// For simple cases
export function ItemManagerPage() {
  const domainProps = useItemManager();
  
  return (
    <PageLayout>
      <ItemList {...domainProps} />
    </PageLayout>
  );
}
```

## Common Refactoring Patterns

### Pattern 1: Form with API Submission

#### Before: Mixed Concerns

```typescript
function UserForm() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [submitting, setSubmitting] = useState(false);
  
  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    setSubmitting(true);
    try {
      await api.updateUser({ name, email });
      toast.success('Saved!');
    } catch (err) {
      toast.error('Failed to save');
    } finally {
      setSubmitting(false);
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input value={name} onChange={e => setName(e.target.value)} />
      <input value={email} onChange={e => setEmail(e.target.value)} />
      <button disabled={submitting}>Save</button>
    </form>
  );
}
```

#### After: Presentation Component

```typescript
interface UserFormProps {
  initialName: string;
  initialEmail: string;
  isSubmitting: boolean;
  onSubmit: (data: { name: string; email: string }) => void;
}

function UserForm({ initialName, initialEmail, isSubmitting, onSubmit }: UserFormProps) {
  const [name, setName] = useState(initialName); // Local: form state
  const [email, setEmail] = useState(initialEmail); // Local: form state
  
  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();
    onSubmit({ name, email });
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input value={name} onChange={e => setName(e.target.value)} />
      <input value={email} onChange={e => setEmail(e.target.value)} />
      <button disabled={isSubmitting}>Save</button>
    </form>
  );
}
```

#### After: Domain Container

```typescript
function UserFormContainer() {
  const { user } = useAuth();
  const [submitting, setSubmitting] = useState(false);
  
  const handleSubmit = async (data: { name: string; email: string }) => {
    setSubmitting(true);
    try {
      await updateUser(user.id, data); // Data layer
      toast.success('Saved!');
    } catch (err) {
      toast.error('Failed to save');
    } finally {
      setSubmitting(false);
    }
  };
  
  return (
    <UserForm
      initialName={user.name}
      initialEmail={user.email}
      isSubmitting={submitting}
      onSubmit={handleSubmit}
    />
  );
}
```

### Pattern 2: List with Filtering and Actions

#### Before: Mixed Concerns

```typescript
function ItemList() {
  const { items } = useItems();
  const [filter, setFilter] = useState('');
  const dispatch = useDispatch();
  
  const filtered = items.filter(item => 
    item.name.toLowerCase().includes(filter.toLowerCase())
  );
  
  const handleDelete = (id: string) => {
    dispatch(deleteItem(id));
  };
  
  return (
    <div>
      <input 
        value={filter} 
        onChange={e => setFilter(e.target.value)} 
        placeholder="Filter..."
      />
      {filtered.map(item => (
        <ItemCard 
          key={item.id} 
          item={item} 
          onDelete={() => handleDelete(item.id)}
        />
      ))}
    </div>
  );
}
```

#### After: Presentation Component

```typescript
interface ItemListProps {
  items: Item[];
  onDeleteItem: (id: string) => void;
}

function ItemList({ items, onDeleteItem }: ItemListProps) {
  const [filter, setFilter] = useState(''); // Local: UI filter
  
  const filtered = items.filter(item => 
    item.name.toLowerCase().includes(filter.toLowerCase())
  );
  
  return (
    <div>
      <input 
        value={filter} 
        onChange={e => setFilter(e.target.value)} 
        placeholder="Filter..."
      />
      {filtered.map(item => (
        <ItemCard 
          key={item.id} 
          item={item} 
          onDelete={() => onDeleteItem(item.id)}
        />
      ))}
    </div>
  );
}
```

#### After: Domain Container

```typescript
function ItemListContainer() {
  const { items } = useItems();
  const dispatch = useDispatch();
  
  const handleDelete = useCallback((id: string) => {
    dispatch(deleteItem(id));
  }, [dispatch]);
  
  return <ItemList items={items} onDeleteItem={handleDelete} />;
}
```

### Pattern 3: Component with Navigation

#### Before: Mixed Concerns

```typescript
function ItemCard({ item }: { item: Item }) {
  const navigate = useNavigate();
  
  const handleClick = () => {
    navigate(`/items/${item.id}`);
  };
  
  return (
    <div onClick={handleClick}>
      <h3>{item.name}</h3>
      <p>{item.description}</p>
    </div>
  );
}
```

#### After: Presentation Component

```typescript
interface ItemCardProps {
  item: Item;
  onSelect: (id: string) => void;
}

function ItemCard({ item, onSelect }: ItemCardProps) {
  return (
    <div onClick={() => onSelect(item.id)}>
      <h3>{item.name}</h3>
      <p>{item.description}</p>
    </div>
  );
}
```

#### After: Domain Container

```typescript
function ItemCardContainer({ item }: { item: Item }) {
  const navigate = useNavigate();
  
  const handleSelect = useCallback((id: string) => {
    navigate(`/items/${id}`);
  }, [navigate]);
  
  return <ItemCard item={item} onSelect={handleSelect} />;
}
```

## Complete Example: Task Manager

### Before: Tightly Coupled Component

```typescript
// TaskManager.tsx - Mixed concerns
import { useState, useEffect } from 'react';
import { useAuth } from '../auth/AuthContext';
import { useNavigate } from 'react-router-dom';
import { useTasks } from '../hooks/useTasks';
import { api } from '../api/client';

function TaskManager() {
  const { user } = useAuth();
  const navigate = useNavigate();
  const { tasks, refetch } = useTasks();
  const [selectedTab, setSelectedTab] = useState<'all' | 'active' | 'completed'>('all');
  const [creating, setCreating] = useState(false);
  const [newTaskTitle, setNewTaskTitle] = useState('');
  
  useEffect(() => {
    if (!user) {
      navigate('/login');
    }
  }, [user, navigate]);
  
  const handleCreateTask = async () => {
    if (!newTaskTitle.trim()) return;
    
    setCreating(true);
    try {
      await api.post('/tasks', { 
        title: newTaskTitle,
        userId: user.id 
      });
      setNewTaskTitle('');
      await refetch();
    } catch (err) {
      alert('Failed to create task');
    } finally {
      setCreating(false);
    }
  };
  
  const handleToggleComplete = async (taskId: string, completed: boolean) => {
    try {
      await api.patch(`/tasks/${taskId}`, { completed: !completed });
      await refetch();
    } catch (err) {
      alert('Failed to update task');
    }
  };
  
  const handleDeleteTask = async (taskId: string) => {
    try {
      await api.delete(`/tasks/${taskId}`);
      await refetch();
    } catch (err) {
      alert('Failed to delete task');
    }
  };
  
  const filteredTasks = tasks.filter(task => {
    if (selectedTab === 'active') return !task.completed;
    if (selectedTab === 'completed') return task.completed;
    return true;
  });
  
  return (
    <div className="task-manager">
      <h1>Tasks for {user?.name}</h1>
      
      <div className="tabs">
        <button 
          className={selectedTab === 'all' ? 'active' : ''}
          onClick={() => setSelectedTab('all')}
        >
          All
        </button>
        <button 
          className={selectedTab === 'active' ? 'active' : ''}
          onClick={() => setSelectedTab('active')}
        >
          Active
        </button>
        <button 
          className={selectedTab === 'completed' ? 'active' : ''}
          onClick={() => setSelectedTab('completed')}
        >
          Completed
        </button>
      </div>
      
      <div className="create-task">
        <input
          value={newTaskTitle}
          onChange={e => setNewTaskTitle(e.target.value)}
          placeholder="New task..."
          disabled={creating}
        />
        <button onClick={handleCreateTask} disabled={creating}>
          {creating ? 'Creating...' : 'Add Task'}
        </button>
      </div>
      
      <ul className="task-list">
        {filteredTasks.map(task => (
          <li key={task.id}>
            <input
              type="checkbox"
              checked={task.completed}
              onChange={() => handleToggleComplete(task.id, task.completed)}
            />
            <span className={task.completed ? 'completed' : ''}>
              {task.title}
            </span>
            <button onClick={() => handleDeleteTask(task.id)}>
              Delete
            </button>
          </li>
        ))}
      </ul>
    </div>
  );
}

export default TaskManager;
```

### After: Clean Presentation Component

```typescript
// presentation/TaskManager.tsx - Context-free, portable
import { useState } from 'react';

export interface Task {
  id: string;
  title: string;
  completed: boolean;
}

export interface TaskManagerProps {
  userName: string;
  tasks: Task[];
  isCreating: boolean;
  onCreateTask: (title: string) => void;
  onToggleComplete: (taskId: string) => void;
  onDeleteTask: (taskId: string) => void;
}

export function TaskManager({
  userName,
  tasks,
  isCreating,
  onCreateTask,
  onToggleComplete,
  onDeleteTask,
}: TaskManagerProps) {
  // Local UI state only
  const [selectedTab, setSelectedTab] = useState<'all' | 'active' | 'completed'>('all');
  const [newTaskTitle, setNewTaskTitle] = useState('');
  
  const handleCreate = () => {
    if (newTaskTitle.trim()) {
      onCreateTask(newTaskTitle);
      setNewTaskTitle('');
    }
  };
  
  const filteredTasks = tasks.filter(task => {
    if (selectedTab === 'active') return !task.completed;
    if (selectedTab === 'completed') return task.completed;
    return true;
  });
  
  return (
    <div className="task-manager">
      <h1>Tasks for {userName}</h1>
      
      <div className="tabs">
        <button 
          className={selectedTab === 'all' ? 'active' : ''}
          onClick={() => setSelectedTab('all')}
        >
          All
        </button>
        <button 
          className={selectedTab === 'active' ? 'active' : ''}
          onClick={() => setSelectedTab('active')}
        >
          Active
        </button>
        <button 
          className={selectedTab === 'completed' ? 'active' : ''}
          onClick={() => setSelectedTab('completed')}
        >
          Completed
        </button>
      </div>
      
      <div className="create-task">
        <input
          value={newTaskTitle}
          onChange={e => setNewTaskTitle(e.target.value)}
          placeholder="New task..."
          disabled={isCreating}
          onKeyPress={e => e.key === 'Enter' && handleCreate()}
        />
        <button onClick={handleCreate} disabled={isCreating}>
          {isCreating ? 'Creating...' : 'Add Task'}
        </button>
      </div>
      
      <ul className="task-list">
        {filteredTasks.map(task => (
          <li key={task.id}>
            <input
              type="checkbox"
              checked={task.completed}
              onChange={() => onToggleComplete(task.id)}
            />
            <span className={task.completed ? 'completed' : ''}>
              {task.title}
            </span>
            <button onClick={() => onDeleteTask(task.id)}>
              Delete
            </button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### After: Domain Container

```typescript
// domain/TaskManagerContainer.tsx
import { useState, useCallback } from 'react';
import { useAuth } from '../auth/AuthContext';
import { useTaskManager } from './useTaskManager';
import { TaskManager } from '../presentation/TaskManager';

export function TaskManagerContainer() {
  const { user } = useAuth();
  const {
    tasks,
    isLoading,
    createTask,
    toggleComplete,
    deleteTask,
  } = useTaskManager(user?.id);
  
  const [isCreating, setIsCreating] = useState(false);
  
  const handleCreateTask = useCallback(async (title: string) => {
    setIsCreating(true);
    try {
      await createTask(title);
    } finally {
      setIsCreating(false);
    }
  }, [createTask]);
  
  if (!user) {
    return <div>Please log in</div>;
  }
  
  if (isLoading) {
    return <div>Loading tasks...</div>;
  }
  
  return (
    <TaskManager
      userName={user.name}
      tasks={tasks}
      isCreating={isCreating}
      onCreateTask={handleCreateTask}
      onToggleComplete={toggleComplete}
      onDeleteTask={deleteTask}
    />
  );
}
```

### After: Domain Hook

```typescript
// domain/useTaskManager.ts
import { useState, useEffect, useCallback } from 'react';
import { fetchTasks, createTask as apiCreateTask, updateTask, deleteTask as apiDeleteTask } from '../data/tasks';
import type { Task } from '../presentation/TaskManager';

export function useTaskManager(userId: string | undefined) {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  
  const loadTasks = useCallback(async () => {
    if (!userId) return;
    
    setIsLoading(true);
    setError(null);
    try {
      const data = await fetchTasks(userId);
      setTasks(data);
    } catch (err) {
      setError(err as Error);
    } finally {
      setIsLoading(false);
    }
  }, [userId]);
  
  useEffect(() => {
    loadTasks();
  }, [loadTasks]);
  
  const createTask = useCallback(async (title: string) => {
    if (!userId) return;
    
    try {
      const newTask = await apiCreateTask({ title, userId });
      setTasks(prev => [...prev, newTask]);
    } catch (err) {
      setError(err as Error);
      throw err;
    }
  }, [userId]);
  
  const toggleComplete = useCallback(async (taskId: string) => {
    const task = tasks.find(t => t.id === taskId);
    if (!task) return;
    
    try {
      const updated = await updateTask(taskId, { completed: !task.completed });
      setTasks(prev => prev.map(t => t.id === taskId ? updated : t));
    } catch (err) {
      setError(err as Error);
      throw err;
    }
  }, [tasks]);
  
  const deleteTask = useCallback(async (taskId: string) => {
    try {
      await apiDeleteTask(taskId);
      setTasks(prev => prev.filter(t => t.id !== taskId));
    } catch (err) {
      setError(err as Error);
      throw err;
    }
  }, []);
  
  return {
    tasks,
    isLoading,
    error,
    createTask,
    toggleComplete,
    deleteTask,
    refresh: loadTasks,
  };
}
```

### After: Data Layer

```typescript
// data/tasks.ts
import { api } from './client';

export interface TaskDTO {
  id: string;
  title: string;
  completed: boolean;
  userId: string;
  createdAt: string;
  updatedAt: string;
}

export interface Task {
  id: string;
  title: string;
  completed: boolean;
}

// Transform DTOs to domain models
function toTask(dto: TaskDTO): Task {
  return {
    id: dto.id,
    title: dto.title,
    completed: dto.completed,
  };
}

export async function fetchTasks(userId: string): Promise<Task[]> {
  const response = await api.get<TaskDTO[]>(`/tasks?userId=${userId}`);
  return response.data.map(toTask);
}

export async function createTask(data: { title: string; userId: string }): Promise<Task> {
  const response = await api.post<TaskDTO>('/tasks', data);
  return toTask(response.data);
}

export async function updateTask(taskId: string, updates: Partial<Task>): Promise<Task> {
  const response = await api.patch<TaskDTO>(`/tasks/${taskId}`, updates);
  return toTask(response.data);
}

export async function deleteTask(taskId: string): Promise<void> {
  await api.delete(`/tasks/${taskId}`);
}
```

## Testing Hoisted Components

### Testing Presentation Components

Presentation components are easy to test because they only depend on props:

```typescript
import { render, screen, fireEvent } from '@testing-library/react';
import { TaskManager } from './TaskManager';

describe('TaskManager', () => {
  const mockTasks = [
    { id: '1', title: 'Task 1', completed: false },
    { id: '2', title: 'Task 2', completed: true },
  ];
  
  it('renders all tasks', () => {
    const onDelete = jest.fn();
    const onToggle = jest.fn();
    const onCreate = jest.fn();
    
    render(
      <TaskManager
        userName="John"
        tasks={mockTasks}
        isCreating={false}
        onCreateTask={onCreate}
        onToggleComplete={onToggle}
        onDeleteTask={onDelete}
      />
    );
    
    expect(screen.getByText('Task 1')).toBeInTheDocument();
    expect(screen.getByText('Task 2')).toBeInTheDocument();
  });
  
  it('calls onDeleteTask when delete is clicked', () => {
    const onDelete = jest.fn();
    
    render(
      <TaskManager
        userName="John"
        tasks={mockTasks}
        isCreating={false}
        onCreateTask={jest.fn()}
        onToggleComplete={jest.fn()}
        onDeleteTask={onDelete}
      />
    );
    
    const deleteButtons = screen.getAllByText('Delete');
    fireEvent.click(deleteButtons[0]);
    
    expect(onDelete).toHaveBeenCalledWith('1');
  });
  
  it('filters tasks based on selected tab', () => {
    render(
      <TaskManager
        userName="John"
        tasks={mockTasks}
        isCreating={false}
        onCreateTask={jest.fn()}
        onToggleComplete={jest.fn()}
        onDeleteTask={jest.fn()}
      />
    );
    
    // Click "Active" tab
    fireEvent.click(screen.getByText('Active'));
    
    expect(screen.getByText('Task 1')).toBeInTheDocument();
    expect(screen.queryByText('Task 2')).not.toBeInTheDocument();
  });
});
```

### Testing Domain Logic

Domain hooks can be tested independently:

```typescript
import { renderHook, act, waitFor } from '@testing-library/react';
import { useTaskManager } from './useTaskManager';
import * as dataLayer from '../data/tasks';

jest.mock('../data/tasks');

describe('useTaskManager', () => {
  it('loads tasks on mount', async () => {
    const mockTasks = [{ id: '1', title: 'Task 1', completed: false }];
    (dataLayer.fetchTasks as jest.Mock).mockResolvedValue(mockTasks);
    
    const { result } = renderHook(() => useTaskManager('user1'));
    
    expect(result.current.isLoading).toBe(true);
    
    await waitFor(() => {
      expect(result.current.isLoading).toBe(false);
    });
    
    expect(result.current.tasks).toEqual(mockTasks);
  });
  
  it('handles create action', async () => {
    const newTask = { id: '2', title: 'New Task', completed: false };
    (dataLayer.fetchTasks as jest.Mock).mockResolvedValue([]);
    (dataLayer.createTask as jest.Mock).mockResolvedValue(newTask);
    
    const { result } = renderHook(() => useTaskManager('user1'));
    
    await act(async () => {
      await result.current.createTask('New Task');
    });
    
    expect(dataLayer.createTask).toHaveBeenCalledWith({
      title: 'New Task',
      userId: 'user1',
    });
    expect(result.current.tasks).toContainEqual(newTask);
  });
});
```

## Troubleshooting

### "Too Many Props Syndrome"

**Problem:** The props interface has 10+ props and feels unwieldy.

**Solution:** 
- Group related props into objects: `userData: UserData`, `taskActions: TaskActions`
- Consider splitting the component into smaller pieces
- Verify you're not over-abstracting - some coupling might be acceptable

### "Circular Dependencies"

**Problem:** Domain layer needs to know about presentation types.

**Solution:**
- Define shared types in a separate `types` file
- Export types from presentation and import in domain
- This is acceptable - domain *should* know the presentation's prop interface

### "Where Does Validation Go?"

**Problem:** Should form validation live in presentation or domain?

**Solution:**
- **UI validation** (required fields, format): Presentation layer
- **Business validation** (uniqueness, authorization): Domain layer
- Pass validation errors as props to presentation

### "Component Feels Too Dumb"

**Problem:** The presentation component feels like it's just passing props around.

**Solution:**
- This is often correct! Presentation *should* be simple
- Look for UI logic that can stay: filtering, sorting, expanding/collapsing
- Don't over-abstract - it's okay to have some logic in presentation

## Next Steps

1. **Start small**: Pick a single, non-critical component
2. **Identify dependencies**: Use the checklist from Step 1
3. **Create the interface**: Design clean props (Step 2)
4. **Extract the container**: Move logic to domain (Step 3)
5. **Test both layers**: Verify presentation and domain independently
6. **Iterate**: Apply learnings to more complex components

## Additional Resources

- [Technical Strategy: Hoisting Algorithm](../technical-strategy/hoisting-algorithm.md) - Deep dive into execution principles
- [Architecture: Three-Layer Pattern](../architecture.md#core-pattern-three-layer-architecture) - Overall pattern context
- [Architecture: Compositional UI](../architecture.md#compositional-ui-techniques) - How to compose hoisted components

