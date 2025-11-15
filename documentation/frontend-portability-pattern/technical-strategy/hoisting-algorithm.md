# Hoisting Algorithm: Technical Strategy

## Conceptual Overview

### What is Hoisting?

In the context of frontend portability, **hoisting** refers to the systematic process of extracting application-specific concerns from UI components and moving them to a higher-level orchestration layer. This process transforms tightly-coupled components into reusable, context-free presentation components.

The term "hoisting" comes from the act of "lifting" or "raising" dependencies out of a component, similar to how hoisting works in JavaScript where declarations are moved to the top of their scope. In our case, we're moving concerns up the component hierarchy.

### Why Hoisting Creates Portability

When a component directly depends on application-specific context (global state, routing, authentication, API clients, etc.), it becomes bound to that specific application environment. By hoisting these dependencies out and injecting them as props, the component becomes:

1. **Environment-agnostic**: No assumptions about the surrounding application context
2. **Composable**: Can be used in any application that can provide the required props
3. **Testable**: Can be tested in isolation by passing mock props
4. **Portable**: Can be moved between applications without modification

The hoisting process is the mechanism that enables the three-layer architecture (Data, Domain, Presentation) to function effectively.

### Relationship to Layer Separation

Hoisting is the practical technique for enforcing layer boundaries:

- **Presentation Layer**: After hoisting, contains only pure UI components that receive all dependencies via props
- **Domain Layer**: Becomes the owner of hoisted concerns, orchestrating state, side effects, and business logic
- **Data Layer**: Remains the source of truth for data access, called by the domain layer

The hoisting algorithm is applied at the boundary between Presentation and Domain, ensuring that presentation components remain truly context-free while domain logic remains centralized.

## The Algorithm: Step-by-Step

### Step 1: Identify External Dependencies

Scan the component for any dependency that reaches outside the component's local scope:

#### 1a. Non-Local State
- Context consumers (e.g., `useContext`, `useSelector`)
- Global state hooks (e.g., `useRecoilState`, `useAtom`)
- Router hooks (e.g., `useParams`, `useNavigate`, `useLocation`)
- Authentication hooks (e.g., `useAuth`, `useUser`)

#### 1b. Data Fetching
- API calls (e.g., `fetch`, `axios`)
- GraphQL queries/mutations (e.g., `useQuery`, `useMutation`)
- Any hook that performs network requests
- Data loading side effects

#### 1c. Side-Effectful Handlers
- Handlers that modify non-local state
- Handlers that trigger API calls
- Handlers that navigate or manipulate the router
- Handlers that interact with external services

#### What to Keep Internal
- Local UI state (collapsed/expanded, selected tab, input focus)
- UI-only side effects (scroll position, animation triggers)
- Derived values calculated from props
- Event handlers that only manipulate local state

### Step 2: Convert Dependencies to Props Interface

Transform each identified external dependency into a prop that will be injected from the parent:

#### 2a. State → Data Props
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

#### 2b. Data Fetching → Data Props + Loading States
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

#### 2c. Handlers → Callback Props
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

#### 2d. Design the Complete Props Interface
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

Create a domain container (hook or component) that owns the hoisted concerns and provides them to the presentation component:

