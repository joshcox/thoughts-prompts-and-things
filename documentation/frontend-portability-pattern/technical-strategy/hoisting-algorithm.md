---
status: draft
priority: critical
type: strategy
tags: [frontend-portability, hoisting, architecture, strategy, patterns]
---

# Hoisting Algorithm: Technical Strategy

## Overview

**Hoisting** is the process of extracting application-specific concerns from UI components and moving them to a higher layer. This transforms tightly-coupled components into reusable, portable components that can work in multiple applications.

The result: components that can be rendered in Storybook, tested in isolation, embedded in different applications, and composed into new workflows—all without modification.

## When to Use Hoisting

Apply hoisting when you need:

- **Portability**: Components that work across multiple applications
- **Testability**: Components that are easy to test without complex mocking
- **Reusability**: A library of components for building new interfaces
- **Flexibility**: The ability to embed your application in different contexts
- **Documentation**: Components that can be showcased in Storybook

Skip hoisting for one-off, application-specific components that won't be reused.

## How It Works

Hoisting separates components into two pieces:

1. **Presentation Component** - Pure UI that receives data and callbacks via props
2. **Domain Container** - Manages state, data fetching, and business logic

This creates a clean boundary: presentation renders what it's told, domain tells it what to render.

## The Hoisting Process

### Step 1: Identify External Dependencies

Look for anything in your component that reaches outside its boundaries:

- **State** that comes from Context, Redux, or other global sources
- **Data** fetched from APIs or network resources  
- **Handlers** that affect external state or trigger network requests
- **Services** like routing (`useNavigate`), authentication (`useAuth`), or analytics

**Keep these inside the component:**
- Local UI state (expanded, focused, hovered)
- UI-only effects (scroll, focus, animation)
- Derived values calculated from props

### Step 2: Move Dependencies to Props

Refactor the component to receive external dependencies as props:

```typescript
// Before: Component reaches out for dependencies
function TaskList() {
  const tasks = useQuery(GET_TASKS);
  const dispatch = useDispatch();
  const navigate = useNavigate();
  
  const handleDelete = (id: string) => {
    dispatch(deleteTask(id));
  };
  
  const handleEdit = (id: string) => {
    navigate(`/tasks/${id}/edit`);
  };
  
  return <div>{/* render tasks */}</div>;
}

// After: Component receives dependencies as props
function TaskList({ 
  tasks, 
  onDelete, 
  onEdit 
}: TaskListProps) {
  return <div>{/* render tasks */}</div>;
}
```

### Step 3: Create a Domain Container

Build a container that provides the hoisted dependencies:

```typescript
// Domain container manages concerns
function TaskListContainer() {
  const tasks = useQuery(GET_TASKS);
  const dispatch = useDispatch();
  const navigate = useNavigate();
  
  const handleDelete = useCallback((id: string) => {
    dispatch(deleteTask(id));
  }, [dispatch]);
  
  const handleEdit = useCallback((id: string) => {
    navigate(`/tasks/${id}/edit`);
  }, [navigate]);
  
  return (
    <TaskList 
      tasks={tasks.data || []}
      onDelete={handleDelete}
      onEdit={handleEdit}
    />
  );
}
```

## The Result

After hoisting, you have:

**A portable presentation component** that:
- Works in any application with compatible props
- Renders in Storybook with mock data
- Tests easily without mocking infrastructure
- Composes into new workflows

**A domain container** that:
- Handles application-specific concerns
- Can be customized per application
- Encapsulates business logic
- Manages state and data

## Design Guidelines

### Props Interface

Keep props simple and stable:

```typescript
// Good: Clear, typed, stable
interface Props {
  items: Item[];
  onSelect: (id: string) => void;
  isLoading: boolean;
}

// Bad: Leaky, coupled
interface Props {
  reduxStore: Store;        // Exposes Redux
  formik: FormikBag;        // Exposes Formik
  onSelect: (item: any) => any;  // Weak types
}
```

### State Placement

- **Component state**: UI interactions (expanded, focused)
- **Domain state**: Application data (user info, fetched resources)

### Callback Design

- Pass simple callback functions
- Use `useCallback` for stable references
- Keep callbacks focused on single responsibilities

## Common Patterns

### Parallel Composition
```typescript
function Dashboard() {
  const tasks = useTaskManager();
  const projects = useProjectManager();
  
  return (
    <>
      <TaskList {...tasks} />
      <ProjectList {...projects} />
    </>
  );
}
```

### Nested Composition
```typescript
function ProjectDetail({ projectId }: Props) {
  const project = useProject(projectId);
  const tasks = useProjectTasks(projectId);
  
  return (
    <ProjectCard {...project}>
      <TaskList {...tasks} />
    </ProjectCard>
  );
}
```

### Conditional Rendering
```typescript
function AdaptiveView() {
  const data = useData();
  
  if (data.type === 'list') return <ListView {...data} />;
  if (data.type === 'grid') return <GridView {...data} />;
  return <TableView {...data} />;
}
```

## Common Pitfalls

### Over-Hoisting
Don't hoist purely local UI state:

```typescript
// Bad: Hoisting UI-only state
function Container() {
  const [expanded, setExpanded] = useState(false);
  return <Component expanded={expanded} onToggle={setExpanded} />;
}

// Good: Keep UI state local
function Component() {
  const [expanded, setExpanded] = useState(false);
  return <div>...</div>;
}
```

### Under-Hoisting
Don't leave external dependencies in presentation:

```typescript
// Bad: External dependency in presentation
function Component({ items }: Props) {
  const navigate = useNavigate(); // Should be hoisted!
  return <div onClick={() => navigate('/items')}>...</div>;
}

// Good: Navigation hoisted
function Component({ items, onNavigate }: Props) {
  return <div onClick={onNavigate}>...</div>;
}
```

## Summary

Hoisting is a simple but powerful technique:

1. **Identify** external dependencies in your component
2. **Move** them to props in the component interface
3. **Create** a container that provides those props

This creates portable, testable, reusable components that work across different applications and contexts.

**Related documentation:**
- [Guide: Hoisting Refactoring](../guides/hoisting-refactoring-guide.md) - Detailed refactoring examples
- [Architecture: Three-Layer Pattern](../architecture.md) - Overall architecture context
