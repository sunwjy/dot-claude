# React Best Practices

**Based on Vercel Engineering Guidelines**

> Common React rules for using the React core APIs (hooks, memo, Suspense, etc.)

---

## Table of Contents

1. [Eliminating Waterfalls](#1-eliminating-waterfalls) — **CRITICAL**
   - 1.1 [Defer Await Until Needed](#11-defer-await-until-needed)
   - 1.2 [Promise.all() for Independent Operations](#12-promiseall-for-independent-operations)
   - 1.3 [Strategic Suspense Boundaries](#13-strategic-suspense-boundaries)
2. [Bundle Size Optimization](#2-bundle-size-optimization) — **CRITICAL**
   - 2.1 [Avoid Barrel File Imports](#21-avoid-barrel-file-imports)
   - 2.2 [Conditional Module Loading](#22-conditional-module-loading)
   - 2.3 [Preload Based on User Intent](#23-preload-based-on-user-intent)
3. [Client-Side Data Fetching](#3-client-side-data-fetching) — **MEDIUM-HIGH**
   - 3.1 [Deduplicate Global Event Listeners](#31-deduplicate-global-event-listeners)
   - 3.2 [Use Passive Event Listeners for Scrolling Performance](#32-use-passive-event-listeners-for-scrolling-performance)
   - 3.3 [Use React Query for Automatic Deduplication](#33-use-react-query-for-automatic-deduplication)
   - 3.4 [Version and Minimize localStorage Data](#34-version-and-minimize-localstorage-data)
4. [Re-render Optimization](#4-re-render-optimization) — **MEDIUM**
   - 4.1 [Defer State Reads to Usage Point](#41-defer-state-reads-to-usage-point)
   - 4.2 [Extract to Memoized Components](#42-extract-to-memoized-components)
   - 4.3 [Narrow Effect Dependencies](#43-narrow-effect-dependencies)
   - 4.4 [Subscribe to Derived State](#44-subscribe-to-derived-state)
   - 4.5 [Use Functional setState Updates](#45-use-functional-setstate-updates)
   - 4.6 [Use Lazy State Initialization](#46-use-lazy-state-initialization)
   - 4.7 [Use Transitions for Non-Urgent Updates](#47-use-transitions-for-non-urgent-updates)
5. [Rendering Performance](#5-rendering-performance) — **MEDIUM**
   - 5.1 [Animate SVG Wrapper Instead of SVG Element](#51-animate-svg-wrapper-instead-of-svg-element)
   - 5.2 [CSS content-visibility for Long Lists](#52-css-content-visibility-for-long-lists)
   - 5.3 [Hoist Static JSX Elements](#53-hoist-static-jsx-elements)
   - 5.4 [Prevent Hydration Mismatch Without Flickering](#54-prevent-hydration-mismatch-without-flickering)
   - 5.5 [Use Activity Component for Show/Hide](#55-use-activity-component-for-showhide)
   - 5.6 [Use Explicit Conditional Rendering](#56-use-explicit-conditional-rendering)
6. [JavaScript Performance](#6-javascript-performance) — **LOW-MEDIUM**
   - 6.1 [Batch DOM CSS Changes](#61-batch-dom-css-changes)
   - 6.2 [Build Index Maps for Repeated Lookups](#62-build-index-maps-for-repeated-lookups)
   - 6.3 [Use Set/Map for O(1) Lookups](#63-use-setmap-for-o1-lookups)
   - 6.4 [Use toSorted() Instead of sort() for Immutability](#64-use-tosorted-instead-of-sort-for-immutability)
7. [Advanced Patterns](#7-advanced-patterns) — **LOW**
   - 7.1 [Store Event Handlers in Refs](#71-store-event-handlers-in-refs)
   - 7.2 [useLatest for Stable Callback Refs](#72-uselatest-for-stable-callback-refs)

---

## 1. Eliminating Waterfalls

**Impact: CRITICAL**

### 1.1 Defer Await Until Needed

**Impact: HIGH (avoids blocking unused code paths)**

Move `await` operations into the branches where they're actually used.

**Incorrect:**

```typescript
async function handleRequest(userId: string, skipProcessing: boolean) {
  const userData = await fetchUserData(userId)
  
  if (skipProcessing) {
    return { skipped: true }
  }
  
  return processUserData(userData)
}
```

**Correct:**

```typescript
async function handleRequest(userId: string, skipProcessing: boolean) {
  if (skipProcessing) {
    return { skipped: true }
  }
  
  const userData = await fetchUserData(userId)
  return processUserData(userData)
}
```

### 1.2 Promise.all() for Independent Operations

**Impact: CRITICAL (2-10× improvement)**

When async operations have no interdependencies, execute them concurrently.

**Incorrect:**

```typescript
const user = await fetchUser()
const posts = await fetchPosts()
const comments = await fetchComments()
```

**Correct:**

```typescript
const [user, posts, comments] = await Promise.all([
  fetchUser(),
  fetchPosts(),
  fetchComments()
])
```

### 1.3 Strategic Suspense Boundaries

**Impact: HIGH (faster initial paint)**

Use Suspense boundaries to show wrapper UI faster while data loads.

**Incorrect:**

```tsx
async function Page() {
  const data = await fetchData()
  
  return (
    <div>
      <div>Sidebar</div>
      <div>Header</div>
      <div>
        <DataDisplay data={data} />
      </div>
      <div>Footer</div>
    </div>
  )
}
```

**Correct:**

```tsx
function Page() {
  return (
    <div>
      <div>Sidebar</div>
      <div>Header</div>
      <div>
        <Suspense fallback={<Skeleton />}>
          <DataDisplay />
        </Suspense>
      </div>
      <div>Footer</div>
    </div>
  )
}

async function DataDisplay() {
  const data = await fetchData()
  return <div>{data.content}</div>
}
```

**Alternative: share promise across components**

```tsx
function Page() {
  const dataPromise = fetchData()
  
  return (
    <div>
      <div>Sidebar</div>
      <Suspense fallback={<Skeleton />}>
        <DataDisplay dataPromise={dataPromise} />
        <DataSummary dataPromise={dataPromise} />
      </Suspense>
      <div>Footer</div>
    </div>
  )
}

function DataDisplay({ dataPromise }: { dataPromise: Promise<Data> }) {
  const data = use(dataPromise)
  return <div>{data.content}</div>
}
```

---

## 2. Bundle Size Optimization

**Impact: CRITICAL**

### 2.1 Avoid Barrel File Imports

**Impact: CRITICAL (200-800ms import cost, slow builds)**

Import directly from source files instead of barrel files to avoid loading thousands of unused modules. **Barrel files** are entry points that re-export multiple modules (e.g., `index.js` that does `export * from './module'`).

Popular icon and component libraries can have **up to 10,000 re-exports** in their entry file. For many React packages, **it takes 200-800ms just to import them**, affecting both development speed and production cold starts.

**Why tree-shaking doesn't help:** When a library is marked as external (not bundled), the bundler can't optimize it. If you bundle it to enable tree-shaking, builds become substantially slower analyzing the entire module graph.

**Incorrect:**

```tsx
import { Check, X, Menu } from 'lucide-react'
// Loads 1,583 modules, takes ~2.8s extra in dev
// Runtime cost: 200-800ms on every cold start

import { Button, TextField } from '@mui/material'
// Loads 2,225 modules, takes ~4.2s extra in dev
```

**Correct:**

```tsx
import Check from 'lucide-react/dist/esm/icons/check'
import X from 'lucide-react/dist/esm/icons/x'
import Menu from 'lucide-react/dist/esm/icons/menu'
// Loads only 3 modules (~2KB vs ~1MB)

import Button from '@mui/material/Button'
import TextField from '@mui/material/TextField'
// Loads only what you use
```

**Impact:** 15-70% faster dev boot, 28% faster builds, 40% faster cold starts, and significantly faster HMR.

**Commonly affected libraries:** `lucide-react`, `@mui/material`, `@mui/icons-material`, `@tabler/icons-react`, `react-icons`, `@headlessui/react`, `@radix-ui/react-*`, `lodash`, `ramda`, `date-fns`, `rxjs`, `react-use`.

### 2.2 Conditional Module Loading

**Impact: HIGH (loads large data only when needed)**

Load large data or modules only when a feature is activated.

```tsx
function AnimationPlayer({ enabled, setEnabled }: Props) {
  const [frames, setFrames] = useState<Frame[] | null>(null)

  useEffect(() => {
    if (enabled && !frames && typeof window !== 'undefined') {
      import('./animation-frames.js')
        .then(mod => setFrames(mod.frames))
        .catch(() => setEnabled(false))
    }
  }, [enabled, frames, setEnabled])

  if (!frames) return <Skeleton />
  return <Canvas frames={frames} />
}
```

### 2.3 Preload Based on User Intent

**Impact: MEDIUM (reduces perceived latency)**

Preload heavy bundles before they're needed.

```tsx
function EditorButton({ onClick }: { onClick: () => void }) {
  const preload = () => {
    if (typeof window !== 'undefined') {
      void import('./monaco-editor')
    }
  }

  return (
    <button
      onMouseEnter={preload}
      onFocus={preload}
      onClick={onClick}
    >
      Open Editor
    </button>
  )
}
```

---

## 3. Client-Side Data Fetching

**Impact: MEDIUM-HIGH**

### 3.1 Deduplicate Global Event Listeners

**Impact: LOW (single listener for N components)**

Use a shared subscription pattern with `useQuery` to share global event listeners across component instances.

**Incorrect: N instances = N listeners**

```tsx
function useKeyboardShortcut(key: string, callback: () => void) {
  useEffect(() => {
    const handler = (e: KeyboardEvent) => {
      if (e.metaKey && e.key === key) {
        callback()
      }
    }
    window.addEventListener('keydown', handler)
    return () => window.removeEventListener('keydown', handler)
  }, [key, callback])
}
```

**Correct: N instances = 1 listener**

```tsx
import { useQueryClient, useQuery } from '@tanstack/react-query'

const keyCallbacks = new Map<string, Set<() => void>>()
let isListenerAttached = false

function setupGlobalKeyListener() {
  if (isListenerAttached) return
  isListenerAttached = true
  
  const handler = (e: KeyboardEvent) => {
    if (e.metaKey && keyCallbacks.has(e.key)) {
      keyCallbacks.get(e.key)!.forEach(cb => cb())
    }
  }
  window.addEventListener('keydown', handler)
}

function useKeyboardShortcut(key: string, callback: () => void) {
  useEffect(() => {
    if (!keyCallbacks.has(key)) {
      keyCallbacks.set(key, new Set())
    }
    keyCallbacks.get(key)!.add(callback)

    return () => {
      const set = keyCallbacks.get(key)
      if (set) {
        set.delete(callback)
        if (set.size === 0) {
          keyCallbacks.delete(key)
        }
      }
    }
  }, [key, callback])

  useQuery({
    queryKey: ['global-keydown'],
    queryFn: () => {
      setupGlobalKeyListener()
      return null
    },
    staleTime: Infinity,
    refetchOnWindowFocus: false,
  })
}
```

### 3.2 Use Passive Event Listeners for Scrolling Performance

**Impact: MEDIUM (eliminates scroll delay)**

Add `{ passive: true }` to touch and wheel event listeners.

**Incorrect:**

```typescript
useEffect(() => {
  const handleTouch = (e: TouchEvent) => console.log(e.touches[0].clientX)
  
  document.addEventListener('touchstart', handleTouch)
  
  return () => {
    document.removeEventListener('touchstart', handleTouch)
  }
}, [])
```

**Correct:**

```typescript
useEffect(() => {
  const handleTouch = (e: TouchEvent) => console.log(e.touches[0].clientX)
  
  document.addEventListener('touchstart', handleTouch, { passive: true })
  
  return () => {
    document.removeEventListener('touchstart', handleTouch)
  }
}, [])
```

### 3.3 Use React Query for Automatic Deduplication

**Impact: MEDIUM-HIGH (automatic deduplication)**

```tsx
import { useQuery } from '@tanstack/react-query'

function UserList() {
  const { data: users } = useQuery({
    queryKey: ['users'],
    queryFn: () => fetch('/api/users').then(res => res.json())
  })
}
```

### 3.4 Version and Minimize localStorage Data

**Impact: MEDIUM (prevents schema conflicts)**

```typescript
const VERSION = 'v2'

function saveConfig(config: { theme: string; language: string }) {
  try {
    localStorage.setItem(`userConfig:${VERSION}`, JSON.stringify(config))
  } catch {
    // Throws in incognito/private browsing
  }
}

function loadConfig() {
  try {
    const data = localStorage.getItem(`userConfig:${VERSION}`)
    return data ? JSON.parse(data) : null
  } catch {
    return null
  }
}
```

---

## 4. Re-render Optimization

**Impact: MEDIUM**

### 4.1 Defer State Reads to Usage Point

**Impact: MEDIUM (avoids unnecessary subscriptions)**

**Incorrect:**

```tsx
function ShareButton({ chatId }: { chatId: string }) {
  const searchParams = useSearchParams()

  const handleShare = () => {
    const ref = searchParams.get('ref')
    shareChat(chatId, { ref })
  }

  return <button onClick={handleShare}>Share</button>
}
```

**Correct:**

```tsx
function ShareButton({ chatId }: { chatId: string }) {
  const handleShare = () => {
    const params = new URLSearchParams(window.location.search)
    const ref = params.get('ref')
    shareChat(chatId, { ref })
  }

  return <button onClick={handleShare}>Share</button>
}
```

### 4.2 Extract to Memoized Components

**Impact: MEDIUM (enables early returns)**

**Incorrect:**

```tsx
function Profile({ user, loading }: Props) {
  const avatar = useMemo(() => {
    const id = computeAvatarId(user)
    return <Avatar id={id} />
  }, [user])

  if (loading) return <Skeleton />
  return <div>{avatar}</div>
}
```

**Correct:**

```tsx
const UserAvatar = memo(function UserAvatar({ user }: { user: User }) {
  const id = useMemo(() => computeAvatarId(user), [user])
  return <Avatar id={id} />
})

function Profile({ user, loading }: Props) {
  if (loading) return <Skeleton />
  return (
    <div>
      <UserAvatar user={user} />
    </div>
  )
}
```

> **Note:** When the React Compiler is enabled, manual memoization is not necessary.

### 4.3 Narrow Effect Dependencies

**Impact: LOW (minimizes effect re-runs)**

**Incorrect:**

```tsx
useEffect(() => {
  console.log(user.id)
}, [user])
```

**Correct:**

```tsx
useEffect(() => {
  console.log(user.id)
}, [user.id])
```

### 4.4 Subscribe to Derived State

**Impact: MEDIUM (reduces re-render frequency)**

**Incorrect:**

```tsx
function Sidebar() {
  const width = useWindowWidth()  // updates continuously
  const isMobile = width < 768
  return <nav className={isMobile ? 'mobile' : 'desktop'} />
}
```

**Correct:**

```tsx
function Sidebar() {
  const isMobile = useMediaQuery('(max-width: 767px)')
  return <nav className={isMobile ? 'mobile' : 'desktop'} />
}
```

### 4.5 Use Functional setState Updates

**Impact: MEDIUM (prevents stale closures)**

**Incorrect:**

```tsx
function TodoList() {
  const [items, setItems] = useState(initialItems)
  
  const addItems = useCallback((newItems: Item[]) => {
    setItems([...items, ...newItems])
  }, [items])
}
```

**Correct:**

```tsx
function TodoList() {
  const [items, setItems] = useState(initialItems)
  
  const addItems = useCallback((newItems: Item[]) => {
    setItems(curr => [...curr, ...newItems])
  }, [])
}
```

### 4.6 Use Lazy State Initialization

**Impact: MEDIUM (avoids wasted computation)**

**Incorrect:**

```tsx
const [settings, setSettings] = useState(
  JSON.parse(localStorage.getItem('settings') || '{}')
)
```

**Correct:**

```tsx
const [settings, setSettings] = useState(() => {
  const stored = localStorage.getItem('settings')
  return stored ? JSON.parse(stored) : {}
})
```

### 4.7 Use Transitions for Non-Urgent Updates

**Impact: MEDIUM (maintains UI responsiveness)**

**Incorrect:**

```tsx
function ScrollTracker() {
  const [scrollY, setScrollY] = useState(0)
  useEffect(() => {
    const handler = () => setScrollY(window.scrollY)
    window.addEventListener('scroll', handler, { passive: true })
    return () => window.removeEventListener('scroll', handler)
  }, [])
}
```

**Correct:**

```tsx
import { startTransition } from 'react'

function ScrollTracker() {
  const [scrollY, setScrollY] = useState(0)
  useEffect(() => {
    const handler = () => {
      startTransition(() => setScrollY(window.scrollY))
    }
    window.addEventListener('scroll', handler, { passive: true })
    return () => window.removeEventListener('scroll', handler)
  }, [])
}
```

---

## 5. Rendering Performance

**Impact: MEDIUM**

### 5.1 Animate SVG Wrapper Instead of SVG Element

**Impact: LOW (enables hardware acceleration)**

**Incorrect:**

```tsx
function LoadingSpinner() {
  return (
    <svg className="animate-spin" width="24" height="24" viewBox="0 0 24 24">
      <circle cx="12" cy="12" r="10" stroke="currentColor" />
    </svg>
  )
}
```

**Correct:**

```tsx
function LoadingSpinner() {
  return (
    <div className="animate-spin">
      <svg width="24" height="24" viewBox="0 0 24 24">
        <circle cx="12" cy="12" r="10" stroke="currentColor" />
      </svg>
    </div>
  )
}
```

### 5.2 CSS content-visibility for Long Lists

**Impact: HIGH (faster initial render)**

```css
.message-item {
  content-visibility: auto;
  contain-intrinsic-size: 0 80px;
}
```

```tsx
function MessageList({ messages }: { messages: Message[] }) {
  return (
    <div className="overflow-y-auto h-screen">
      {messages.map(msg => (
        <div key={msg.id} className="message-item">
          <Avatar user={msg.author} />
          <div>{msg.content}</div>
        </div>
      ))}
    </div>
  )
}
```

### 5.3 Hoist Static JSX Elements

**Impact: LOW (avoids re-creation)**

**Incorrect:**

```tsx
function Container() {
  return (
    <div>
      {loading && <div className="animate-pulse h-20 bg-gray-200" />}
    </div>
  )
}
```

**Correct:**

```tsx
const loadingSkeleton = (
  <div className="animate-pulse h-20 bg-gray-200" />
)

function Container() {
  return (
    <div>
      {loading && loadingSkeleton}
    </div>
  )
}
```

### 5.4 Prevent Hydration Mismatch Without Flickering

**Impact: MEDIUM (avoids visual flicker)**

**Correct:**

```tsx
function ThemeWrapper({ children }: { children: ReactNode }) {
  return (
    <>
      <div id="theme-wrapper">
        {children}
      </div>
      <script
        dangerouslySetInnerHTML={{
          __html: `
            (function() {
              try {
                var theme = localStorage.getItem('theme') || 'light';
                var el = document.getElementById('theme-wrapper');
                if (el) el.className = theme;
              } catch (e) {}
            })();
          `,
        }}
      />
    </>
  )
}
```

### 5.5 Use Activity Component for Show/Hide

**Impact: MEDIUM (preserves state/DOM)**

```tsx
import { Activity } from 'react'

function Dropdown({ isOpen }: Props) {
  return (
    <Activity mode={isOpen ? 'visible' : 'hidden'}>
      <ExpensiveMenu />
    </Activity>
  )
}
```

### 5.6 Use Explicit Conditional Rendering

**Impact: LOW (prevents rendering 0 or NaN)**

**Incorrect:**

```tsx
function Badge({ count }: { count: number }) {
  return (
    <div>
      {count && <span className="badge">{count}</span>}
    </div>
  )
}
// When count = 0, renders: <div>0</div>
```

**Correct:**

```tsx
function Badge({ count }: { count: number }) {
  return (
    <div>
      {count > 0 ? <span className="badge">{count}</span> : null}
    </div>
  )
}
```

---

## 6. JavaScript Performance

**Impact: LOW-MEDIUM**

### 6.1 Batch DOM CSS Changes

**Impact: MEDIUM (reduces reflows)**

**Incorrect:**

```typescript
element.style.width = '100px'
element.style.height = '200px'
element.style.backgroundColor = 'blue'
```

**Correct:**

```typescript
element.classList.add('highlighted-box')
// or
element.style.cssText = `width: 100px; height: 200px; background-color: blue;`
```

### 6.2 Build Index Maps for Repeated Lookups

**Impact: LOW-MEDIUM (O(n) to O(1))**

**Incorrect:**

```typescript
function processOrders(orders: Order[], users: User[]) {
  return orders.map(order => ({
    ...order,
    user: users.find(u => u.id === order.userId)
  }))
}
```

**Correct:**

```typescript
function processOrders(orders: Order[], users: User[]) {
  const userById = new Map(users.map(u => [u.id, u]))

  return orders.map(order => ({
    ...order,
    user: userById.get(order.userId)
  }))
}
```

### 6.3 Use Set/Map for O(1) Lookups

**Incorrect:**

```typescript
const allowedIds = ['a', 'b', 'c', ...]
items.filter(item => allowedIds.includes(item.id))
```

**Correct:**

```typescript
const allowedIds = new Set(['a', 'b', 'c', ...])
items.filter(item => allowedIds.has(item.id))
```

### 6.4 Use toSorted() Instead of sort() for Immutability

**Impact: MEDIUM-HIGH (prevents mutation bugs)**

**Incorrect:**

```typescript
const sorted = useMemo(
  () => users.sort((a, b) => a.name.localeCompare(b.name)),
  [users]
)
```

**Correct:**

```typescript
const sorted = useMemo(
  () => users.toSorted((a, b) => a.name.localeCompare(b.name)),
  [users]
)
```

---

## 7. Advanced Patterns

**Impact: LOW**

### 7.1 Store Event Handlers in Refs

**Impact: LOW (stable subscriptions)**

**Incorrect:**

```tsx
function useWindowEvent(event: string, handler: () => void) {
  useEffect(() => {
    window.addEventListener(event, handler)
    return () => window.removeEventListener(event, handler)
  }, [event, handler])
}
```

**Correct:**

```tsx
import { useEffectEvent } from 'react'

function useWindowEvent(event: string, handler: () => void) {
  const onEvent = useEffectEvent(handler)

  useEffect(() => {
    window.addEventListener(event, onEvent)
    return () => window.removeEventListener(event, onEvent)
  }, [event])
}
```

### 7.2 useLatest for Stable Callback Refs

**Impact: LOW (prevents effect re-runs)**

```typescript
function useLatest<T>(value: T) {
  const ref = useRef(value)
  useEffect(() => {
    ref.current = value
  }, [value])
  return ref
}
```

**Usage:**

```tsx
function SearchInput({ onSearch }: { onSearch: (q: string) => void }) {
  const [query, setQuery] = useState('')
  const onSearchRef = useLatest(onSearch)

  useEffect(() => {
    const timeout = setTimeout(() => onSearchRef.current(query), 300)
    return () => clearTimeout(timeout)
  }, [query])
}
```

---

## References

1. [https://react.dev](https://react.dev)
2. [https://tanstack.com/query](https://tanstack.com/query)
