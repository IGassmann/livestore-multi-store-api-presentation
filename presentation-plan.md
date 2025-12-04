# Multi-Store API Presentation Plan (5-10 minutes)

## Part 1: Current API & Its Limitations (1-2 min)

### Show the Current API
```tsx
function App() {
  return (
    <LiveStoreProvider
      schema={appSchema}
      adapter={adapter}
      renderLoading={() => <div>Loading...</div>}
      renderError={(error) => <div>Error: {error.message}</div>}
    >
      <MainContent />
    </LiveStoreProvider>
  )
}

function MainContent() {
  const store = useStore() // Always returns the same global store
  const data = store.useQuery(myQuery)
  return <div>{data}</div>
}
```

### Issues with Current API

1. **One store per app** - Can't have multiple independent stores
2. **No dynamic store IDs** - Store identity fixed at provider level
3. **No lifecycle management** - Can't load/unload stores dynamically
4. **Render props** - Not using modern Suspense/Error Boundaries

### Why This Matters

"Imagine an issue tracking app with many thousands of threads - you can't load them all at once due to memory constraints. You want to load/unload them dynamically based on user interaction."

- **Partial sync** - browsers have memory limits; can't load 100k issues at once
- **Multi-workspace apps** (Slack, Notion, Linear) - each workspace needs isolated data

---

## Part 2: The New Multi-Store API (2-3 min)

### API Overview

| API                       | Purpose                                          |
|:--------------------------|:-------------------------------------------------|
| `storeOptions()`          | Define reusable, type-safe store configuration   |
| `new StoreRegistry()`     | Manages all store instances (caching, GC)        |
| `<StoreRegistryProvider>` | Provides registry to React tree                  |
| `useStore(options)`       | Get store instance (suspends until ready)        |
| `useStoreRegistry()`      | Access registry for preloading                   |

### Step 1: Single Store (Simplest Case)

"For simple apps with one store, the new API is just as straightforward:"

```tsx
// Define store options (once, alongside your schema)
const workspaceStoreOptions = storeOptions({
  storeId: 'workspace-root',
  schema,
  adapter,
})

// Set up the registry at app root
function App() {
  const [registry] = useState(() => new StoreRegistry())
  return (
    <StoreRegistryProvider storeRegistry={registry}>
      <Suspense fallback={<Loading />}>
        <MainContent />
      </Suspense>
    </StoreRegistryProvider>
  )
}

// Use the store in components
function MainContent() {
  const store = useStore(workspaceStoreOptions)  // Suspends until ready
  const data = store.useQuery(myQuery)
  return <div>{data}</div>
}
```

### Step 2: Multiple Stores

"Now let's add a second store - maybe issues alongside your workspace:"

```tsx
// Two different store types
const workspaceStoreOptions = storeOptions({
  storeId: 'workspace-root',
  schema: workspaceSchema,
  adapter: workspaceAdapter,
})

const issueStoreOptions = (issueId: string) =>
  storeOptions({
    storeId: `issue-${issueId}`,
    schema: issueSchema,
    adapter: issueAdapter,
    unusedCacheTime: 5_000,  // Garbage collect after 5s unused
  })

function IssueCard({ issueId }) {
  const store = useStore(issueStoreOptions(issueId))
  // Each issue gets its own isolated store, loaded on demand
}

// Multiple instances at once
function IssueList({ issueIds }) {
  return (
    // Error and loading states can be placed where needed
    <ErrorBoundary fallback={<IssueListError />}>
      <Suspense fallback={<IssueListSkeleton />}>
        {issueIds.map((id) => (
          <IssueCard key={id} issueId={id} />
        ))}
      </Suspense>
    </ErrorBoundary>
  )
}
```

**Key points:**
- Each store instance is fully isolated
- Automatic caching - same `storeId` returns same instance
- Automatic memory eviction - unused stores disposed after `unusedCacheTime`
- Suspense and error boundary integration - control loading and error state placement

---

## Part 3: web-multi-store Demo (2-3 min)

### 3A. Quick App Demo (~30 sec)
1. Run `examples/web-multi-store`
2. Click through different routes to show stores loading
3. Hover over items to show preloading (instant navigation)
4. Navigate away and back - show cache hit