#### 3a. Create a Domain Hook
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

  const handleUpdateItem = useCallback(async (id: string, updates: Partial<Item>) => {
    setIsLoading(true);
    try {
      const updated = await updateItem(id, updates); // Data layer call
      dispatch(setItem(updated));
    } catch (err) {
      setError(err);
    } finally {
      setIsLoading(false);
    }
  }, [dispatch]);

  const handleRefresh = useCallback(async () => {
    setIsLoading(true);
    try {
      const fresh = await fetchItems(); // Data layer call
      dispatch(setItems(fresh));
    } catch (err) {
      setError(err);
    } finally {
      setIsLoading(false);
    }
  }, [dispatch]);

  return {
    items,
    user,
    isLoading,
    error,
    onDeleteItem: handleDeleteItem,
    onUpdateItem: handleUpdateItem,
    onRefresh: handleRefresh,
  };
}
```

#### 3b. Create a Domain Container Component
```typescript
// domain/ItemManagerContainer.tsx
export function ItemManagerContainer() {
  const domainProps = useItemManager();
  
  return <ItemList {...domainProps} />;
}
```

#### 3c. Alternative: Inline Container
```typescript
// For simple cases, the container can be inline
export function ItemManagerPage() {
  const domainProps = useItemManager();
  
  return (
    <PageLayout>
      <ItemList {...domainProps} />
    </PageLayout>
  );
}
```

## Implementation Details

### Identifying State That Should Be Hoisted vs Local

Use this decision tree:

1. **Does the state affect anything outside this component?**
   - Yes → Hoist it
   - No → Continue to question 2

2. **Does the state need to persist across unmount/remount?**
   - Yes → Hoist it
   - No → Continue to question 3

3. **Is the state derived from external data sources?**
   - Yes → Hoist it
   - No → Keep it local

4. **Is the state purely for UI presentation (collapsed, selected, focused)?**
   - Yes → Keep it local
   - No → Hoist it

**Examples of Local State:**
- Modal open/closed
- Dropdown expanded/collapsed
- Selected tab index
- Form input values (before submission)
- Hover/focus states
- Animation states

**Examples of Hoisted State:**
- User authentication status
- Fetched data from APIs
- Form submission results
- Navigation state
- Shopping cart contents
- Application preferences

### Refactoring Patterns for Common Scenarios

#### Pattern 1: Form with API Submission

**Before:**
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

**After (Presentation):**
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

**After (Domain):**
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

#### Pattern 2: List with Filtering and Actions

**Before:**
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

**After (Presentation):**
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

**After (Domain):**
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

#### Pattern 3: Component with Navigation

**Before:**
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

**After (Presentation):**
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

**After (Domain):**
```typescript
function ItemCardContainer({ item }: { item: Item }) {
  const navigate = useNavigate();
  
  const handleSelect = useCallback((id: string) => {
    navigate(`/items/${id}`);
  }, [navigate]);
  
  return <ItemCard item={item} onSelect={handleSelect} />;
}
```

### Handling Side Effects and Data Fetching

#### Data Fetching Should Live in Domain

**Domain Hook Pattern:**
```typescript
// domain/useItemData.ts
export function useItemData(itemId: string) {
  const [item, setItem] = useState<Item | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    let cancelled = false;
    
    async function load() {
      setIsLoading(true);
      setError(null);
      
      try {
        const data = await fetchItem(itemId); // Data layer
        if (!cancelled) {
          setItem(data);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err);
        }
      } finally {
        if (!cancelled) {
          setIsLoading(false);
        }
      }
    }
    
    load();
    
    return () => {
      cancelled = true;
    };
  }, [itemId]);
  
  return { item, isLoading, error };
}
```

**Using the Domain Hook:**
```typescript
function ItemDetailContainer({ itemId }: { itemId: string }) {
  const { item, isLoading, error } = useItemData(itemId);
  
  if (isLoading) return <LoadingSpinner />;
  if (error) return <ErrorMessage error={error} />;
  if (!item) return <NotFound />;
  
  return <ItemDetail item={item} />;
}
```

#### Side Effects Follow the Same Pattern

```typescript
// Domain manages side effects
function ItemActionsContainer({ itemId }: { itemId: string }) {
  const [saving, setSaving] = useState(false);
  
  const handleSave = useCallback(async (updates: Partial<Item>) => {
    setSaving(true);
    try {
      await updateItem(itemId, updates); // Data layer
      analytics.track('item_updated'); // Side effect in domain
    } finally {
      setSaving(false);
    }
  }, [itemId]);
  
  return <ItemActions onSave={handleSave} isSaving={saving} />;
}
```

### Testing Hoisted Components

One of the major benefits of hoisting is improved testability. Presentation components can be tested without mocking complex application context.

#### Testing Presentation Components

```typescript
import { render, screen, fireEvent } from '@testing-library/react';
import { ItemList } from './ItemList';

