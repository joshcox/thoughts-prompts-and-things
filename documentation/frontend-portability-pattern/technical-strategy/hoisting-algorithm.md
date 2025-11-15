# Hoisting Algorithm: Technical Strategy

- [Hoisting Algorithm: Technical Strategy](#hoisting-algorithm-technical-strategy)
  - [Overview](#overview)
  - [What is Hoisting?](#what-is-hoisting)
    - [The Technical Transformation](#the-technical-transformation)
  - [Why Hoisting Creates Portability](#why-hoisting-creates-portability)
    - [Breaking Environmental Bindings](#breaking-environmental-bindings)
    - [Creating Substitutability](#creating-substitutability)
    - [Enabling Composition](#enabling-composition)
  - [Relationship to Layer Separation](#relationship-to-layer-separation)
    - [Presentation Layer](#presentation-layer)
    - [Domain Layer](#domain-layer)
    - [Data Layer](#data-layer)
    - [The Boundary](#the-boundary)
  - [Execution Principles](#execution-principles)
    - [Principle 1: Dependency Identification](#principle-1-dependency-identification)
    - [Principle 2: Interface Design](#principle-2-interface-design)
    - [Principle 3: Concern Relocation](#principle-3-concern-relocation)
  - [Technical Considerations](#technical-considerations)
    - [State Management](#state-management)
    - [Side Effect Management](#side-effect-management)
    - [Callback Design](#callback-design)
    - [Data Flow Architecture](#data-flow-architecture)
    - [Testing Strategy](#testing-strategy)
  - [Advanced Patterns](#advanced-patterns)
    - [Graduated Hoisting](#graduated-hoisting)
    - [Composition Patterns](#composition-patterns)
    - [Props Forwarding](#props-forwarding)
  - [Performance Implications](#performance-implications)
    - [Re-render Optimization](#re-render-optimization)
    - [Data Fetching Patterns](#data-fetching-patterns)
  - [Common Pitfalls](#common-pitfalls)
    - [Over-Hoisting](#over-hoisting)
    - [Under-Hoisting](#under-hoisting)
    - [Interface Leakage](#interface-leakage)
  - [Summary](#summary)


## Overview

The **hoisting algorithm** is the core technical strategy that enables the three-layer architecture and achieves component portability. This document explains the execution details, underlying principles, and technical considerations that make hoisting work.

For step-by-step refactoring instructions, see [Guide: Hoisting Refactoring](../guides/hoisting-refactoring-guide.md).

## What is Hoisting?

**Hoisting** is the systematic process of extracting application-specific concerns from UI components and relocating them to a higher-level orchestration layer (the domain layer). This transforms tightly-coupled components into reusable, context-free presentation components.

The term derives from "lifting" or "raising" dependencies out of a component - similar to JavaScript's hoisting mechanism where declarations move to the top of their scope. Here, we're moving concerns up the component hierarchy.

### The Technical Transformation

Hoisting performs a **concern extraction** that produces:

1. **Context-Free Presentation Component**
   - Receives all external dependencies via props
   - Contains only local UI state
   - Pure function of props → UI
   - No side effects that reach beyond component boundaries

2. **Domain Container**
   - Owns extracted concerns (state, data, side effects)
   - Orchestrates calls to data layer
   - Transforms domain data into presentation props
   - Manages business logic and workflows

This separation is achieved through **dependency inversion**: instead of the component reaching out for dependencies, dependencies are injected into the component.

## Why Hoisting Creates Portability

### Breaking Environmental Bindings

Components without hoisting are **environmentally coupled** to specific:

- **Authentication systems**: `useAuth()` assumes a specific auth provider
- **Routing libraries**: `useNavigate()` binds to React Router
- **State management**: `useSelector()` requires Redux
- **API clients**: Direct `fetch()` calls bind to specific endpoints
- **Service integrations**: Analytics, logging, error tracking

Each of these creates a **hard dependency** on environmental infrastructure.

### Creating Substitutability

Hoisting breaks these hard dependencies by introducing **indirection**:

```typescript
// Hard dependency (environmentally coupled)
function Component() {
  const data = useQuery(GET_DATA); // Bound to GraphQL client
  const user = useAuth(); // Bound to auth system
  return <UI />;
}

// Indirection (substitutable)
function Component({ data, user }: Props) {
  return <UI />;
}
```

The component now depends only on the **shape of props**, not their source. Any system that can provide props matching the interface can render this component.

### Enabling Composition

Hoisting enables **compositional flexibility** because:

1. **Multiple consumption contexts**: The same component works in app A, app B, and Storybook
2. **Different data sources**: Props can come from GraphQL, REST, mock data, or localStorage
3. **Varied orchestration**: Different containers can compose the same component differently
4. **Independent evolution**: Presentation and domain can change independently

## Relationship to Layer Separation

Hoisting is the **practical technique** for enforcing architectural layer boundaries:

### Presentation Layer

After hoisting, presentation contains:
- Components that receive props (data + callbacks)
- Local UI state (collapsed, selected, focused)
- Derived values calculated from props
- UI-only side effects (scroll, animation)

**No:**
- Direct data fetching
- Global state access
- Router manipulation
- External service calls

### Domain Layer

Hoisting relocates concerns to domain:
- Business logic and workflows
- State management and orchestration
- Data fetching coordination
- Side effect execution

**Interface:** Provides props to presentation, calls data layer

### Data Layer

Unaffected by hoisting (already isolated):
- API clients and data access
- DTO transformations
- Network request handling

**Interface:** Provides functions to domain

### The Boundary

Hoisting operates at the **Domain ← → Presentation boundary**, establishing a clean contract:

```
Data Layer
    ↓ (function calls)
Domain Layer
    ↓ (props: data + callbacks)
Presentation Layer
```

## Execution Principles

### Principle 1: Dependency Identification

**Objective:** Distinguish between **local concerns** (keep) and **external concerns** (hoist).

**Technical criteria for external concerns:**

- **Scope violation**: Reaches beyond component's lexical scope
- **Side effect propagation**: Affects state outside component tree
- **Environmental coupling**: Depends on context/provider higher in tree
- **Network dependency**: Performs or triggers I/O operations

**Decision framework:**

```
Concern has effect outside component?
├─ No → Local concern (keep internal)
└─ Yes → External concern (hoist)
    ├─ Affects parent state? → Hoist
    ├─ Triggers network? → Hoist
    ├─ Uses context/global state? → Hoist
    └─ Navigates router? → Hoist
```

### Principle 2: Interface Design

**Objective:** Create a **minimal, stable interface** between layers.

**Technical considerations:**

**Props should be:**
- **Serializable**: Can be logged, stored, transmitted
- **Stateless**: No hidden mutable state
- **Type-safe**: Fully typed with TypeScript
- **Self-contained**: No external type dependencies

**Anti-patterns:**
- Passing class instances with methods
- Leaking internal library types (e.g., `FormikValues`)
- Using `any` or weak types
- Coupling to specific data structures

**Interface stability:**

The props interface is a **contract**. Breaking changes to this contract break all consumers. Design with stability in mind:

```typescript
// Stable: Clear, typed, minimal
interface Props {
  items: Item[];
  onSelect: (id: string) => void;
}

// Unstable: Leaky, coupled, complex
interface Props {
  store: ReduxStore; // Leaks Redux
  formik: FormikBag; // Leaks Formik
  onSelect: (item: any, context: any) => any; // Weak types
}
```

### Principle 3: Concern Relocation

**Objective:** Move external concerns to domain while **preserving behavior**.

**Technical approach:**

1. **Closure over dependencies**: Domain container captures concerns in closures
2. **Callback stabilization**: Use `useCallback` to prevent unnecessary re-renders
3. **State co-location**: Group related state together
4. **Effect isolation**: Each effect should have single responsibility

**Example:**

```typescript
// Presentation: Pure, stable
function ItemList({ items, onDelete }: Props) {
  return <ul>{items.map(item => 
    <Item key={item.id} item={item} onDelete={onDelete} />
  )}</ul>;
}

// Domain: Encapsulates complexity
function useItemList() {
  const items = useSelector(selectItems); // External state
  const dispatch = useDispatch(); // External action
  
  // Stabilized callback (won't cause re-renders)
  const handleDelete = useCallback((id: string) => {
    dispatch(deleteItem(id));
  }, [dispatch]);
  
  return { items, onDelete: handleDelete };
}
```

## Technical Considerations

### State Management

**Question:** What state belongs where?

**Technical criterion: State scope**

- **Component-scoped state**: Local to component (keep)
  - UI interactions: hover, focus, expanded
  - Temporary input: text field values before submission
  - View preferences: selected tab, sort order

- **Application-scoped state**: Beyond component (hoist)
  - User session data
  - Fetched resources
  - Global application state
  - Cross-component coordination

**Implementation:**

```typescript
// Component-scoped: useState in presentation
function Component({ data }: Props) {
  const [expanded, setExpanded] = useState(false); // Local
  return <div>...</div>;
}

// Application-scoped: useState in domain
function useComponent() {
  const [data, setData] = useState([]); // Hoisted
  useEffect(() => {
    fetchData().then(setData);
  }, []);
  return { data };
}
```

### Side Effect Management

**Question:** Where should side effects execute?

**Technical criterion: Effect scope**

- **UI-only effects**: Presentation layer
  - Scroll to element
  - Focus input
  - Trigger animation
  - Update canvas

- **Domain effects**: Domain layer
  - Data fetching
  - Analytics tracking
  - Error reporting
  - State synchronization

**Implementation pattern:**

```typescript
// UI-only effect: stays in presentation
function Component() {
  const ref = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    ref.current?.scrollIntoView(); // UI-only
  }, []);
  
  return <div ref={ref}>...</div>;
}

// Domain effect: lives in domain hook
function useComponent() {
  const [data, setData] = useState([]);
  
  useEffect(() => {
    fetchData().then(data => {
      setData(data);
      analytics.track('data_loaded'); // Domain effect
    });
  }, []);
  
  return { data };
}
```

### Callback Design

**Question:** How should callbacks be structured?

**Technical principles:**

1. **Single responsibility**: Each callback does one thing
2. **Stable identity**: Use `useCallback` for referential stability
3. **Error handling**: Domain handles errors, presentation reports them
4. **Async coordination**: Domain manages loading states

**Pattern:**

```typescript
// Presentation: Invokes callback, displays state
function Component({ onSave, isSaving }: Props) {
  return (
    <button onClick={() => onSave(data)} disabled={isSaving}>
      {isSaving ? 'Saving...' : 'Save'}
    </button>
  );
}

// Domain: Handles complexity
function useComponent() {
  const [saving, setSaving] = useState(false);
  
  const handleSave = useCallback(async (data: Data) => {
    setSaving(true);
    try {
      await saveData(data); // Data layer
      toast.success('Saved');
    } catch (err) {
      toast.error('Failed');
      throw err; // Presentation might need to know
    } finally {
      setSaving(false);
    }
  }, []);
  
  return { onSave: handleSave, isSaving: saving };
}
```

### Data Flow Architecture

**Question:** How does data flow through layers?

**Technical model: Unidirectional flow**

```
External APIs/Services
    ↓
Data Layer (fetch, transform)
    ↓
Domain Layer (orchestrate, manage state)
    ↓
Presentation Layer (render)
    ↑ (callbacks)
Domain Layer (handle events)
    ↑ (mutations)
Data Layer (persist changes)
    ↑
External APIs/Services
```

**Key insight:** Data flows down, events flow up.

- **Down:** State and data passed as props
- **Up:** Callbacks invoked to trigger actions

This creates a **predictable data flow** that's easy to trace and debug.

### Testing Strategy

**Question:** How does hoisting affect testing?

**Technical advantage: Independent testability**

**Presentation testing:**
- **No mocking needed**: Just pass props
- **Fast execution**: No async operations
- **Predictable**: Pure function of inputs
- **Visual testing**: Easy to render in Storybook

```typescript
// Simple, no mocks
test('renders items', () => {
  render(<ItemList items={mockItems} onDelete={jest.fn()} />);
  expect(screen.getByText('Item 1')).toBeInTheDocument();
});
```

**Domain testing:**
- **Mock data layer**: Stub API calls
- **Test logic**: Verify state management
- **Test effects**: Check side effects execute
- **Test error handling**: Verify error paths

```typescript
// Mock only data layer
jest.mock('../data/items');

test('loads items', async () => {
  (fetchItems as jest.Mock).mockResolvedValue(mockItems);
  const { result } = renderHook(() => useItemManager());
  await waitFor(() => expect(result.current.items).toEqual(mockItems));
});
```

## Advanced Patterns

### Graduated Hoisting

Not all components need full hoisting. Apply hoisting **proportional to portability needs**:

**Level 1: No hoisting** (component-specific, one-off)
- Acceptable for app-specific pages
- Tightly coupled to environment
- Fast to build, not reusable

**Level 2: Partial hoisting** (some portability)
- Extract data fetching
- Keep some context usage
- Moderate reusability

**Level 3: Full hoisting** (maximum portability)
- All external dependencies hoisted
- Complete context-freedom
- Maximum reusability, more upfront work

**Choose based on:** How likely is reuse? How stable is the API?

### Composition Patterns

Hoisted components enable powerful composition:

**Pattern 1: Parallel composition**
```typescript
function Dashboard() {
  const tasks = useTaskManager();
  const projects = useProjectManager();
  const users = useUserManager();
  
  return (
    <>
      <TaskList {...tasks} />
      <ProjectList {...projects} />
      <UserList {...users} />
    </>
  );
}
```

**Pattern 2: Nested composition**
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

**Pattern 3: Conditional composition**
```typescript
function AdaptiveView() {
  const data = useData();
  
  if (data.type === 'list') return <ListView {...data} />;
  if (data.type === 'grid') return <GridView {...data} />;
  return <TableView {...data} />;
}
```

### Props Forwarding

When composing multiple layers, props may need to flow through intermediaries:

**Anti-pattern: Prop drilling**
```typescript
// Don't pass through many levels
<A data={data}>
  <B data={data}>
    <C data={data}>
      <D data={data} /> {/* Finally used here */}
    </C>
  </B>
</A>
```

**Better: Composition**
```typescript
// Domain provides directly to consumer
function Container() {
  const data = useData();
  return (
    <Layout>
      <ConsumerComponent data={data} />
    </Layout>
  );
}
```

**Or: Context for deep trees**
```typescript
// For truly shared data across deep trees
const DataContext = createContext<Data | null>(null);

function Provider() {
  const data = useData();
  return (
    <DataContext.Provider value={data}>
      <DeepTree />
    </DataContext.Provider>
  );
}
```

## Performance Implications

### Re-render Optimization

Hoisting affects re-render behavior:

**Challenge:** Props changes trigger re-renders

```typescript
// Every container re-render creates new callback reference
function Container() {
  const handleClick = (id: string) => { /* ... */ }; // New every render!
  return <Component onClick={handleClick} />;
}
```

**Solution:** Stabilize callbacks

```typescript
function Container() {
  const handleClick = useCallback((id: string) => {
    /* ... */
  }, []); // Stable reference
  
  return <Component onClick={handleClick} />;
}
```

**Optimization:** Memo presentation components

```typescript
export const Component = memo(function Component({ items, onClick }: Props) {
  return <div>...</div>;
}); // Only re-renders when props actually change
```

### Data Fetching Patterns

Hoisting centralizes data fetching in domain layer, enabling:

**Pattern 1: Parallel fetching**
```typescript
function useDashboard() {
  const [tasks] = useAsync(() => fetchTasks());
  const [projects] = useAsync(() => fetchProjects());
  const [users] = useAsync(() => fetchUsers());
  
  return { tasks, projects, users }; // All fetched in parallel
}
```

**Pattern 2: Dependent fetching**
```typescript
function useProjectDetail(projectId: string) {
  const [project] = useAsync(() => fetchProject(projectId));
  const [tasks] = useAsync(
    () => project ? fetchProjectTasks(project.id) : null,
    [project]
  ); // Waits for project
  
  return { project, tasks };
}
```

**Pattern 3: Cached fetching**
```typescript
function useData(id: string) {
  const cache = useCache();
  const [data, setData] = useState(() => cache.get(id));
  
  useEffect(() => {
    if (!data) {
      fetchData(id).then(d => {
        setData(d);
        cache.set(id, d);
      });
    }
  }, [id, data, cache]);
  
  return data;
}
```

## Common Pitfalls

### Over-Hoisting

**Problem:** Hoisting UI-local concerns unnecessarily

```typescript
// Anti-pattern: Hoisting purely local state
function Container() {
  const [expanded, setExpanded] = useState(false); // UI-only!
  return <Component expanded={expanded} onToggle={setExpanded} />;
}
```

**Solution:** Keep UI state local

```typescript
function Component() {
  const [expanded, setExpanded] = useState(false); // Belongs here
  return <div>...</div>;
}
```

### Under-Hoisting

**Problem:** Leaving external dependencies in presentation

```typescript
// Anti-pattern: Leaking context into presentation
function Component({ items }: Props) {
  const navigate = useNavigate(); // External dependency!
  return <div onClick={() => navigate('/items')}>...</div>;
}
```

**Solution:** Hoist navigation concern

```typescript
function Component({ items, onNavigate }: Props) {
  return <div onClick={onNavigate}>...</div>;
}
```

### Interface Leakage

**Problem:** Exposing internal types in props

```typescript
// Anti-pattern: Leaking Formik types
interface Props {
  formik: FormikBag<Values>; // Couples to Formik!
}
```

**Solution:** Use domain types

```typescript
interface Props {
  values: FormValues;
  errors: FormErrors;
  onSubmit: (values: FormValues) => void;
}
```

## Summary

The hoisting algorithm is a **systematic technique** for achieving component portability through **concern extraction** and **dependency inversion**.

**Core principles:**
1. **Identify**: Distinguish local from external concerns
2. **Interface**: Design stable props contracts
3. **Relocate**: Move concerns to domain layer

**Technical benefits:**
- **Portability**: Components work in any compatible environment
- **Testability**: Each layer tested independently
- **Composability**: Components combine in flexible ways
- **Maintainability**: Clear boundaries, stable interfaces

**When to apply:**
- Building shared component libraries
- Creating portable feature modules
- Maximizing test coverage
- Enabling Storybook documentation
- Supporting multiple consumption contexts

**Related documentation:**
- [Guide: Hoisting Refactoring](../guides/hoisting-refactoring-guide.md) - Step-by-step tutorial
- [Architecture: Three-Layer Pattern](../architecture.md#core-pattern-three-layer-architecture) - Overall architecture
- [Architecture: Compositional UI](../architecture.md#compositional-ui-techniques) - Composition patterns
