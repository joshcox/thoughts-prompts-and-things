## The Pattern: Server-Side Prefetching with TanStack Query

You can prefetch on the server and hydrate TanStack Query. Here's how:

### 1. Create a Server-Side Query Client Utility

Create a new file for server-side query operations:

```typescript
// src/lib/query/server.ts
import { QueryClient } from '@tanstack/react-query';
import { getNotes } from '../actions/note-actions';
import { NOTE_QUERY_KEY } from './use-notes';

export function getServerQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,
        gcTime: 10 * 60 * 1000,
      },
    },
  });
}

export async function prefetchNotes() {
  const queryClient = getServerQueryClient();
  
  await queryClient.prefetchQuery({
    queryKey: NOTE_QUERY_KEY,
    queryFn: async () => {
      const result = await getNotes();
      if (!result.success) {
        throw new Error(result.error);
      }
      return result.data;
    },
  });
  
  return queryClient;
}
```

### 2. Update Your QueryProvider to Accept Prefetched Data

```typescript
// src/lib/query/client.tsx
'use client';

import { QueryClient, QueryClientProvider, HydrationBoundary, dehydrate } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { useState } from 'react';

export function QueryProvider({ 
  children,
  dehydratedState 
}: { 
  children: React.ReactNode;
  dehydratedState?: any;
}) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000,
            gcTime: 10 * 60 * 1000,
          },
        },
      })
  );

  return (
    <QueryClientProvider client={queryClient}>
      {dehydratedState ? (
        <HydrationBoundary state={dehydratedState}>
          {children}
        </HydrationBoundary>
      ) : (
        children
      )}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

### 3. Prefetch in a Server Component (Layout or Page)

**Option A: Prefetch in Layout (for all notes pages)**

```typescript
// src/app/notes/layout.tsx (NEW FILE - Server Component)
import { QueryProvider } from '@/lib/query/client';
import { prefetchNotes } from '@/lib/query/server';
import { dehydrate } from '@tanstack/react-query';

export default async function NotesLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const queryClient = await prefetchNotes();
  const dehydratedState = dehydrate(queryClient);

  return (
    <QueryProvider dehydratedState={dehydratedState}>
      {children}
    </QueryProvider>
  );
}
```

**Option B: Prefetch in Page (for specific page)**

```typescript
// src/app/notes/page.tsx (convert to Server Component wrapper)
import { dehydrate } from '@tanstack/react-query';
import { prefetchNotes } from '@/lib/query/server';
import { NotesPageClient } from './notes-page-client'; // Your current component

export default async function NotesPage() {
  const queryClient = await prefetchNotes();
  const dehydratedState = dehydrate(queryClient);

  return (
    <HydrationBoundary state={dehydratedState}>
      <NotesPageClient />
    </HydrationBoundary>
  );
}
```

### 4. Export Query Keys for Server-Side Use

```typescript
// src/lib/query/use-notes.ts
export const NOTE_QUERY_KEY = ['notes'];
export const NOTE_QUERY_KEY_BY_ID = (id: number) => ['notes', id];

// ... rest of your hooks
```

## How It Works:

1. Server Component prefetches data using `prefetchQuery`
2. `dehydrate()` serializes the query cache
3. `HydrationBoundary` passes it to the client
4. Client components use the same query keys and get the prefetched data
5. No loading state — data is already there
6. Still reactive — mutations still work and update the cache

## Benefits:

- Faster initial load — data is in HTML
- No loading spinner — data is prefetched
- Same query keys — works with existing hooks
- Automatic hydration — TanStack Query handles it
- Still reactive — mutations and refetches work normally

## Example: Prefetching a Single Note

```typescript
// src/lib/query/server.ts
export async function prefetchNote(id: number) {
  const queryClient = getServerQueryClient();
  
  await queryClient.prefetchQuery({
    queryKey: ['notes', id],
    queryFn: async () => {
      const result = await getNote(id);
      if (!result.success) {
        throw new Error(result.error);
      }
      return result.data;
    },
  });
  
  return queryClient;
}

// In your note detail page layout:
export default async function NoteLayout({ params }: { params: { id: string } }) {
  const id = parseInt(params.id, 10);
  const queryClient = await prefetchNote(id);
  const dehydratedState = dehydrate(queryClient);
  
  return (
    <HydrationBoundary state={dehydratedState}>
      {children}
    </HydrationBoundary>
  );
}
```

This prefetches on the server and hydrates into your existing TanStack Query cache. Your client components don't need to change — they'll automatically use the prefetched data.

Should I show you how to implement this for your specific pages?