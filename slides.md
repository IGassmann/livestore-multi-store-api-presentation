---
theme: default
title: Multi-Store API
info: |
  LiveStore Multi-Store API - Managing multiple stores in React applications
class: text-center
drawings:
  persist: false
transition: slide-left
mdc: true
---

# Multi-Store API

A new React API for LiveStore

<!--
Today I want to present the new Multi-Store API for LiveStore's React integration.
-->

---

# Current API

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

function Content() {
  const store = useStore() // Always returns the same global store
  const data = store.useQuery(myQuery)
  return <div>{data}</div>
}
```

<v-clicks>

1. **One store per app** - Can't have multiple independent stores
2. **No lifecycle management** - Can't load/unload stores dynamically
3. **Render props** - Not using modern Suspense/Error Boundaries

</v-clicks>

---

# Why This Matters

> "Imagine an issue tracking app with many thousands of threads - you can't load them all at once due to memory constraints. You want to load/unload them dynamically based on user interaction"

<v-clicks>

- **Partial sync** - browsers have memory limits; can't load 100k issues at once

- **Multi-workspace apps** (Slack, Notion, Linear) - each workspace needs isolated data

</v-clicks>

---

# New API Overview

| API | Purpose |
|:--|:--|
| `storeOptions()` | Define reusable, type-safe store configuration |
| `new StoreRegistry()` | Manages all store instances (caching, GC) |
| `<StoreRegistryProvider>` | Provides registry to React tree |
| `useStore(options)` | Get store instance (suspends until ready) |
| `useStoreRegistry()` | Access registry for preloading |

---

# Single Store

```tsx {all|1-2|4-14|16-24}
// Define store options (once, alongside your schema)
const workspaceStoreOptions = storeOptions({ storeId: 'workspace-root', schema, adapter })

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
  const store = useStore(workspaceStoreOptions) // Suspends until ready
  const data = store.useQuery(myQuery)
  return <div>{data}</div>
}
```

<!--
For simple apps with one store, the new API is just as straightforward.

1. Define store options alongside your schema
2. Set up the registry at app root with Suspense
3. Use the store in components - it suspends until ready
-->

---

# Multiple Stores

```tsx {all|3-8|7|10-12|14-24}
const workspaceStoreOptions = storeOptions({ storeId: 'workspace-root', schema, adapter })

const issueStoreOptions = (issueId: string) => storeOptions({
  storeId: `issue-${issueId}`,
  schema: issueSchema,
  adapter: issueAdapter,
  unusedCacheTime: 5_000, // Evicted from cache after 5s unused
})

function IssueCard({ issueId }) {
  const store = useStore(issueStoreOptions(issueId))
}

function IssueList({ issueIds }) {
  return (
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

<!--
Now let's add a second store - issues alongside your workspace.

Key things to notice:
- Factory function for dynamic storeId
- unusedCacheTime: store is evicted from cache after 5s unused
- Each IssueCard gets its own isolated store
- Suspense and ErrorBoundary give you control over loading/error states
-->

---

# Key Points

<v-clicks>

- Each store instance is **fully isolated**

- **Automatic caching** - same `storeId` returns same instance

- **Automatic memory eviction** - unused stores disposed after `unusedCacheTime`

- **Suspense and Error Boundary integration** - control loading and error state placement

</v-clicks>

---
layout: center
class: text-center
---

# Demo Time

## web-multi-store

Different patterns for managing multiple stores

<!--
Demo: web-multi-store example

Show:
- Click through different routes to show stores loading
- Hover over items to show preloading (instant navigation)
- Navigate away and back - show cache hit
- Show the store options files
- Show useStore usage in components
-->

---
layout: center
class: text-center
---

<div class="text-xl space-y-6">

The API is already available in [`@livestore/react` (experimental)](https://dev.docs.livestore.dev/reference/framework-integrations/react-integration/#multi-store).

It will replace the current API in the upcoming 0.4 release.

<v-click>

**Feedback is always appreciated!**

</v-click>

</div>