### 3B. Code Walkthrough (~2 min)

**File 1: Singleton store options**
`examples/web-multi-store/src/stores/workspace/index.ts`
- Show `storeOptions()` pattern
- Point out `storeId: 'workspace-root'` (static)
- Point out `unusedCacheTime: Infinity` (never dispose)

**File 2: Multi-instance store options**
`examples/web-multi-store/src/stores/issue/index.ts`
- Show factory function `issueStoreOptions(issueId)`
- Point out dynamic `storeId: \`issue-${issueId}\``
- Point out `unusedCacheTime: 20_000` (dispose after 20s)

**File 3: Component usage**
`examples/web-multi-store/src/components/WorkspaceView.tsx`
- Show `useStore(workspaceStoreOptions)`
- Show `useStoreRegistry()` + `storeRegistry.preload()` on hover
- "Preloading warms the cache so navigation is instant"

**File 4: Dynamic store loading**
`examples/web-multi-store/src/components/IssueView.tsx`
- Show `useStore(issueStoreOptions(issueId))`
- "Each issue gets its own isolated store, loaded on demand"

**Key points to emphasize:**
- `storeOptions()` - reusable, type-safe config
- `StoreRegistry` - manages caching, ref-counting, garbage collection
- `useStore(options)` - suspends until store is ready
- Same API works for singleton or multi-instance

---

## Part 4: web-email-client Demo (2-3 min)

**Transition:** "Now let's see how this unlocks partial synchronization..."

### 4A. Explain the Architecture (~1 min)
Show the README diagram or explain verbally:
- **Mailbox store** (singleton, always loaded): labels, thread index, UI state
- **Thread stores** (multi-instance, lazy-loaded): individual thread data
- "You can't load 100k threads - so we partition and load on demand"

### 4B. Code Walkthrough (~1-2 min)

**File 1: Projection tables**
`examples/web-email-client/src/stores/mailbox/schema.ts`
- Show `threadIndex` table - "This is a projection, mirrored from Thread stores"
- Show `labels` table with `threadCount` - "Cached count so we don't need all stores"

**File 2: Thread store options**
`examples/web-email-client/src/stores/thread/index.ts`
- Show dynamic `storeId: \`thread-${threadId}\``
- "Each thread is its own store, garbage collected when inactive"

**File 3: Preloading in list**
`examples/web-email-client/src/components/ThreadList.tsx`
- Show `storeRegistry.preload(threadStoreOptions(threadId))` on hover
- "User hovers, we start loading, by the time they click it's ready"

**Optional - Cross-store sync (if time/interest):**
- Briefly mention `ThreadClientDO.ts` publishes changes to queue
- `MailboxClientDO.ts` consumes and updates projections
- "Cross-store sync is manual for now - no built-in primitives yet"

---

## Part 5: Announcement - This Will Replace the Current API (1 min)

**Key announcement:**

"This multi-store API will very soon become the main React API for LiveStore."

**Timeline & Migration:**
- We'll provide a migration guide when we make the switch
- Migration is straightforward - mainly moving config into `storeOptions()`

**Call to action:**
- The API is available now in [`@livestore/react` (experimental)](https://dev.docs.livestore.dev/reference/framework-integrations/react-integration/#multi-store)
- We want to make sure it works for your use cases. Feedback is always appreciated!

---

## Demo Preparation Checklist

Before the call:
- [ ] Run `examples/web-multi-store` locally and verify it works
- [ ] Run `examples/web-email-client` locally and verify it works
- [ ] Open these files in your editor (tabs ready to switch):
  - `examples/web-multi-store/src/stores/workspace/index.ts`
  - `examples/web-multi-store/src/stores/issue/index.ts`
  - `examples/web-multi-store/src/components/WorkspaceView.tsx`
  - `examples/web-multi-store/src/components/IssueView.tsx`
  - `examples/web-email-client/src/stores/mailbox/schema.ts`
  - `examples/web-email-client/src/stores/thread/index.ts`
  - `examples/web-email-client/src/components/ThreadList.tsx`
- [ ] Have `examples/web-email-client/README.md` open for the architecture diagram

---