describe('ItemList', () => {
  const mockItems = [
    { id: '1', name: 'Item 1', description: 'First' },
    { id: '2', name: 'Item 2', description: 'Second' },
  ];
  
  it('renders all items', () => {
    const onDelete = jest.fn();
    render(<ItemList items={mockItems} onDeleteItem={onDelete} />);
    
    expect(screen.getByText('Item 1')).toBeInTheDocument();
    expect(screen.getByText('Item 2')).toBeInTheDocument();
  });
  
  it('calls onDeleteItem when delete is clicked', () => {
    const onDelete = jest.fn();
    render(<ItemList items={mockItems} onDeleteItem={onDelete} />);
    
    const deleteButtons = screen.getAllByText('Delete');
    fireEvent.click(deleteButtons[0]);
    
    expect(onDelete).toHaveBeenCalledWith('1');
  });
  
  it('filters items based on search input', () => {
    const onDelete = jest.fn();
    render(<ItemList items={mockItems} onDeleteItem={onDelete} />);
    
    const searchInput = screen.getByPlaceholderText('Filter...');
    fireEvent.change(searchInput, { target: { value: 'Item 1' } });
    
    expect(screen.getByText('Item 1')).toBeInTheDocument();
    expect(screen.queryByText('Item 2')).not.toBeInTheDocument();
  });
});
```

#### Testing Domain Logic

```typescript
import { renderHook, act } from '@testing-library/react';
import { useItemManager } from './useItemManager';
import * as dataLayer from '../data/items';

jest.mock('../data/items');

describe('useItemManager', () => {
  it('loads items on mount', async () => {
    const mockItems = [{ id: '1', name: 'Item 1' }];
    (dataLayer.fetchItems as jest.Mock).mockResolvedValue(mockItems);
    
    const { result, waitForNextUpdate } = renderHook(() => useItemManager());
    
    expect(result.current.isLoading).toBe(true);
    
    await waitForNextUpdate();
    
    expect(result.current.items).toEqual(mockItems);
    expect(result.current.isLoading).toBe(false);
  });
  
  it('handles delete action', async () => {
    (dataLayer.deleteItem as jest.Mock).mockResolvedValue(undefined);
    
    const { result } = renderHook(() => useItemManager());
    
    await act(async () => {
      await result.current.onDeleteItem('1');
    });
    
    expect(dataLayer.deleteItem).toHaveBeenCalledWith('1');
  });
});
```

## Before/After Examples

### Complete Example: Task Manager Component

#### Before: Mixed Concerns

```typescript
// TaskManager.tsx - Tightly coupled to application
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

#### After: Clean Presentation Component

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

#### After: Domain Container

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

#### After: Domain Hook

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

#### After: Data Layer

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

### Key Improvements After Hoisting

1. **Presentation Component (`TaskManager`)**:
   - Can be rendered in Storybook with mock props
   - Can be tested without mocking auth, API, or navigation
   - Can be used in any application that provides the props
   - All business logic removed, pure UI rendering

2. **Domain Container (`TaskManagerContainer`)**:
   - Owns all application-specific logic
   - Orchestrates data fetching and state management
   - Handles loading states and errors
   - Can be easily modified without touching UI

3. **Domain Hook (`useTaskManager`)**:
   - Reusable across multiple components
   - Testable in isolation
   - Encapsulates business logic
   - Clean separation of concerns

4. **Data Layer (`tasks.ts`)**:
   - Single source of truth for API calls
   - Transforms DTOs to domain models
   - Can be swapped for mock implementation
   - No UI or business logic

This separation enables each layer to evolve independently and be reused in different contexts, achieving true portability.

