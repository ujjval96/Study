# 📘 React Interview Preparation Handbook
### Senior / Staff / Architect Level (10–15 Years Experience)
> Covers 120+ questions across 10 sections. No fluff. No beginner content. Production-grade depth.

---

## TABLE OF CONTENTS
1. [React Core & Internals](#section-1)
2. [Hooks – Deep Internals](#section-2)
3. [Performance Optimization](#section-3)
4. [Component Architecture](#section-4)
5. [State Management](#section-5)
6. [Rendering Strategies](#section-6)
7. [Security](#section-7)
8. [Testing](#section-8)
9. [System Design – Frontend](#section-9)
10. [Real-World Debugging](#section-10)

---

# SECTION 1: REACT CORE & INTERNALS {#section-1}

---

## Q1. What is React and how does it work internally?

### Short Answer
React is a declarative UI library that maintains a virtual representation of the DOM (Virtual DOM) and efficiently syncs changes to the real DOM through a diffing + reconciliation process, now powered by the Fiber architecture for concurrent, interruptible rendering.

### Deep Explanation
React's rendering pipeline has two major phases:

**1. Render Phase (pure, side-effect free)**
- React calls your component functions / render methods
- Builds a new Fiber tree (work-in-progress tree)
- Diffs this against the current tree
- Computes the minimal set of DOM mutations needed

**2. Commit Phase (has side effects)**
- Applies the computed mutations to the real DOM
- Runs `useLayoutEffect` (synchronous, before paint)
- Browser paints the screen
- Runs `useEffect` (asynchronous, after paint)

```
User Action / setState
        ↓
  Schedule Work (Scheduler)
        ↓
  Render Phase
  ┌─────────────────────────────────────┐
  │  beginWork() → process fiber node   │
  │  completeWork() → bubble up effects │
  │  [Can be interrupted/paused here]   │
  └─────────────────────────────────────┘
        ↓
  Commit Phase (cannot be interrupted)
  ┌─────────────────────────────────────┐
  │  Before Mutation → getSnapshotBeforeUpdate │
  │  Mutation → DOM updates             │
  │  Layout → useLayoutEffect           │
  └─────────────────────────────────────┘
        ↓
  Browser Paint
        ↓
  useEffect callbacks
```

### Code Example
```jsx
// What you write:
function Counter({ initialCount }) {
  const [count, setCount] = useState(initialCount);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// What React does internally (simplified):
// 1. Creates a Fiber node for Counter with { memoizedState: initialCount }
// 2. On click → schedules update via dispatchAction
// 3. Render phase: re-runs Counter(), produces new React element tree
// 4. Reconciler diffs old fiber (count: 0) vs new (count: 1)
// 5. Marks fiber with "Update" effect flag
// 6. Commit phase: patches button textContent in real DOM
```

### Follow-up Questions
- "Where exactly does React decide to skip re-rendering?"
- "How does React prioritize multiple state updates?"
- "What happens if an error is thrown during the render phase?"

### Common Mistakes
- Confusing render phase with commit phase — side effects in render phase cause bugs in Concurrent mode
- Thinking `setState` triggers a synchronous DOM update — it schedules work

### Trade-offs
| | Virtual DOM Approach | Direct DOM Manipulation |
|---|---|---|
| Developer experience | Declarative, predictable | Imperative, error-prone |
| Small updates | Slight overhead (diff cost) | Faster for targeted patches |
| Large/complex UIs | Efficient batching | Hard to optimize manually |
| Concurrent features | Possible (Fiber) | Not possible |

---

## Q2. Virtual DOM vs Real DOM — Diffing Algorithm & Complexity

### Short Answer
React's diffing (reconciliation) algorithm runs in O(n) instead of the theoretical O(n³) by applying three heuristics: same-level comparison only, same `type` assumption, and `key`-based list diffing.

### Deep Explanation

**Three Heuristics React Uses:**

**Heuristic 1: Same-level comparison only**
React never compares nodes across tree levels. If a parent type changes (e.g., `<div>` → `<span>`), the entire subtree is unmounted and remounted. Zero cross-level diffing.

**Heuristic 2: Same type → update, different type → remount**
```jsx
// Old tree:         New tree:
<div>               <span>      ← different type → unmount div, mount span
  <Counter />         <Counter />
</div>              </span>
// Counter is DESTROYED and recreated — state is LOST
```

**Heuristic 3: Keys stabilize list identity**
```jsx
// Without keys — React diffs by position:
// Old: [A, B, C]  →  New: [X, A, B, C]
// React sees position 0 changed A→X, position 1 changed B→A, etc.
// 3 updates instead of 1 insert

// With keys — React diffs by identity:
// Old: [{key:'a',A}, {key:'b',B}]  →  New: [{key:'x',X}, {key:'a',A}, {key:'b',B}]
// React sees: insert X at start, A and B are unchanged ✓
```

**The Real Diffing Cost:**
```
Tree with n nodes:
- Naive algorithm: O(n³)  [find all possible matches across trees]
- React's algorithm: O(n)  [single DFS pass with heuristics]

But: key-based list diffing is still O(k log k) where k = list length
     due to map construction for key lookup
```

### Code Example — Keys done wrong in production
```jsx
// ❌ WRONG: index as key — causes bugs with reordering + state
function TodoList({ todos }) {
  return todos.map((todo, index) => (
    <TodoItem key={index} todo={todo} />  // If you insert at top, all states shift
  ));
}

// ❌ WRONG: random key — forces full remount every render
function TodoList({ todos }) {
  return todos.map(todo => (
    <TodoItem key={Math.random()} todo={todo} />  // Every render = destroy + create
  ));
}

// ✅ CORRECT: stable, unique, semantic key
function TodoList({ todos }) {
  return todos.map(todo => (
    <TodoItem key={todo.id} todo={todo} />  // React tracks by todo.id
  ));
}

// ✅ ACCEPTABLE: index as key ONLY when:
// 1. List is never reordered
// 2. Items have no local state
// 3. List is static / append-only
```

### Follow-up Questions
- "What happens to `useEffect` cleanup when a component is remounted due to type change?"
- "Why does wrapping a component in a different container cause state loss?"
- "How do you force a component to fully reset its state?"

### Common Mistakes
- Using array index as key in sortable/filterable lists (causes state corruption)
- Wrapping a component in a conditional different element type and being surprised state resets
- Forgetting that `key` forces remount — useful as a reset mechanism

### Trade-offs
Using `key` to force reset:
```jsx
// Intentional remount to reset state — valid pattern
<UserProfile key={userId} userId={userId} />
// When userId changes, entire component tree resets → avoids manual state cleanup
```

---

## Q3. Fiber Architecture — Priority Scheduling & Interruptible Rendering

### Short Answer
React Fiber is a reimplementation of React's core algorithm that represents each unit of work as a linked list node (Fiber), enabling the scheduler to pause, resume, abort, or prioritize work — unlocking concurrent rendering.

### Deep Explanation

**What is a Fiber?**
A Fiber is a JavaScript object representing a unit of work for a single component:

```javascript
// Simplified Fiber node structure
{
  type: Counter,           // Component function or DOM tag string
  key: null,
  stateNode: domNode,      // For host components, the actual DOM node

  // Linked list pointers — this is the "tree"
  child: Fiber | null,     // First child
  sibling: Fiber | null,   // Next sibling
  return: Fiber | null,    // Parent (NOT called "parent" to avoid confusion)

  // Work
  pendingProps: {},        // Props for this render
  memoizedProps: {},       // Props from last render
  memoizedState: Hook | null, // Head of hooks linked list
  updateQueue: UpdateQueue | null,

  // Effect tracking
  flags: Flags,            // Bitmask: Placement | Update | Deletion | etc.
  subtreeFlags: Flags,     // Aggregated flags from subtree (React 18 optimization)
  
  // Scheduling
  lanes: Lanes,            // Priority lanes (bitmask)
  childLanes: Lanes,
  
  // Double buffering
  alternate: Fiber | null, // Points to the other tree (current ↔ work-in-progress)
}
```

**Double Buffering — The Two-Tree Architecture:**
```
Current Tree (displayed)    Work-In-Progress Tree (being built)
┌──────────────────┐        ┌──────────────────┐
│   App Fiber      │◄──────►│   App Fiber      │
│   ↓              │alternate│   ↓              │
│   Header Fiber   │◄──────►│   Header Fiber   │
│   ↓              │        │   ↓              │
│   Main Fiber     │◄──────►│   Main Fiber     │
└──────────────────┘        └──────────────────┘
                                    ↑
                             React builds this
                             Once complete → swap
                             WIP becomes Current
```

**Priority Lanes (React 18):**
```javascript
// Simplified lane model (React source uses bitmask)
const SyncLane           = 0b0000000000000000000000000000001; // Highest
const InputContinuousLane= 0b0000000000000000000000000000100;
const DefaultLane        = 0b0000000000000000000000000010000;
const TransitionLane1    = 0b0000000000000000000000001000000;
const IdleLane           = 0b0100000000000000000000000000000; // Lowest

// startTransition marks updates as TransitionLane — can be interrupted
// Direct user input (typing) gets SyncLane or InputContinuousLane — never interrupted
```

### Code Example — Seeing Fiber's Interruptibility
```jsx
import { useState, startTransition } from 'react';

function SearchApp() {
  const [inputValue, setInputValue] = useState('');
  const [searchQuery, setSearchQuery] = useState('');

  function handleInput(e) {
    // Input update: SyncLane (high priority) — never interrupted
    setInputValue(e.target.value);

    // Search results update: TransitionLane (low priority) — CAN be interrupted
    // If user types again before results render, React discards the old work
    startTransition(() => {
      setSearchQuery(e.target.value);
    });
  }

  return (
    <>
      <input value={inputValue} onChange={handleInput} />
      {/* This can show stale results while new ones compute */}
      <Suspense fallback={<Spinner />}>
        <SearchResults query={searchQuery} />
      </Suspense>
    </>
  );
}

// The key insight: without startTransition, a slow SearchResults component
// would BLOCK the input from feeling responsive.
// With startTransition, React can pause SearchResults re-render when the user types again.
```

### Follow-up Questions
- "How does React ensure the UI doesn't show a partial state from an interrupted render?"
- "What is the `lanes` bitmask and why use bitmask instead of a priority number?"
- "How does Suspense integrate with Fiber scheduling?"

### Common Mistakes
- Thinking Fiber means renders happen on separate threads — it's still single-threaded, just interruptible via `MessageChannel` / `setTimeout` yielding
- Using `startTransition` for everything — synchronous urgent updates shouldn't be in transitions

---

## Q4. Render Phase vs Commit Phase

### Short Answer
The render phase is pure and may be interrupted/replayed (especially in Concurrent mode and StrictMode). The commit phase is synchronous, cannot be interrupted, and is where actual DOM mutations and most side effects happen.

### Deep Explanation

```
RENDER PHASE (Reconciliation)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• Pure: No DOM mutations, no external interactions
• Can be paused, restarted, aborted (Concurrent mode)
• StrictMode: deliberately invokes this phase TWICE in dev
• What happens: beginWork() and completeWork() on each fiber
• Output: A tree of fibers tagged with effect flags

COMMIT PHASE (3 sub-phases, all synchronous)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Phase 1: Before Mutation
  • commitBeforeMutationEffects()
  • getSnapshotBeforeUpdate() class component lifecycle
  • Reads DOM state BEFORE mutations (for scroll position, etc.)

Phase 2: Mutation
  • commitMutationEffects()
  • Applies DOM insertions, updates, deletions
  • Calls old useLayoutEffect CLEANUP functions
  • Calls old useEffect CLEANUP functions (scheduled, not run yet)
  • currentFiber is still the OLD tree here

Phase 3: Layout
  • commitLayoutEffects()
  • Switches currentFiber to the new tree
  • Runs componentDidMount / componentDidUpdate
  • Runs useLayoutEffect setup functions synchronously
  • Flushes any setState calls here synchronously

Browser Paint happens here

Passive Effects (async, scheduled via MessageChannel):
  • Runs useEffect cleanup functions
  • Runs useEffect setup functions
```

### Code Example — Why mutation in render phase breaks Concurrent mode
```jsx
// ❌ DANGEROUS: Side effect in render phase
function BadComponent({ userId }) {
  // This runs during render phase
  // In Concurrent mode, this can run multiple times before committing!
  localStorage.setItem('lastViewedUser', userId); // ← WRONG: runs 2x in StrictMode

  return <UserProfile userId={userId} />;
}

// ✅ CORRECT: Side effect in useEffect (commit phase, passive)
function GoodComponent({ userId }) {
  useEffect(() => {
    localStorage.setItem('lastViewedUser', userId); // ← runs exactly once after commit
  }, [userId]);

  return <UserProfile userId={userId} />;
}

// ✅ CORRECT: DOM reading before mutation — use getSnapshotBeforeUpdate / useLayoutEffect
function ScrollPreserver() {
  const ref = useRef(null);
  
  useLayoutEffect(() => {
    // Runs synchronously AFTER mutation, BEFORE paint
    // Use for: measuring DOM, scroll restoration, synchronous animations
    const element = ref.current;
    element.scrollTop = savedScrollPosition;
  });

  return <div ref={ref}>{/* ... */}</div>;
}
```

### Follow-up Questions
- "Why does StrictMode double-invoke render functions? What bugs does it catch?"
- "What happens if you call `setState` inside `useLayoutEffect`?"
- "Why can't `useEffect` be used for scroll restoration?"

### Common Mistakes
- Writing side effects (API calls, subscriptions, DOM mutations) directly in render — these get called twice in StrictMode
- Using `useLayoutEffect` for everything "to be safe" — blocks painting, hurts perf
- Not knowing that `useLayoutEffect` is synchronous and can cause visual glitches if it does heavy work

---

## Q5. Concurrent Rendering (React 18)

### Short Answer
React 18's concurrent rendering allows React to prepare multiple versions of the UI simultaneously, interrupt long renders, and show consistent intermediate states — enabling better perceived performance without blocking user interactions.

### Deep Explanation

**New Concurrent Features in React 18:**

| Feature | API | Purpose |
|---|---|---|
| Concurrent rendering | Enabled by `createRoot()` | Prerequisite for all below |
| Transitions | `startTransition`, `useTransition` | Mark low-priority updates |
| Deferred values | `useDeferredValue` | Debounce derived state rendering |
| Suspense improvements | Streaming, selective hydration | Progressive loading |
| Automatic batching | All updates batched | Fewer re-renders |

### Code Example
```jsx
import { createRoot } from 'react-dom/client';
import { useTransition, useDeferredValue, Suspense } from 'react';

// MUST use createRoot to opt into concurrent features
const root = createRoot(document.getElementById('root'));
root.render(<App />);

// ─────────────────────────────────────────────
// Pattern 1: useTransition — for state updates
function FilterableList({ items }) {
  const [isPending, startTransition] = useTransition();
  const [filter, setFilter] = useState('');
  const [filteredItems, setFilteredItems] = useState(items);

  function handleFilterChange(e) {
    const value = e.target.value;
    setFilter(value); // Urgent: update input immediately

    startTransition(() => {
      // Non-urgent: filter can lag behind
      setFilteredItems(items.filter(item => item.name.includes(value)));
    });
  }

  return (
    <>
      <input value={filter} onChange={handleFilterChange} />
      {isPending && <Spinner />}  {/* Show while transition is pending */}
      <List items={filteredItems} />
    </>
  );
}

// ─────────────────────────────────────────────
// Pattern 2: useDeferredValue — for props you don't control
function SearchResults({ query }) {
  // Defers re-rendering with new query until urgent work is done
  // Shows stale results while new ones compute
  const deferredQuery = useDeferredValue(query);
  const isStale = deferredQuery !== query;

  return (
    <div style={{ opacity: isStale ? 0.7 : 1, transition: 'opacity 0.2s' }}>
      {/* Only re-renders when deferredQuery changes */}
      <ExpensiveResultsList query={deferredQuery} />
    </div>
  );
}
```

### Follow-up Questions
- "How is `useDeferredValue` different from debouncing?"
- "Can you use `useTransition` in an event handler? What about in `useEffect`?"
- "What does `isPending` represent exactly?"

### Common Mistakes
- Using `startTransition` for urgent user feedback (form submit buttons, etc.)
- Forgetting that transitions don't prevent re-renders, they just deprioritize them
- Using `useDeferredValue` when you actually need `useTransition` — the difference matters

---

## Q6. Batching Updates

### Short Answer
React batches multiple `setState` calls into a single re-render. React 18 introduced automatic batching for ALL async contexts (Promise callbacks, setTimeout, native event handlers) — previously only synthetic event handlers were batched.

### Deep Explanation

```
React 17 and below:
  ✅ Batched: onClick, onChange (synthetic events)
  ❌ NOT batched: setTimeout, Promise.then, fetch callbacks, native addEventListener

React 18 (createRoot):
  ✅ Batched: EVERYTHING, including setTimeout, Promises, native events
  ✅ No change required — works automatically
```

### Code Example
```jsx
// React 17: This caused 2 re-renders
fetch('/api/data').then(data => {
  setState1(data.value1);  // re-render 1
  setState2(data.value2);  // re-render 2
});

// React 18 (createRoot): This causes 1 re-render (automatic batching)
fetch('/api/data').then(data => {
  setState1(data.value1);  // batched
  setState2(data.value2);  // batched → single re-render
});

// ─────────────────────────────────────────────
// Opt OUT of batching when needed (rare)
import { flushSync } from 'react-dom';

function handleClick() {
  flushSync(() => {
    setCounter(c => c + 1);
  });
  // DOM is updated HERE — before next line
  // Use case: reading DOM layout immediately after state update
  const height = divRef.current.getBoundingClientRect().height;

  flushSync(() => {
    setFlag(true);
  });
}

// ─────────────────────────────────────────────
// Functional updates to avoid batching race conditions
function incrementTwice() {
  // ❌ WRONG: Both closures capture same count value
  setCount(count + 1);
  setCount(count + 1); // Still just count + 1, not count + 2!

  // ✅ CORRECT: Functional update uses latest state
  setCount(c => c + 1);
  setCount(c => c + 1); // Correctly becomes count + 2
}
```

### Follow-up Questions
- "When would you use `flushSync`? Give a real-world example."
- "How does batching interact with `useReducer`?"
- "What happens with batching in React Server Components?"

### Common Mistakes
- Relying on `setState` being synchronous for reading updated state immediately after
- Not using functional updates (`prev => prev + 1`) when multiple updates happen in sequence
- Using `flushSync` unnecessarily, blocking React's ability to optimize

---

# SECTION 2: HOOKS — DEEP INTERNALS {#section-2}

---

## Q7. How React Tracks Hooks Internally (Linked List & Call Order)

### Short Answer
React stores hooks in a singly-linked list attached to the Fiber node (`memoizedState`). Hooks are identified by their position in the call order, not by name — which is why hooks cannot be called conditionally.

### Deep Explanation

```javascript
// When you write:
function Counter() {
  const [count, setCount] = useState(0);      // Hook #1
  const [name, setName] = useState('Alice');   // Hook #2
  const doubled = useMemo(() => count * 2, [count]); // Hook #3
  useEffect(() => { document.title = name; }, [name]); // Hook #4
  return <div>{count} {name} {doubled}</div>;
}

// React stores in Fiber.memoizedState as linked list:
// Hook1 → Hook2 → Hook3 → Hook4 → null

// Each Hook object looks like:
{
  memoizedState: 0,            // For useState: the state value
                               // For useEffect: the effect object
                               // For useMemo: [value, deps]
                               // For useRef: { current: value }
  baseState: 0,
  baseQueue: null,
  queue: UpdateQueue | null,   // For useState/useReducer
  next: Hook | null            // Pointer to next hook
}

// On EVERY render, React traverses this list in order:
// useState(0) → reads Hook1.memoizedState
// useState('Alice') → reads Hook2.memoizedState
// useMemo(...) → reads Hook3.memoizedState
// etc.

// If you conditionally skip a hook, the list gets out of sync:
// Render 1: Hook1=count, Hook2=name, Hook3=memo
// Render 2 (condition false, Hook1 skipped): React reads Hook1 as name, Hook2 as memo
// → CATASTROPHIC: wrong state mapped to wrong hook
```

### Code Example — Why conditional hooks break
```jsx
// ❌ This is why conditional hooks are forbidden:
function BadCounter({ showDouble }) {
  const [count, setCount] = useState(0);       // Hook #1 always

  if (showDouble) {
    const [doubled, setDoubled] = useState(0); // Hook #2 — ONLY SOMETIMES
  }

  const [name, setName] = useState('');        // Hook #3 or #2 depending on condition
  // When showDouble flips false, name hook reads doubled's slot → wrong value!
}

// ✅ Correct: Move condition INSIDE the hook or use a separate component
function GoodCounter({ showDouble }) {
  const [count, setCount] = useState(0);
  const [doubled, setDoubled] = useState(0); // Always declare
  const [name, setName] = useState('');

  // Condition is fine INSIDE — just don't call fewer hooks
  const displayDouble = showDouble ? doubled : null;
}

// ✅ Alternative: Separate component
function DoubleDisplay({ count }) {
  const [doubled, setDoubled] = useState(count * 2);
  return <span>{doubled}</span>;
}

function GoodCounter2({ showDouble }) {
  const [count, setCount] = useState(0);
  return (
    <>
      {showDouble && <DoubleDisplay count={count} />}  {/* Conditionally render component */}
    </>
  );
}
```

---

## Q8. useState — Internals, Closures, and Edge Cases

### Short Answer
`useState` returns a value from the Fiber's hook linked list and a `dispatch` function. The dispatch schedules an update on the Fiber; the closure captures the dispatch reference which is stable across renders.

### Deep Explanation

**State initialization:**
```javascript
// Lazy initialization — expensive computation runs ONCE
const [state, setState] = useState(() => {
  return expensiveComputation(); // Function form: only called on mount
});

// ❌ This runs on EVERY render:
const [state, setState] = useState(expensiveComputation()); // Eagerly evaluated

// Internal: React checks `hook.memoizedState === null` (mount) vs existing value (update)
// On mount: stores initialState into hook.memoizedState
// On update: dispatches action through UpdateQueue, applies reducer
```

**State batching and the queue:**
```javascript
// React processes state updates as a queue per hook
// Multiple setStates before render are queued then applied

// For useState, the "reducer" is: (state, action) => action
// (action is the new value or the updater function)
```

### Code Example — Stale Closure Problem
```jsx
// ❌ CLASSIC STALE CLOSURE BUG
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1); // "count" is captured at 0, forever
      // After 3 seconds: count is STILL 0 in this closure
      // Result: count never goes above 1
    }, 1000);
    return () => clearInterval(id);
  }, []); // Empty deps — closure captures initial count=0

  return <div>{count}</div>;
}

// ✅ Fix 1: Functional update (recommended)
useEffect(() => {
  const id = setInterval(() => {
    setCount(c => c + 1); // Uses latest state, no closure issue
  }, 1000);
  return () => clearInterval(id);
}, []);

// ✅ Fix 2: Include count in deps (recreates interval every render)
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1);
  }, 1000);
  return () => clearInterval(id);
}, [count]); // Interval resets on every count change — may cause issues

// ✅ Fix 3: useRef to hold latest value
function Timer() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);
  countRef.current = count; // Always fresh

  useEffect(() => {
    const id = setInterval(() => {
      setCount(countRef.current + 1); // Reads latest via ref
    }, 1000);
    return () => clearInterval(id);
  }, []);
}
```

### Follow-up Questions
- "What does React do if `setState` is called with the same value?"
- "How is `useState` implemented in terms of `useReducer`?"
- "When does `useState`'s initializer run? What if the component unmounts and remounts?"

### Common Mistakes
- Reading state immediately after `setState` expecting the new value
- Using state for values derived from props (should use `useMemo` or derive inline)
- Mutating state objects directly — React uses `Object.is` for comparison, not deep equality

---

## Q9. useEffect — The Most Important Hook

### Short Answer
`useEffect` registers a side effect to run after the browser paints. React compares its dependency array using `Object.is` and re-runs the effect only when a dependency changes. The cleanup function runs before the next effect and on unmount.

### Deep Explanation

**Execution Timeline:**
```
Component renders (render phase)
        ↓
DOM is mutated (commit phase, mutation sub-phase)
        ↓
useLayoutEffect runs (synchronous, blocks paint)
        ↓
Browser paints to screen
        ↓
useEffect cleanup from PREVIOUS render runs
        ↓
useEffect setup for CURRENT render runs
```

**Dependency Array Semantics:**
```javascript
useEffect(fn);          // No array: runs after EVERY render
useEffect(fn, []);      // Empty array: runs only on MOUNT (and cleanup on UNMOUNT)
useEffect(fn, [a, b]);  // Array: runs when a or b changes (Object.is comparison)
```

**Internal Storage:**
```javascript
// useEffect stores this in hook.memoizedState:
{
  tag: HookPassive,           // vs HookLayout for useLayoutEffect
  create: setupFunction,      // Your effect function
  destroy: cleanupFunction,   // Return value from create
  deps: [a, b],               // Dependency array
  next: null                  // Next effect in circular linked list
}
```

### Code Example — Real-World useEffect Patterns
```jsx
// ─────────────────────────────────────────────
// Pattern 1: Data fetching with race condition handling
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false; // ← Prevents race condition
    
    setLoading(true);
    setError(null);

    fetchUser(userId)
      .then(data => {
        if (!cancelled) { // ← Only update if this effect is still current
          setUser(data);
          setLoading(false);
        }
      })
      .catch(err => {
        if (!cancelled) {
          setError(err.message);
          setLoading(false);
        }
      });

    return () => {
      cancelled = true; // ← Cleanup: mark as cancelled
    };
  }, [userId]);

  // AbortController alternative (cancels the actual request):
  useEffect(() => {
    const controller = new AbortController();
    
    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then(r => r.json())
      .then(setUser)
      .catch(err => {
        if (err.name !== 'AbortError') setError(err.message);
      });

    return () => controller.abort();
  }, [userId]);
}

// ─────────────────────────────────────────────
// Pattern 2: WebSocket subscription
function LiveChat({ roomId }) {
  const [messages, setMessages] = useState([]);

  useEffect(() => {
    const ws = new WebSocket(`wss://chat.example.com/rooms/${roomId}`);
    
    ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      setMessages(prev => [...prev, msg]); // Functional update!
    };
    
    ws.onerror = () => { /* handle */ };

    return () => {
      ws.close(); // ← Critical: cleanup prevents memory leak and duplicate connections
    };
  }, [roomId]); // ← Re-runs when roomId changes, closing old ws, opening new one
}

// ─────────────────────────────────────────────
// Pattern 3: StrictMode double-invocation handling
// In dev + StrictMode, React runs: setup → cleanup → setup
// This WILL fire twice, which is intentional
// Your effect MUST be idempotent with its cleanup:

useEffect(() => {
  const subscription = subscribe(id); // Setup
  return () => unsubscribe(subscription); // Cleanup must undo setup completely
}, [id]);
// StrictMode: subscribe → unsubscribe → subscribe
// If you see unexpected behavior in dev, your cleanup is incomplete
```

### Common Mistakes
- Not returning cleanup for subscriptions → memory leaks in production
- Missing dependencies in the dependency array → stale closures
- Race conditions from multiple rapid state changes (old fetch resolves after new one)
- Using `async` directly as the effect: `useEffect(async () => {...})` — returns a Promise, not a cleanup function

### useEffect Infinite Loop Causes
```jsx
// Cause 1: Object/array created in render as dependency
useEffect(() => {
  fetchData(options); // options is {limit: 10} — new object every render
}, [options]); // ❌ Object identity changes every render → infinite loop

// Fix: Memoize or destructure
useEffect(() => {
  fetchData({ limit: options.limit });
}, [options.limit]); // ✅ Primitive value

// Cause 2: setState inside effect without condition
useEffect(() => {
  setCount(count + 1); // ← triggers re-render → effect runs again → infinite loop
}, [count]); // count changes every render

// Fix: Only set state based on external events, not state itself
useEffect(() => {
  if (count < 10) setCount(count + 1); // ← at least has a termination condition
}, [count]);

// Cause 3: Function dependency not memoized
function Component() {
  const handleData = () => processData(); // New function every render
  
  useEffect(() => {
    subscribe(handleData);
    return () => unsubscribe(handleData);
  }, [handleData]); // ❌ New function ref every render → infinite loop

  // Fix: useCallback
  const handleData = useCallback(() => processData(), []);
}
```

---

## Q10. useLayoutEffect vs useEffect

### Short Answer
`useLayoutEffect` fires synchronously after DOM mutations but before the browser paints. Use it when you need to read/modify the DOM before the user sees it (scroll restoration, tooltip positioning, animations). Use `useEffect` for everything else.

### Deep Explanation

```
useLayoutEffect runs HERE:   ←──── (synchronous, blocks paint)
                 ↓
     Commit: DOM mutations complete
                 ↓
     useLayoutEffect() ←── runs here, before browser paints
                 ↓
     Browser paints to screen
                 ↓
     useEffect() ←── runs here, after paint (async)
```

### Code Example
```jsx
// USE CASE: Measure DOM element after render, adjust layout
// (e.g., tooltip positioning)
function Tooltip({ targetRef, content }) {
  const tooltipRef = useRef(null);
  const [position, setPosition] = useState({ top: 0, left: 0 });

  // useLayoutEffect: Calculate position BEFORE user sees tooltip at wrong position
  useLayoutEffect(() => {
    const target = targetRef.current;
    const tooltip = tooltipRef.current;
    
    if (!target || !tooltip) return;
    
    const targetRect = target.getBoundingClientRect();
    const tooltipRect = tooltip.getBoundingClientRect();
    
    setPosition({
      top: targetRect.bottom + 8,
      left: targetRect.left + (targetRect.width - tooltipRect.width) / 2,
    });
    // Position is calculated and set synchronously before paint
    // User never sees the "flash" of tooltip at wrong position
  });

  // ❌ WRONG: Using useEffect here → user sees tooltip flash to correct position
  // useEffect(() => { ... }, []); // Async: runs AFTER paint

  return (
    <div ref={tooltipRef} style={{ position: 'fixed', ...position }}>
      {content}
    </div>
  );
}

// SSR WARNING: useLayoutEffect does not run on server
// React 18 fix: useInsertionEffect for CSS-in-JS
// Or suppress warning: if (typeof window !== 'undefined') { ... }
```

---

## Q11. useMemo and useCallback — Internals & When NOT to Use

### Short Answer
`useMemo` memoizes a computed value; `useCallback` memoizes a function reference. Both cache based on dependency comparison using `Object.is`. They have overhead themselves — only use when the saved re-render cost outweighs the memoization cost.

### Deep Explanation

**Internal storage:**
```javascript
// useMemo stores: { memoizedState: [value, deps] }
// On each render: if deps changed → recompute, else return cached value

// useCallback is literally: useMemo(() => fn, deps)
// It memoizes the function reference, not the return value of calling fn

// React's actual implementation of useCallback:
function useCallback(callback, deps) {
  return useMemo(() => callback, deps); // That's it
}
```

### Code Example — When memoization HURTS performance
```jsx
// ─────────────────────────────────────────────
// ❌ Premature optimization — memoization has overhead
// Every useMemo/useCallback:
//   1. Allocates a new array for deps (every render)
//   2. Runs Object.is() comparison for each dep
//   3. Allocates memory to cache the value

// Bad: Memoizing a cheap computation
const doubled = useMemo(() => count * 2, [count]); // More expensive than just: count * 2

// Bad: Memoizing inline JSX (React.memo needed on child for this to help)
const header = useMemo(() => <Header title="Dashboard" />, []); // Pointless without React.memo

// ─────────────────────────────────────────────
// ✅ WHEN useMemo is justified:
// 1. Expensive computation (sorting, filtering large arrays)
function ProductList({ products, filter, sortKey }) {
  const processedProducts = useMemo(() => {
    // 10,000 products × filter + sort = expensive
    return products
      .filter(p => p.category === filter)
      .sort((a, b) => a[sortKey].localeCompare(b[sortKey]));
  }, [products, filter, sortKey]);

  return <List items={processedProducts} />;
}

// 2. Referential stability for child component / hook deps
function Parent() {
  const [count, setCount] = useState(0);
  const [theme, setTheme] = useState('dark');

  // Without useCallback: new function ref every render → Child always re-renders
  const handleSubmit = useCallback((data) => {
    submitForm(data, count);
  }, [count]); // New reference ONLY when count changes

  return (
    <>
      <button onClick={() => setTheme(t => t === 'dark' ? 'light' : 'dark')}>
        Toggle Theme
      </button>
      {/* Theme changes don't cause Child to re-render (it uses React.memo) */}
      <Child onSubmit={handleSubmit} />
    </>
  );
}

const Child = React.memo(({ onSubmit }) => {
  // Only re-renders when onSubmit reference changes
  return <button onClick={() => onSubmit({ data: 'test' })}>Submit</button>;
});
```

### Comparison Table
| | `useMemo` | `useCallback` | `React.memo` |
|---|---|---|---|
| What it memoizes | Computed value | Function reference | Component output |
| Where it lives | Inside component | Inside component | Wraps component |
| Overhead | Allocation + comparison | Allocation + comparison | Props comparison |
| Skip if | Computation is cheap | No deps / no referential need | Props change frequently |

---

## Q12. useRef — Beyond DOM References

### Short Answer
`useRef` returns a mutable container (`{ current: value }`) that persists across renders without triggering re-renders. It's the escape hatch for values that need to survive re-renders but shouldn't cause them.

### Deep Explanation

**Three use cases:**
1. **DOM ref** — access DOM node directly
2. **Instance variable** — like `this.x` in class components
3. **Memoizing previous value** — track previous render's value

```javascript
// Internal storage: hook.memoizedState = { current: initialValue }
// The SAME object is returned on every render
// Changing .current does not schedule a re-render
```

### Code Example
```jsx
// ─────────────────────────────────────────────
// Use case 1: Interval ID tracking (mutable, no re-render needed)
function Stopwatch() {
  const [elapsed, setElapsed] = useState(0);
  const intervalRef = useRef(null); // Store interval ID without re-rendering

  const start = useCallback(() => {
    intervalRef.current = setInterval(() => {
      setElapsed(t => t + 10);
    }, 10);
  }, []);

  const stop = useCallback(() => {
    clearInterval(intervalRef.current); // Access latest interval ID
  }, []);

  return (/* ... */);
}

// ─────────────────────────────────────────────
// Use case 2: Previous value tracking
function usePrevious(value) {
  const prevRef = useRef(undefined);
  
  useEffect(() => {
    prevRef.current = value; // After render, update ref to current value
  }); // No deps: runs after every render

  return prevRef.current; // Returns value from PREVIOUS render
}

function PriceTracker({ price }) {
  const prevPrice = usePrevious(price);
  const direction = price > prevPrice ? '↑' : price < prevPrice ? '↓' : '–';
  return <span>{price} {direction}</span>;
}

// ─────────────────────────────────────────────
// Use case 3: Stable event handler without stale closures
// (The "ref callback" pattern — avoids useCallback dependency issues)
function SearchInput({ onSearch, delay }) {
  const onSearchRef = useRef(onSearch);
  onSearchRef.current = onSearch; // Always latest, no stale closure

  useEffect(() => {
    const timer = setTimeout(() => {
      onSearchRef.current(query); // Calls latest onSearch
    }, delay);
    return () => clearTimeout(timer);
  }, [query, delay]); // onSearch NOT in deps — no stale closure issue
}
```

---

## Q13. useContext — Performance Pitfalls

### Short Answer
`useContext` re-renders ALL consumers whenever context value changes, even if the consuming component only uses part of the context. This makes context a poor choice for high-frequency updates.

### Deep Explanation

```javascript
// Context comparison: React uses Object.is on the VALUE provided
// If you provide an object literal, new object every render = all consumers re-render

const ThemeContext = createContext(null);

// ❌ BAD: New object every render → all consumers re-render every render
function App() {
  const [theme, setTheme] = useState('dark');
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}> {/* ← New object every render */}
      <DeepTree />
    </ThemeContext.Provider>
  );
}

// ✅ GOOD: Stable reference — only re-renders consumers when theme changes
function App() {
  const [theme, setTheme] = useState('dark');
  const value = useMemo(() => ({ theme, setTheme }), [theme]); // setTheme is stable
  return (
    <ThemeContext.Provider value={value}>
      <DeepTree />
    </ThemeContext.Provider>
  );
}
```

### Code Example — Context Splitting for Performance
```jsx
// ─────────────────────────────────────────────
// Pattern: Split context by update frequency
// Instead of one big context, split into:
// 1. Static context (seldom changes) — theme, locale, user profile
// 2. Dynamic context (frequent changes) — current selection, hover state

const UserContext = createContext(null);        // User profile: rarely changes
const NotificationContext = createContext(null); // Notification count: changes often

// Components consuming UserContext don't re-render when notifications change
function UserMenu() {
  const user = useContext(UserContext); // Only re-renders when user changes
  return <span>{user.name}</span>;
}

function NotificationBell() {
  const { count } = useContext(NotificationContext); // Only re-renders for count
  return <span>{count}</span>;
}

// ─────────────────────────────────────────────
// Pattern: Selector pattern (zustand-style) using useSyncExternalStore
// Context doesn't support selectors natively
// Use-context-selector library or state managers for granular subscriptions
import { useContextSelector } from 'use-context-selector'; // 3rd party

const store = createContext(null);

function Counter() {
  // Only re-renders when count changes, NOT when other parts of store change
  const count = useContextSelector(store, s => s.count);
  return <span>{count}</span>;
}
```

---

## Q14. Custom Hooks — Design Patterns

### Short Answer
Custom hooks extract stateful logic for reuse across components. They must start with `use`, can call other hooks, and each call creates completely independent state — not a singleton.

### Code Example — Production-quality custom hooks
```jsx
// ─────────────────────────────────────────────
// Hook 1: useAsync — handles loading/error/data states
function useAsync(asyncFn, deps = []) {
  const [state, dispatch] = useReducer(
    (state, action) => ({ ...state, ...action }),
    { data: null, loading: false, error: null }
  );

  const fnRef = useRef(asyncFn);
  fnRef.current = asyncFn; // Always latest

  useEffect(() => {
    let cancelled = false;
    dispatch({ loading: true, error: null });
    
    fnRef.current()
      .then(data => { if (!cancelled) dispatch({ data, loading: false }); })
      .catch(error => { if (!cancelled) dispatch({ error, loading: false }); });
    
    return () => { cancelled = true; };
  // eslint-disable-next-line react-hooks/exhaustive-deps
  }, deps);

  return state;
}

// Usage:
function UserPage({ userId }) {
  const { data: user, loading, error } = useAsync(
    () => fetchUser(userId),
    [userId]
  );
  // ...
}

// ─────────────────────────────────────────────
// Hook 2: useDebounce — search input optimization
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);
  
  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer); // Clear on every value change
  }, [value, delay]);
  
  return debouncedValue;
}

// ─────────────────────────────────────────────
// Hook 3: useLocalStorage — with cross-tab sync
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setStoredValue = useCallback((newValue) => {
    try {
      const valueToStore = newValue instanceof Function ? newValue(value) : newValue;
      setValue(valueToStore);
      localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error('localStorage write failed:', error);
    }
  }, [key, value]);

  // Cross-tab sync
  useEffect(() => {
    const handleStorage = (e) => {
      if (e.key === key && e.newValue !== null) {
        setValue(JSON.parse(e.newValue));
      }
    };
    window.addEventListener('storage', handleStorage);
    return () => window.removeEventListener('storage', handleStorage);
  }, [key]);

  return [value, setStoredValue];
}
```

---

# SECTION 3: PERFORMANCE OPTIMIZATION {#section-3}

---

## Q15. Why Do Components Re-render? Complete Taxonomy

### Short Answer
A React component re-renders when: its own state changes, its parent re-renders (by default), its consumed context value changes, or a hook it uses internally triggers a re-render.

### Deep Explanation

**Re-render Triggers:**
```
1. setState / dispatch called inside the component
2. Parent component re-renders (most common, unexpected cause)
3. Context value changes (useContext consumers)
4. Hook internal state changes (useReducer, custom hooks)
5. forceUpdate (class components)

NOT a re-render trigger:
• Ref changes (useRef, createRef)
• DOM events without setState
• External variable changes (non-reactive)
```

### Code Example — Diagnosing Re-renders
```jsx
// Tool: React DevTools Profiler
// But for quick debugging, use the render counter pattern:

function withRenderCount(Component) {
  return function WrappedComponent(props) {
    const renderCount = useRef(0);
    renderCount.current++;
    console.log(`${Component.displayName || Component.name} rendered ${renderCount.current} times`);
    return <Component {...props} />;
  };
}

// Or use the why-did-you-render library in development:
// import whyDidYouRender from '@welldone-software/why-did-you-render';
// Component.whyDidYouRender = true;

// ─────────────────────────────────────────────
// PROBLEM: Parent re-renders → all children re-render
function Dashboard() {
  const [selectedTab, setSelectedTab] = useState('overview');
  const [notifications, setNotifications] = useState([]);

  // notifications change every 30 seconds
  useEffect(() => {
    const id = setInterval(() => fetchNotifications().then(setNotifications), 30000);
    return () => clearInterval(id);
  }, []);

  return (
    <div>
      <TabBar selected={selectedTab} onChange={setSelectedTab} />
      {/* This re-renders every 30s even if selectedTab didn't change */}
      <ExpensiveChart tab={selectedTab} />
      <NotificationList items={notifications} />
    </div>
  );
}

// SOLUTION: Separate volatile from stable state OR React.memo
const ExpensiveChart = React.memo(({ tab }) => {
  // Only re-renders when tab changes, not when notifications update
  return <HeavyVisualization tab={tab} />;
});
```

---

## Q16. React.memo, useMemo, useCallback — When Each Helps

### Short Answer
`React.memo` prevents re-renders when props are shallowly equal. It only helps when the parent re-renders but props don't change. `useMemo`/`useCallback` help maintain referential stability so `React.memo`'d components actually see unchanged props.

### Deep Explanation

**The Three-Layer Memoization Strategy:**
```
Layer 1: React.memo (component boundary)
  → Prevents re-render when parent re-renders with same props
  → Props compared with shallow equality (Object.is per prop)
  → Overhead: stores previous props, runs comparison every parent render

Layer 2: useMemo (value stability)
  → Ensures object/array props don't create new references
  → Paired with React.memo for effect

Layer 3: useCallback (function stability)
  → Ensures callback props don't create new references
  → Paired with React.memo for effect
```

### Code Example — Complete Memoization Chain
```jsx
// Scenario: Dashboard with filters, 10k row table, expensive chart
// Filters change frequently, chart data changes rarely

const ExpensiveChart = React.memo(function ExpensiveChart({ data, config }) {
  // React.memo: skip re-render if data and config haven't changed
  return <D3Chart data={data} config={config} />;
});

const DataTable = React.memo(function DataTable({ rows, onRowClick }) {
  // Only re-renders when rows array or onRowClick reference changes
  return <VirtualizedTable rows={rows} onRowClick={onRowClick} />;
});

function Dashboard({ rawData }) {
  const [filter, setFilter] = useState({ category: 'all', status: 'active' });
  const [selectedRow, setSelectedRow] = useState(null);

  // useMemo: stable reference for filtered data
  const filteredRows = useMemo(() => {
    return rawData
      .filter(r => filter.category === 'all' || r.category === filter.category)
      .filter(r => r.status === filter.status);
  }, [rawData, filter.category, filter.status]);

  // useMemo: chart data derived from filtered rows
  const chartData = useMemo(() => {
    return aggregateForChart(filteredRows);
  }, [filteredRows]);

  // useMemo: stable config object for chart
  const chartConfig = useMemo(() => ({
    colors: ['#4CAF50', '#f44336'],
    showLegend: true,
    animate: filteredRows.length < 1000, // Disable animation for large sets
  }), [filteredRows.length]);

  // useCallback: stable function reference for table
  const handleRowClick = useCallback((row) => {
    setSelectedRow(row);
    analytics.track('row_clicked', { id: row.id });
  }, []); // analytics is stable (imported module)

  return (
    <>
      <FilterPanel filter={filter} onChange={setFilter} />
      <ExpensiveChart data={chartData} config={chartConfig} />
      <DataTable rows={filteredRows} onRowClick={handleRowClick} />
    </>
  );
}

// ─────────────────────────────────────────────
// When React.memo DOESN'T help:
const Item = React.memo(({ config }) => <div>{config.label}</div>);

function List() {
  return items.map(item => (
    <Item
      key={item.id}
      config={{ label: item.name }} // ← New object every render! React.memo is useless
    />
  ));
}
// Fix: Pass primitive props OR memoize config upstream
```

---

## Q17. List Virtualization & Windowing

### Short Answer
Virtualization renders only the items visible in the viewport plus a small buffer. This reduces DOM nodes from O(n) to O(viewport_height / item_height), making 100k row lists feel instant.

### Code Example
```jsx
// react-window: lightweight, for fixed-size rows
import { FixedSizeList, VariableSizeList } from 'react-window';
import AutoSizer from 'react-virtualized-auto-sizer';

// Simple fixed-height list
function UserList({ users }) {
  const Row = useCallback(({ index, style }) => (
    // ⚠️ MUST apply the style prop! It contains absolute positioning
    <div style={style}>
      <UserCard user={users[index]} />
    </div>
  ), [users]);

  return (
    <AutoSizer>
      {({ height, width }) => (
        <FixedSizeList
          height={height}       // Viewport height
          width={width}
          itemCount={users.length}
          itemSize={72}         // Row height in px
          overscanCount={5}     // Render 5 extra rows above/below viewport
        >
          {Row}
        </FixedSizeList>
      )}
    </AutoSizer>
  );
}

// ─────────────────────────────────────────────
// Variable height rows (e.g., chat messages)
function ChatWindow({ messages }) {
  const getItemSize = useCallback((index) => {
    // Estimate based on content length — or measure after render
    const msg = messages[index];
    return Math.max(60, Math.ceil(msg.text.length / 50) * 20 + 40);
  }, [messages]);

  return (
    <VariableSizeList
      height={600}
      width="100%"
      itemCount={messages.length}
      itemSize={getItemSize}
      estimatedItemSize={80}
    >
      {({ index, style }) => (
        <div style={style}>
          <ChatMessage message={messages[index]} />
        </div>
      )}
    </VariableSizeList>
  );
}

// ─────────────────────────────────────────────
// For tables: TanStack Virtual (formerly react-virtual)
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualTable({ rows }) {
  const parentRef = useRef(null);

  const virtualizer = useVirtualizer({
    count: rows.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 40,
    overscan: 10,
  });

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      {/* Total height to maintain correct scrollbar */}
      <div style={{ height: virtualizer.getTotalSize() + 'px', position: 'relative' }}>
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={rows[virtualRow.index].id}
            style={{
              position: 'absolute',
              top: virtualRow.start + 'px', // ← Absolute positioning key
              width: '100%',
            }}
          >
            <TableRow row={rows[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## Q18. Code Splitting & Lazy Loading

### Short Answer
Code splitting defers loading JavaScript until needed. Use `React.lazy` + `Suspense` for component-level splitting, and dynamic `import()` for library-level splitting. Route-based splitting is the highest ROI.

### Code Example
```jsx
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

// ─────────────────────────────────────────────
// Route-based splitting (highest impact)
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Reports = lazy(() => import('./pages/Reports'));
const Settings = lazy(() => import('./pages/Settings'));

// Named export workaround (lazy only works with default exports)
const ReportChart = lazy(() =>
  import('./components/ReportChart').then(module => ({
    default: module.ReportChart, // Map named export to default
  }))
);

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/reports" element={<Reports />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}

// ─────────────────────────────────────────────
// Feature-based splitting (heavy features)
function AnalyticsButton({ onOpen }) {
  const [show, setShow] = useState(false);
  const AnalyticsModal = lazy(() => import('./AnalyticsModal'));

  return (
    <>
      <button onClick={() => setShow(true)}>View Analytics</button>
      {show && (
        <Suspense fallback={<Spinner />}>
          <AnalyticsModal onClose={() => setShow(false)} />
        </Suspense>
      )}
    </>
  );
}

// ─────────────────────────────────────────────
// Preloading: Start loading before user clicks
function MenuLink({ to, label, preloadComponent }) {
  const handleMouseEnter = useCallback(() => {
    // Hover → preload → by the time they click, chunk is loaded
    import('./pages/' + preloadComponent);
  }, [preloadComponent]);

  return (
    <Link to={to} onMouseEnter={handleMouseEnter}>
      {label}
    </Link>
  );
}
```

---

## Q19. Debugging Performance — React DevTools Profiler

### Short Answer
The React DevTools Profiler records render timings, commit durations, and component render counts. Use the flame chart to find components with long render times, and the ranked chart to find the most expensive renders.

### Deep Explanation

**How to read the Profiler:**
```
Flame Chart: horizontal = time, vertical = call depth
• Wide bars = slow component
• Grey bars = component did NOT re-render (used memo)
• Colored bars = DID re-render (color = % of total commit time)

Ranked Chart: components sorted by self render time
• Top of list = most expensive
• Click to see why it rendered (which props/state changed)

Commit Timeline: Each bar = one commit (batch of DOM updates)
• Tall bar = slow commit → performance issue
• Click a bar → see all components that rendered in that commit
```

### Code Example — Identifying Wasted Renders
```jsx
// After profiling, you find <ProductCard> renders 200 times when only 5 change
// Root cause: parent re-renders (notifications), ProductCard has no React.memo

// Debugging checklist:
// 1. Open DevTools → Profiler tab → Record → Interact → Stop
// 2. Look at "Why did this render?" for suspicious components
// 3. "Props changed: { onClick: fn → fn }" → need useCallback
// 4. "Context changed" → need context splitting
// 5. "Parent rendered" with no prop changes → need React.memo

// Programmatic profiling for production:
import { Profiler } from 'react';

function onRenderCallback(
  id,           // Component name
  phase,        // "mount" | "update" | "nested-update"
  actualDuration,  // Time spent rendering (ms)
  baseDuration,    // Estimated time without memoization (ms)
  startTime,
  commitTime
) {
  if (actualDuration > 16) { // Slower than 1 frame
    analytics.track('slow_render', {
      component: id,
      phase,
      duration: actualDuration,
    });
  }
}

<Profiler id="Dashboard" onRender={onRenderCallback}>
  <Dashboard />
</Profiler>
```

---

## Q20. Real-World Scenario: Slow Dashboard with 1M Users

### Deep Explanation & Solution

```
SYMPTOMS:
• Dashboard loads in 8 seconds
• First interaction lags 2-3 seconds
• Memory grows to 500MB after 10 minutes of use
• Tab freezes when applying filters

DIAGNOSIS APPROACH:
1. Lighthouse audit → FCP, LCP, TTI, TBT
2. Chrome DevTools Performance tab → Main thread flame chart
3. React Profiler → Which components are slow
4. Network tab → Bundle sizes, API timing
5. Memory tab → Heap snapshots for leak detection
```

```jsx
// IDENTIFIED ISSUES AND FIXES:

// Issue 1: 500KB bundle on initial load
// Fix: Route-based splitting (already shown above)
// Impact: Reduces initial bundle from 500KB → 120KB

// Issue 2: 10,000 rows rendered in a table
const DataTable = () => (
  // ❌ Before: render all 10k rows
  data.map(row => <TableRow key={row.id} row={row} />)
);
// ✅ After: virtualize
// Using react-window FixedSizeList (shown above)
// Impact: DOM nodes: 10,000 → ~20, memory: -400MB

// Issue 3: Entire store subscribed to WebSocket updates
// ❌ Before: Every WS message re-renders entire dashboard
// ✅ After: Granular subscriptions via useSyncExternalStore
function useMetric(metricId) {
  return useSyncExternalStore(
    (cb) => store.subscribe(metricId, cb), // Subscribe to ONE metric
    () => store.getMetric(metricId),        // Snapshot of ONE metric
  );
}

// Issue 4: Heavy chart redraws on filter change
// ❌ Before: Chart re-renders with every keystroke in filter input
// ✅ After: Defer with useDeferredValue
function FilteredDashboard() {
  const [rawFilter, setRawFilter] = useState('');
  const deferredFilter = useDeferredValue(rawFilter);
  
  return (
    <>
      <FilterInput value={rawFilter} onChange={setRawFilter} />
      {/* Chart uses deferred value — input feels instant */}
      <ExpensiveChart filter={deferredFilter} />
    </>
  );
}

// Issue 5: Memory leak from uncleared subscriptions
// Audit: Check useEffect cleanups
// Use Chrome Memory profiler: take heap snapshot → GC → take again
// Objects that survive GC = potential leaks
// Common culprits: event listeners, timers, WebSocket connections, observables
```

---

## Q21. Input Lag Issue

```jsx
// PROBLEM: Search input has 200ms lag due to heavy list re-rendering on every keystroke

// SOLUTION 1: Separate input state from search state (concurrent approach)
function SearchableList({ items }) {
  const [inputValue, setInputValue] = useState('');
  const deferredInput = useDeferredValue(inputValue);

  const filteredItems = useMemo(
    () => items.filter(item => item.name.toLowerCase().includes(deferredInput.toLowerCase())),
    [items, deferredInput]
  );

  return (
    <>
      {/* Input state updates synchronously → no lag */}
      <input value={inputValue} onChange={e => setInputValue(e.target.value)} />
      {/* List updates asynchronously → yields to input */}
      <div style={{ opacity: inputValue !== deferredInput ? 0.7 : 1 }}>
        {filteredItems.map(item => <Item key={item.id} item={item} />)}
      </div>
    </>
  );
}

// SOLUTION 2: Debounce (simpler, works in older React)
function SearchableList({ items }) {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 150); // Hook from earlier

  // Only filters when user stops typing for 150ms
  const filteredItems = useMemo(
    () => items.filter(item => item.name.includes(debouncedQuery)),
    [items, debouncedQuery]
  );
}
```

---

# SECTION 4: COMPONENT ARCHITECTURE {#section-4}

---

## Q22. Scalable Folder Structure

### Short Answer
Feature-based structure scales better than type-based. Each feature owns its components, hooks, tests, and types, reducing cross-feature coupling and making code ownership clear.

```
src/
├── app/                    # App-level setup (routing, providers, global styles)
│   ├── App.tsx
│   ├── routes.tsx
│   └── providers/
│       ├── QueryProvider.tsx
│       ├── ThemeProvider.tsx
│       └── index.tsx
│
├── features/               # Feature modules (vertical slices)
│   ├── auth/
│   │   ├── components/     # UI components specific to auth
│   │   │   ├── LoginForm.tsx
│   │   │   └── PermissionGate.tsx
│   │   ├── hooks/
│   │   │   ├── useAuth.ts
│   │   │   └── usePermissions.ts
│   │   ├── api/
│   │   │   └── auth.api.ts  # API calls (React Query mutations/queries)
│   │   ├── store/
│   │   │   └── auth.store.ts # Zustand slice or Redux slice
│   │   ├── types/
│   │   │   └── auth.types.ts
│   │   └── index.ts         # Public API of the feature (barrel exports)
│   │
│   ├── dashboard/
│   ├── products/
│   └── orders/
│
├── shared/                 # Truly shared, no feature deps
│   ├── components/         # Design system components
│   │   ├── Button/
│   │   │   ├── Button.tsx
│   │   │   ├── Button.test.tsx
│   │   │   └── index.ts
│   │   └── Table/
│   ├── hooks/              # Generic hooks (useDebounce, useMediaQuery)
│   ├── utils/              # Pure functions
│   ├── types/              # Shared TypeScript types
│   └── constants/
│
├── lib/                    # Third-party integrations / configurations
│   ├── queryClient.ts      # React Query client setup
│   ├── axios.ts            # Axios instance with interceptors
│   └── sentry.ts
│
└── test/                   # Test utilities, mocks, factories
    ├── mocks/
    ├── factories/
    └── utils.tsx            # Custom render with providers
```

---

## Q23. Compound Components Pattern

### Short Answer
Compound components share implicit state through context, letting parent components control composition while child components handle rendering details. Used by `<select>/<option>`, `<table>/<tr>`, Radix UI, Headless UI.

### Code Example
```jsx
// Building a compound Select component (like Radix UI's approach)

const SelectContext = createContext(null);

function Select({ value, onChange, children }) {
  const [isOpen, setIsOpen] = useState(false);

  const contextValue = useMemo(() => ({
    value,
    onChange: (newValue) => {
      onChange(newValue);
      setIsOpen(false);
    },
    isOpen,
    setIsOpen,
  }), [value, onChange, isOpen]);

  return (
    <SelectContext.Provider value={contextValue}>
      <div className="select-container" style={{ position: 'relative' }}>
        {children}
      </div>
    </SelectContext.Provider>
  );
}

function SelectTrigger({ children }) {
  const { value, isOpen, setIsOpen } = useContext(SelectContext);
  return (
    <button onClick={() => setIsOpen(o => !o)} aria-expanded={isOpen}>
      {children || value}
    </button>
  );
}

function SelectContent({ children }) {
  const { isOpen } = useContext(SelectContext);
  if (!isOpen) return null;
  return <ul role="listbox" className="select-menu">{children}</ul>;
}

function SelectItem({ value, children }) {
  const { value: selectedValue, onChange } = useContext(SelectContext);
  return (
    <li
      role="option"
      aria-selected={value === selectedValue}
      onClick={() => onChange(value)}
    >
      {children}
    </li>
  );
}

// Attach as static properties for discoverability
Select.Trigger = SelectTrigger;
Select.Content = SelectContent;
Select.Item = SelectItem;

// Usage — composition without prop drilling
<Select value={country} onChange={setCountry}>
  <Select.Trigger />
  <Select.Content>
    <Select.Item value="US">United States</Select.Item>
    <Select.Item value="IN">India</Select.Item>
    <Select.Item value="UK">United Kingdom</Select.Item>
  </Select.Content>
</Select>
```

---

## Q24. Headless Components (Renderless Pattern)

### Short Answer
Headless components provide behavior and state management with zero opinions on styling. They expose render props or hooks — consumers control 100% of the UI. Examples: Headless UI, Radix UI, React Table, Downshift.

### Code Example
```jsx
// Building a headless Accordion (behavior only, no styles)
function useAccordion({ allowMultiple = false } = {}) {
  const [openItems, setOpenItems] = useState(new Set());

  const toggle = useCallback((id) => {
    setOpenItems(prev => {
      const next = new Set(prev);
      if (next.has(id)) {
        next.delete(id);
      } else {
        if (!allowMultiple) next.clear();
        next.add(id);
      }
      return next;
    });
  }, [allowMultiple]);

  const isOpen = useCallback((id) => openItems.has(id), [openItems]);

  const getItemProps = useCallback((id) => ({
    isOpen: openItems.has(id),
    toggle: () => toggle(id),
  }), [openItems, toggle]);

  return { isOpen, toggle, getItemProps };
}

// Usage: Consumer controls ALL styling
function ProductFAQ({ faqs }) {
  const accordion = useAccordion({ allowMultiple: false });

  return (
    <div className="my-custom-accordion"> {/* Any styling */}
      {faqs.map(faq => {
        const { isOpen, toggle } = accordion.getItemProps(faq.id);
        return (
          <div key={faq.id} className={`faq-item ${isOpen ? 'open' : ''}`}>
            <button
              onClick={toggle}
              className="faq-trigger"
              aria-expanded={isOpen}
            >
              {faq.question}
              <ChevronIcon rotated={isOpen} />
            </button>
            {isOpen && (
              <div className="faq-content">{faq.answer}</div>
            )}
          </div>
        );
      })}
    </div>
  );
}
```

---

# SECTION 5: STATE MANAGEMENT {#section-5}

---

## Q25. Local vs Global vs Server State

### Short Answer
Local state (useState/useReducer) is for component-specific UI state. Global state is for shared application state. Server state is cache of remote data — best managed by React Query or SWR, NOT by Redux.

### Deep Explanation

```
State Classification:
─────────────────────────────────────────
LOCAL STATE (useState/useReducer)
  • Modal open/closed
  • Form input values
  • Tab selection
  • Hover/focus state
  Rule: If only ONE component needs it, keep it local

GLOBAL STATE (Zustand/Redux/Context)
  • Current user session
  • Theme/locale preferences
  • Shopping cart
  • Application permissions
  Rule: If MULTIPLE unrelated components need the SAME data
  Rule: If state needs to survive navigation (but not page refresh)

SERVER STATE (React Query/SWR)
  • List of products from API
  • User profile from DB
  • Dashboard metrics
  Rule: If data comes from / is persisted to a server
  Rule: Handles: loading, error, caching, refetching, invalidation
─────────────────────────────────────────
```

### Code Example — React Query vs Redux for server data
```jsx
// ❌ WRONG: Managing server state in Redux
// Requires: action creators, reducers, loading/error states, manual invalidation
const productsSlice = createSlice({
  name: 'products',
  initialState: { items: [], loading: false, error: null },
  reducers: {
    fetchStart: state => { state.loading = true; },
    fetchSuccess: (state, action) => {
      state.items = action.payload;
      state.loading = false;
    },
    fetchError: (state, action) => {
      state.error = action.payload;
      state.loading = false;
    },
  },
});
// 50+ lines of boilerplate + manual cache invalidation + no auto-refetch

// ✅ CORRECT: React Query handles server state
function ProductList() {
  const {
    data: products,
    isLoading,
    error,
    refetch,
  } = useQuery({
    queryKey: ['products'],           // Cache key
    queryFn: () => api.getProducts(), // Fetch function
    staleTime: 5 * 60 * 1000,         // Data fresh for 5 min
    gcTime: 10 * 60 * 1000,          // Keep in cache 10 min after unused
    retry: 3,                          // Retry failed requests
    refetchOnWindowFocus: true,        // Refetch when user returns to tab
  });

  if (isLoading) return <Skeleton />;
  if (error) return <ErrorBoundary error={error} onRetry={refetch} />;
  return <List items={products} />;
}

// ─────────────────────────────────────────────
// Mutations with optimistic updates
function useUpdateProduct() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, updates }) => api.updateProduct(id, updates),
    
    // Optimistic update: show change immediately
    onMutate: async ({ id, updates }) => {
      await queryClient.cancelQueries({ queryKey: ['products'] });
      const previousProducts = queryClient.getQueryData(['products']);

      // Optimistically update the cache
      queryClient.setQueryData(['products'], old =>
        old.map(p => p.id === id ? { ...p, ...updates } : p)
      );

      return { previousProducts }; // Rollback data
    },

    // Rollback on error
    onError: (err, variables, context) => {
      queryClient.setQueryData(['products'], context.previousProducts);
      toast.error('Update failed, reverting...');
    },

    // Sync server state after success
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },
  });
}
```

---

## Q26. Redux vs Zustand vs Context

### Short Answer
Use Redux for large teams with complex shared state and time-travel debugging needs. Use Zustand for simpler global state with less boilerplate. Use Context for low-frequency, small data (theme, locale, auth).

### Comparison Table
| | Redux Toolkit | Zustand | Context + useReducer |
|---|---|---|---|
| Bundle size | ~12KB | ~1KB | 0 (built-in) |
| Boilerplate | Medium (RTK reduces it) | Minimal | Low |
| DevTools | Excellent (time-travel) | Good | Basic |
| Performance | Selector-based | Selector-based | Re-renders all consumers |
| Async | RTK Query / Thunks | Built-in | Manual |
| Learning curve | High | Low | Low |
| Best for | Large team, complex state | Medium apps | Theme/locale/auth |

### Code Example — Zustand (production pattern)
```jsx
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

// Slice pattern for Zustand (like Redux slices)
const createCartSlice = (set, get) => ({
  items: [],
  total: 0,

  addItem: (product) => set(state => {
    const existing = state.items.find(i => i.id === product.id);
    if (existing) {
      existing.quantity += 1;
    } else {
      state.items.push({ ...product, quantity: 1 });
    }
    state.total = state.items.reduce((sum, i) => sum + i.price * i.quantity, 0);
  }),

  removeItem: (id) => set(state => {
    state.items = state.items.filter(i => i.id !== id);
    state.total = state.items.reduce((sum, i) => sum + i.price * i.quantity, 0);
  }),
});

const createUserSlice = (set) => ({
  user: null,
  setUser: (user) => set({ user }),
  logout: () => set({ user: null }),
});

// Combine slices with middleware
const useStore = create(
  devtools(
    persist(
      immer((set, get) => ({
        ...createCartSlice(set, get),
        ...createUserSlice(set, get),
      })),
      {
        name: 'app-store',
        partialize: (state) => ({ user: state.user }), // Only persist user, not cart
      }
    )
  )
);

// Granular selectors — components only re-render for their slice
const useCartItems = () => useStore(state => state.items);
const useCartTotal = () => useStore(state => state.total);
const useUser = () => useStore(state => state.user);
```

---

# SECTION 6: RENDERING STRATEGIES {#section-6}

---

## Q27. CSR vs SSR vs SSG vs ISR

### Short Answer
CSR is client-only rendering (poor SEO, slow FCP). SSR renders on every request (good SEO, higher server cost). SSG renders at build time (best performance, stale for dynamic data). ISR revalidates static pages in the background.

### Comparison Table
| | CSR | SSR | SSG | ISR |
|---|---|---|---|---|
| Render time | On client | On request | At build | At build + revalidate |
| FCP | Slow | Fast | Fastest | Fast |
| SEO | Poor | Excellent | Excellent | Excellent |
| Infrastructure | CDN only | Server required | CDN only | CDN + revalidation |
| Dynamic data | ✅ Real-time | ✅ Real-time | ❌ Build-time only | ⚠️ Stale-while-revalidate |
| Personalized content | ✅ | ✅ | ❌ | ⚠️ (via client hydration) |
| Cost | Low | High | Lowest | Low |
| Best for | Dashboards, Apps | E-commerce, News | Marketing, Blog | Product pages |

### Code Example — Next.js rendering strategies
```jsx
// CSR: Fetch on client (dashboard, user-specific pages)
function Dashboard() {
  const { data } = useSWR('/api/metrics', fetcher);
  return <MetricsView data={data} />;
}

// SSR: Fetch on every request (personalized, real-time)
export async function getServerSideProps({ req, params }) {
  const session = await getSession(req);
  const data = await db.getPersonalizedFeed(session.userId);
  return { props: { data } };
}

// SSG: Build-time (blog posts, marketing pages)
export async function getStaticProps() {
  const posts = await cms.getAllPosts();
  return {
    props: { posts },
    revalidate: 3600, // ISR: regenerate after 1 hour if request comes in
  };
}

export async function getStaticPaths() {
  const posts = await cms.getPostSlugs();
  return {
    paths: posts.map(slug => ({ params: { slug } })),
    fallback: 'blocking', // ISR: generate unknown paths on first request
  };
}

// Next.js App Router (React Server Components)
// Server component — runs on server, no JS in bundle
async function ProductPage({ params }) {
  const product = await db.getProduct(params.id); // Direct DB access!
  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <AddToCartButton productId={product.id} /> {/* Client component */}
    </div>
  );
}
```

---

## Q28. Hydration Process & Mismatch Issues

### Short Answer
Hydration is React attaching event listeners to existing server-rendered HTML without re-creating DOM nodes. Mismatches (server HTML ≠ client render) cause React to re-render the entire subtree or warn in development.

### Deep Explanation

```
SSR Hydration Flow:
1. Server: ReactDOMServer.renderToString(<App />) → HTML string
2. Browser receives HTML → user sees content immediately (no JS needed)
3. React JS bundle downloads
4. ReactDOM.hydrateRoot(<App />) → React walks DOM, expects it matches
5. React attaches event handlers to existing DOM nodes
6. Interactivity restored

If mismatch:
• Development: Warning + React corrects it (re-renders)
• Production: Silently replaces server HTML with client render
```

### Code Example — Mismatch Sources & Fixes
```jsx
// ─────────────────────────────────────────────
// Cause 1: Date/time differences (server time ≠ client time)
// ❌ Bad: Different output on server vs client
function Timestamp() {
  return <time>{new Date().toLocaleDateString()}</time>;
  // Server renders "4/14/2026", client renders same — BUT timezone can differ!
}

// ✅ Fix: Suppress hydration on dynamic values
function Timestamp() {
  return (
    <time suppressHydrationWarning>
      {new Date().toLocaleDateString()}
    </time>
  );
  // React skips mismatch check for this element
}

// ─────────────────────────────────────────────
// Cause 2: Browser-only APIs used during render
// ❌ Bad: window is not available on server
function ThemeProvider({ children }) {
  const theme = window.matchMedia('(prefers-color-scheme: dark)').matches
    ? 'dark' : 'light';
  // ReferenceError on server!
}

// ✅ Fix: Use useEffect or check typeof
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light'); // Default value for SSR

  useEffect(() => {
    // Runs only on client, after hydration
    const darkMode = window.matchMedia('(prefers-color-scheme: dark)').matches;
    setTheme(darkMode ? 'dark' : 'light');
  }, []);

  return <ThemeContext.Provider value={theme}>{children}</ThemeContext.Provider>;
}

// ─────────────────────────────────────────────
// Cause 3: User-agent specific rendering
// ❌ Bad: Different content for mobile/desktop
function Layout() {
  const isMobile = navigator.userAgent.includes('Mobile'); // Server has no navigator
  return isMobile ? <MobileLayout /> : <DesktopLayout />;
}

// ✅ Fix: CSS media queries, or client-only detection
function Layout() {
  const [isMobile, setIsMobile] = useState(false); // SSR: assume desktop

  useEffect(() => {
    setIsMobile(window.innerWidth < 768);
  }, []);

  return isMobile ? <MobileLayout /> : <DesktopLayout />;
  // Initial paint = desktop (matches server) → then adjusts if mobile
}
```

---

# SECTION 7: SECURITY {#section-7}

---

## Q29. XSS Prevention in React

### Short Answer
React auto-escapes JSX values, preventing most XSS. The only entry point is `dangerouslySetInnerHTML`, which bypasses this protection and requires explicit sanitization.

### Code Example
```jsx
// React auto-escapes by default:
const userInput = '<script>alert("xss")</script>';
<div>{userInput}</div>
// Renders as text: &lt;script&gt;alert("xss")&lt;/script&gt; — SAFE

// ─────────────────────────────────────────────
// DANGEROUS: dangerouslySetInnerHTML without sanitization
// ❌ NEVER DO THIS with user content
<div dangerouslySetInnerHTML={{ __html: userBio }} />
// If userBio = '<img src=x onerror=stealCookies()>' → XSS attack

// ✅ Sanitize before rendering with DOMPurify
import DOMPurify from 'dompurify';

function UserBio({ html }) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
    ALLOWED_ATTR: ['href', 'title', 'target'],
    FORBID_ATTR: ['style', 'onerror', 'onclick'], // Extra caution
    ALLOW_DATA_ATTR: false,
  });

  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}

// ─────────────────────────────────────────────
// Subtle XSS: href injection
// ❌ Bad:
<a href={userProvidedUrl}>Click me</a>
// If url = 'javascript:stealCookies()' → XSS via protocol

// ✅ Fix: Validate URL protocol
function SafeLink({ href, children }) {
  const isSafe = href.startsWith('https://') || href.startsWith('http://');
  return (
    <a
      href={isSafe ? href : '#'}
      rel="noopener noreferrer" // Prevents window.opener access
      target="_blank"
    >
      {children}
    </a>
  );
}
```

---

## Q30. Token Storage — Cookies vs localStorage

### Short Answer
Store authentication tokens in `HttpOnly Secure SameSite=Strict` cookies — they're inaccessible to JavaScript (XSS-proof) and automatically sent with requests. `localStorage` is accessible to all scripts (XSS vulnerable).

### Comparison Table
| | HttpOnly Cookie | localStorage / sessionStorage |
|---|---|---|
| XSS access | ❌ Inaccessible to JS | ✅ Accessible to any script |
| CSRF | ⚠️ Vulnerable (mitigate with SameSite) | ❌ Not sent automatically, N/A |
| Expiry | Configurable by server | Manual, survives refresh (LS) |
| Size | 4KB | 5-10MB |
| Sent automatically | ✅ With every request | ❌ Must add header manually |
| Server access | ✅ Server can set/clear | ❌ Client only |
| Recommendation | ✅ For auth tokens | ❌ Never for tokens |

```javascript
// Server-side: set HttpOnly cookie (Node/Express)
res.cookie('auth_token', token, {
  httpOnly: true,    // Inaccessible to JavaScript
  secure: true,      // HTTPS only
  sameSite: 'strict', // No cross-site requests (CSRF protection)
  maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
  path: '/',
});

// Client: no JS access to the cookie — it's sent automatically
// For React SPA: use a CSRF token in a non-HttpOnly cookie for CSRF protection
// The auth token stays secure, CSRF token is readable by JS for form submissions
```

---

# SECTION 8: TESTING {#section-8}

---

## Q31. Unit vs Integration vs E2E — The Testing Trophy

### Short Answer
The Testing Trophy (Kent C. Dodds) recommends most coverage from integration tests (components with real data flow), fewer unit tests for utilities, and strategic E2E for critical paths. "Write tests that resemble how your software is used."

### Code Example — Testing at the right level
```jsx
// ─────────────────────────────────────────────
// Integration test (highest value for React)
// Tests component + hook + DOM interaction together
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { QueryClientProvider, QueryClient } from '@tanstack/react-query';
import { server } from '../test/mocks/server'; // MSW mock server
import { rest } from 'msw';
import ProductList from './ProductList';

function renderWithProviders(ui) {
  const queryClient = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  return render(
    <QueryClientProvider client={queryClient}>
      {ui}
    </QueryClientProvider>
  );
}

test('shows products and allows adding to cart', async () => {
  // Setup: MSW intercepts actual fetch call
  server.use(
    rest.get('/api/products', (req, res, ctx) =>
      res(ctx.json([
        { id: '1', name: 'Laptop', price: 999 },
        { id: '2', name: 'Phone', price: 699 },
      ]))
    )
  );

  renderWithProviders(<ProductList />);

  // Loading state
  expect(screen.getByRole('progressbar')).toBeInTheDocument();

  // Loaded state
  const laptopCard = await screen.findByText('Laptop');
  expect(laptopCard).toBeInTheDocument();
  expect(screen.getByText('$999')).toBeInTheDocument();

  // Interaction
  fireEvent.click(screen.getByRole('button', { name: /add laptop to cart/i }));
  expect(await screen.findByText(/1 item in cart/i)).toBeInTheDocument();
});

test('shows error state on fetch failure', async () => {
  server.use(
    rest.get('/api/products', (req, res, ctx) => res(ctx.status(500)))
  );

  renderWithProviders(<ProductList />);

  expect(await screen.findByText(/something went wrong/i)).toBeInTheDocument();
  expect(screen.getByRole('button', { name: /retry/i })).toBeInTheDocument();
});

// ─────────────────────────────────────────────
// Testing custom hooks with renderHook
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

test('useCounter increments and respects max', () => {
  const { result } = renderHook(() => useCounter({ initial: 0, max: 3 }));

  expect(result.current.count).toBe(0);

  act(() => result.current.increment());
  expect(result.current.count).toBe(1);

  act(() => { result.current.increment(); result.current.increment(); result.current.increment(); });
  expect(result.current.count).toBe(3); // Should not exceed max

  act(() => result.current.increment());
  expect(result.current.count).toBe(3); // Still 3
});
```

---

# SECTION 9: SYSTEM DESIGN — FRONTEND {#section-9}

---

## Q32. Micro-Frontend Architecture with Module Federation

### Short Answer
Micro-frontends split a large frontend into independently deployable apps per team/domain. Webpack Module Federation allows runtime sharing of components and libraries without full page navigation.

### Deep Explanation

```
Architecture Overview:
┌──────────────────────────────────────────┐
│            Shell Application             │
│  (App router, auth, global nav)          │
│                                          │
│  ┌─────────┐ ┌─────────┐ ┌───────────┐  │
│  │Products │ │ Orders  │ │Analytics  │  │
│  │  MFE    │ │   MFE   │ │   MFE     │  │
│  │(Team A) │ │(Team B) │ │ (Team C)  │  │
│  └─────────┘ └─────────┘ └───────────┘  │
└──────────────────────────────────────────┘
         ↕ Runtime integration via Module Federation

Each MFE:
• Independent repo, CI/CD pipeline
• Independent deployment
• Shared libraries via MF (React, design system)
• Own state, own routing
```

```javascript
// webpack.config.js for Products MFE (remote)
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'products',
      filename: 'remoteEntry.js',
      exposes: {
        './ProductList': './src/features/products/ProductList',
        './ProductDetail': './src/features/products/ProductDetail',
      },
      shared: {
        react: { singleton: true, requiredVersion: '^18.0.0' },
        'react-dom': { singleton: true, requiredVersion: '^18.0.0' },
        '@company/design-system': { singleton: true }, // Shared UI lib
      },
    }),
  ],
};

// webpack.config.js for Shell (host)
new ModuleFederationPlugin({
  name: 'shell',
  remotes: {
    products: 'products@https://products.company.com/remoteEntry.js',
    orders: 'orders@https://orders.company.com/remoteEntry.js',
  },
  shared: { react: { singleton: true }, 'react-dom': { singleton: true } },
});

// Shell usage — lazy load remote MFE
const ProductList = lazy(() => import('products/ProductList'));
const OrderList = lazy(() => import('orders/OrderList'));

function Shell() {
  return (
    <Router>
      <Routes>
        <Route path="/products/*" element={
          <Suspense fallback={<Spinner />}>
            <ProductList />
          </Suspense>
        } />
        <Route path="/orders/*" element={
          <Suspense fallback={<Spinner />}>
            <OrderList />
          </Suspense>
        } />
      </Routes>
    </Router>
  );
}
```

---

## Q33. High-Traffic Dashboard Design

### Short Answer
For 1M+ user dashboards: CDN-cache static assets, stream data via WebSockets or SSE, virtualize all lists, implement circuit breakers for API failures, and use service workers for offline capability.

### Deep Explanation
```
Architecture for 1M user dashboard:

Client:
├── Service Worker (cache, offline, background sync)
├── React Query (server state, smart caching)
├── Zustand (UI state)
├── Virtual lists (react-window)
├── Web Workers (heavy computation off main thread)
└── WebSocket (live data < 5s latency)

Edge:
├── CDN (static assets, SSG pages)
├── Edge caching (API responses where appropriate)
└── Rate limiting

Backend:
├── BFF (Backend for Frontend) — aggregate API
├── WebSocket server (live metrics)
└── Event streaming (Kafka → SSE/WS)
```

```jsx
// Web Worker for heavy computation (doesn't block main thread)
// worker.js
self.onmessage = (e) => {
  const { data, operation } = e.data;
  let result;
  
  if (operation === 'aggregate') {
    result = data.reduce((acc, item) => {
      // Heavy computation — sorting, grouping, statistics
      acc[item.category] = (acc[item.category] || 0) + item.value;
      return acc;
    }, {});
  }
  
  self.postMessage({ result });
};

// In React component:
function useWorkerAggregation(data) {
  const [result, setResult] = useState(null);
  const workerRef = useRef(null);

  useEffect(() => {
    workerRef.current = new Worker(new URL('./worker.js', import.meta.url));
    workerRef.current.onmessage = (e) => setResult(e.data.result);
    return () => workerRef.current.terminate();
  }, []);

  useEffect(() => {
    if (data && workerRef.current) {
      workerRef.current.postMessage({ data, operation: 'aggregate' });
    }
  }, [data]);

  return result;
}

// Circuit breaker for API failures
class CircuitBreaker {
  constructor({ threshold = 5, timeout = 60000 } = {}) {
    this.failures = 0;
    this.threshold = threshold;
    this.timeout = timeout;
    this.state = 'CLOSED'; // CLOSED = normal, OPEN = failing, HALF_OPEN = testing
    this.nextAttempt = null;
  }

  async call(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN — request rejected');
      }
      this.state = 'HALF_OPEN';
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  onSuccess() {
    this.failures = 0;
    this.state = 'CLOSED';
  }

  onFailure() {
    this.failures++;
    if (this.failures >= this.threshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
    }
  }
}
```

---

# SECTION 10: REAL-WORLD DEBUGGING {#section-10}

---

## Q34. Debugging Stale State

### Short Answer
Stale state comes from closures capturing old state values. Root causes: `useEffect` with empty or incorrect deps, event listeners attached once, or timer callbacks.

### Code Example — Production Debugging Playbook
```jsx
// SYMPTOM: User reports "my likes don't update" after leaving/returning to page

// ACTUAL BUG:
function LikeButton({ postId }) {
  const [liked, setLiked] = useState(false);
  const [likeCount, setLikeCount] = useState(0);

  useEffect(() => {
    // Socket connects once and listens for 'like' events
    socket.on('like', (data) => {
      // This closure captures likeCount = 0 forever
      if (data.postId === postId) {
        setLikeCount(likeCount + 1); // Always: 0 + 1 = 1, never increments past 1!
      }
    });

    return () => socket.off('like');
  }, []); // ← Empty deps = stale closure on likeCount

  // FIX:
  useEffect(() => {
    socket.on('like', (data) => {
      if (data.postId === postId) {
        setLikeCount(prev => prev + 1); // Functional update = no stale closure!
      }
    });
    return () => socket.off('like');
  }, [postId]); // Only re-subscribe if postId changes
}

// ─────────────────────────────────────────────
// Debugging Race Condition
// SYMPTOM: Navigating fast between user profiles shows wrong user data

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetchUser(userId).then(setUser);
    // If userId changes quickly: fetch(1), fetch(2) start
    // fetch(2) resolves first: shows user 2
    // fetch(1) resolves: OVERWRITES with user 1! ← Bug
  }, [userId]);

  // FIX: Cancel stale requests
  useEffect(() => {
    let cancelled = false;
    fetchUser(userId).then(user => {
      if (!cancelled) setUser(user);
    });
    return () => { cancelled = true; }; // Marks in-flight request as stale
  }, [userId]);
}

// ─────────────────────────────────────────────
// Memory Leak Detection
// SYMPTOM: App memory grows continuously, Chrome tab crashes after 2 hours

// Check for:
// 1. Event listeners not cleaned up
useEffect(() => {
  window.addEventListener('resize', handleResize);
  // ❌ Missing: return () => window.removeEventListener('resize', handleResize);
}, []);

// 2. Timers not cleared
useEffect(() => {
  const id = setInterval(pollData, 5000);
  // ❌ Missing: return () => clearInterval(id);
}, []);

// 3. Observables / subscriptions
useEffect(() => {
  const sub = store$.subscribe(setState);
  return () => sub.unsubscribe(); // ✅ Always cleanup RxJS subscriptions
}, []);

// 4. Detached DOM nodes holding references
// Use Chrome DevTools: Memory tab → Take heap snapshot
// Filter by "Detached" → find nodes that should have been GC'd
```

---

## Q35. Production Debugging Approach

### Short Answer
Production debugging requires: structured logging, error boundaries, source maps, distributed tracing, and reproducible error capture. Never console.log in production — use a logging service.

### Code Example
```jsx
// Error Boundary with Sentry integration
import * as Sentry from '@sentry/react';

class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, eventId: null };
  }

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    const eventId = Sentry.captureException(error, {
      contexts: { react: { componentStack: errorInfo.componentStack } },
      extra: {
        route: window.location.pathname,
        userId: getCurrentUserId(),
      },
    });
    this.setState({ eventId });
  }

  render() {
    if (this.state.hasError) {
      return (
        <div>
          <h2>Something went wrong</h2>
          <button onClick={() => Sentry.showReportDialog({ eventId: this.state.eventId })}>
            Report Problem
          </button>
          <button onClick={() => this.setState({ hasError: false })}>
            Try Again
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}

// Structured logging for production tracing
const logger = {
  info: (message, context = {}) => {
    if (process.env.NODE_ENV === 'production') {
      fetch('/api/logs', {
        method: 'POST',
        body: JSON.stringify({
          level: 'info',
          message,
          timestamp: new Date().toISOString(),
          sessionId: getSessionId(),
          userId: getUserId(),
          route: window.location.pathname,
          ...context,
        }),
      });
    } else {
      console.info(message, context);
    }
  },
  error: (message, error, context = {}) => {
    Sentry.captureException(error, { extra: context });
    // Also log to backend for correlation with server logs
  },
};

// Usage in components
function Checkout() {
  const handlePurchase = async () => {
    logger.info('checkout_started', { cartItems: cart.length, total });
    try {
      const order = await processPayment(cart);
      logger.info('checkout_completed', { orderId: order.id });
    } catch (error) {
      logger.error('checkout_failed', error, { cartItems: cart.length });
    }
  };
}
```

---

# QUICK-REFERENCE: 60+ Additional Questions

## React Core
- **Q36**: How does React handle errors in event handlers vs rendering? (Event handlers: not caught by error boundaries; use try/catch. Rendering: caught by error boundaries.)
- **Q37**: What is `React.StrictMode` and what does double-invocation catch? (Catches impure render functions, unsafe lifecycle methods, stale state in effects.)
- **Q38**: How do you implement server components? What can't they do? (No hooks, no browser APIs, no event handlers — but direct DB access, reduced client JS.)
- **Q39**: What is `useId` hook and when do you need it? (Stable, SSR-safe unique IDs for accessibility attributes like aria-labelledby.)
- **Q40**: How does `React.forwardRef` work internally? (Wraps component to pass ref through; component receives `(props, ref)` — ref is not in props.)

## Hooks Edge Cases
- **Q41**: What happens if `useEffect` dependency is an object created inline? (New reference every render → infinite re-run. Fix: useMemo or primitive deps.)
- **Q42**: Can you use async/await directly in `useEffect`? (No — async returns Promise, not cleanup fn. Create async fn inside effect and call it.)
- **Q43**: How does `useReducer` differ from `useState` for complex state? (Centralizes update logic, easier to test, handles related state transitions atomically.)
- **Q44**: What is `useImperativeHandle` and when is it appropriate? (Customizes what ref exposes from a child — for focus management, animation libraries, imperative APIs.)
- **Q45**: What is `useSyncExternalStore` and why was it added? (React 18 hook for subscribing to external stores without tearing in concurrent mode.)

## Performance
- **Q46**: What is "tearing" in concurrent React? (Different parts of UI showing state from different points in time due to concurrent rendering.)
- **Q47**: How do you profile memory usage in React apps? (Chrome DevTools Memory tab: heap snapshots, allocation profiler, GC tracking.)
- **Q48**: What is TTFB and how do React rendering strategies affect it? (Time to First Byte: SSR/SSG serve HTML immediately; CSR serves empty HTML + must wait for JS.)
- **Q49**: When does `React.memo` NOT help? (When props always change — inline objects, inline functions without useCallback, children JSX.)
- **Q50**: How do you handle 100MB CSV rendering in React? (Web Worker for parsing, virtual list for rendering, stream reading for loading.)

## Architecture
- **Q51**: How do you handle shared state between micro-frontends? (Custom events, shared store via import from Module Federation, URL state, postMessage.)
- **Q52**: What are render props and when are they preferable to hooks? (Function-as-child pattern for sharing rendering logic. Hooks preferred now, but render props for UI logic that needs JSX context.)
- **Q53**: How do you implement feature flags in React? (Context-based flag provider, hook `useFeatureFlag('flag-name')`, LaunchDarkly/Unleash integration.)
- **Q54**: How do you handle authentication token refresh in React? (Axios interceptor: on 401, refresh token, retry original request. React Query: onError with retry logic.)
- **Q55**: Design a multi-step form with React. (Controlled form state in parent/context, step validation per component, wizard pattern with progress state.)

## Testing
- **Q56**: How do you test a custom hook that uses fetch? (MSW for network mocking + renderHook from RTL.)
- **Q57**: What is the difference between `act()` and `waitFor()`? (`act` wraps synchronous state updates; `waitFor` polls until assertion passes for async changes.)
- **Q58**: How do you test React context consumers? (Wrap in Provider in custom render utility, test via component behavior not internal state.)
- **Q59**: How do you mock `useNavigate` from React Router? (Mock the entire `react-router-dom` module or use MemoryRouter in tests.)
- **Q60**: What is snapshot testing and when is it valuable? (Renders component to string/JSON, detects unintended changes. Value for UI regression; poor for TDD.)

## SSR / Next.js
- **Q61**: How do you handle third-party libraries that use `window` in SSR? (Dynamic imports with `{ ssr: false }` in Next.js, or `typeof window !== 'undefined'` guards.)
- **Q62**: What is partial prerendering (PPR) in Next.js 14+? (Static shell prerendered, dynamic "holes" streamed in — combines SSG speed + SSR dynamism.)
- **Q63**: How does streaming SSR work? (React sends HTML chunks as they're ready via `renderToPipeableStream`, browser progressively renders + hydrates.)
- **Q64**: What causes double fetching in SSR + client hydration? (Fetching in `getServerSideProps` + again in `useEffect`. Fix: hydrate client state from server data.)
- **Q65**: How do React Server Components avoid hydration cost? (RSC markup has no hydration script — only client components hydrate. RSC is server-only.)

## Security & Accessibility
- **Q66**: How do you prevent CSRF in a React SPA? (SameSite cookies + CSRF token in header, validated server-side.)
- **Q67**: How do you implement accessible modals in React? (Focus trap, `aria-modal`, `role="dialog"`, `aria-labelledby`, restore focus on close, Escape key.)
- **Q68**: What is Content Security Policy and how to configure it for React? (HTTP header limiting script/style sources; challenges with inline styles from CSS-in-JS.)
- **Q69**: How do you handle sensitive data in React components? (Never log/expose PII, mask in UI, clear from state on unmount, no sensitive data in URL params.)
- **Q70**: How do you audit a React app for accessibility? (axe-core, NVDA/JAWS testing, keyboard-only navigation audit, color contrast checking.)

## Advanced Patterns
- **Q71**: What is the Observer pattern in React and how do you implement it? (Custom event system or rxjs + useSyncExternalStore for reactive state outside React.)
- **Q72**: How do you implement undo/redo in React? (useReducer with past/future stacks, immer-style patches, or Zustand temporal middleware.)
- **Q73**: What is Suspense data fetching and how does it work? (Component throws a Promise; Suspense catches it and shows fallback until Promise resolves.)
- **Q74**: How do you implement infinite scroll? (IntersectionObserver on sentinel element + React Query's `useInfiniteQuery` for automatic page fetching.)
- **Q75**: How do you build a drag-and-drop list in React? (dnd-kit for accessible, touch-friendly DnD; react-beautiful-dnd for simpler cases; avoid html5-dnd API.)

## State Management Deep Dives
- **Q76**: What is Redux Toolkit's `createEntityAdapter`? (Normalized state management with CRUD operations and selectors for entity collections.)
- **Q77**: How does Recoil differ from Zustand? (Recoil: atom/selector graph, fine-grained subscriptions, React-specific. Zustand: simpler, works outside React.)
- **Q78**: What is Jotai and when would you use it over Zustand? (Atomic model, bottom-up vs Zustand's top-down. Better for interdependent derived state.)
- **Q79**: How do you handle optimistic updates with rollback in Redux? (Dispatch optimistic action → API call → on failure dispatch rollback action with error.)
- **Q80**: What is the difference between `selector` in Redux vs computed in Zustand? (Redux selector: pure fn with Reselect memoization. Zustand: derive in component or via getState.)

## Build & Infrastructure
- **Q81**: How do you reduce React bundle size? (Code splitting, tree shaking, analyze with `webpack-bundle-analyzer`, avoid large deps like moment.js.)
- **Q82**: What is `React.lazy` + `Suspense` vs loadable-components? (React.lazy: client-only, no SSR. loadable-components: SSR-compatible, more features.)
- **Q83**: How do you set up a React monorepo? (Turborepo or Nx: shared packages, unified build pipeline, workspace-aware tooling.)
- **Q84**: What are the build-time optimizations in Vite vs Webpack? (Vite: esbuild for dev, rollup for prod, no-bundle dev server. Webpack: more ecosystem, better optimization control.)
- **Q85**: How do you configure chunk splitting in Webpack? (splitChunks: vendor chunk, dynamic import per route, maximum asset size limits.)

## Tricky Edge Cases
- **Q86**: What happens when you render a component both as a route element and directly? (Two separate instances with independent state — no shared state unless via context/store.)
- **Q87**: Why might `console.log(state)` inside render show old value after setState? (State is immutable per render; console.log captures closure, not future state.)
- **Q88**: What is `key` prop on the parent element vs the children? (Key on parent — moving/remounting parent. Key on children — React tracks child identity separately.)
- **Q89**: How does React handle null/undefined/false in JSX? (They render nothing — valid as children. But `0` renders! `{count && <Comp />}` → renders `0` when count=0.)
- **Q90**: What is the event delegation model in React 17+ vs 16? (React 17+: events attached to root container, not document. Fixes issues with multiple React instances.)

## System Design Scenarios
- **Q91**: Design a real-time collaborative editor (like Google Docs) in React. (WebSocket + OT/CRDT for conflict resolution, useReducer for doc state, cursor broadcast, debounce saves.)
- **Q92**: Design a notification system for a React app. (WebSocket/SSE for real-time push, notification queue in Zustand, toast UI component, persistence in React Query cache.)
- **Q93**: How would you implement a React design system? (Headless primitives + styled layer, Storybook, component tokens, a11y built-in, versioned packages.)
- **Q94**: How do you handle A/B testing in React? (Feature flag hook + analytics tracking, SSR-safe cookie-based assignment, gradual rollout logic.)
- **Q95**: Design the frontend for a stock trading dashboard. (WebSocket for tick data, canvas/WebGL for charts, Web Workers for indicators, virtual list for order book.)

## React 18 / Future
- **Q96**: What is `useOptimistic` hook in React 19? (Optimistically update UI with temporary state while async action is in flight, auto-reverts on error.)
- **Q97**: What are React Actions (React 19)? (Async functions that handle form submissions, loading, and error state — replaces manual useTransition in forms.)
- **Q98**: What is `use()` hook in React 19? (Reads context and promises anywhere in a component, including inside conditions — unlike useContext.)
- **Q99**: How do React Server Components interact with Client Components? (Server components can render client components; client components cannot render server components; data flows down only.)
- **Q100**: What is the React Compiler (formerly React Forget)? (Babel transform that auto-inserts useMemo/useCallback — eliminates manual memoization for most cases.)

## Advanced Debugging
- **Q101**: How do you debug a React app in production without source maps? (Error IDs in Sentry → link to source map stored in CI; React error boundary captures stack traces.)
- **Q102**: What causes "Maximum update depth exceeded"? (Circular setState loop: useEffect or render calling setState unconditionally. Use condition guards.)
- **Q103**: How do you debug context causing unexpected re-renders? (React DevTools → Profiler → "Why did this render?" → shows "Context changed". Split context to fix.)
- **Q104**: What causes hydration mismatch in a Next.js app? (Server/client render differences: browser extensions injecting HTML, random values, dates, window-dependent logic.)
- **Q105**: How do you find and fix a "Cannot update component while rendering a different component" warning? (State update during render of another component. Move setState to useEffect or event handler.)

## Architecture Trade-offs
- **Q106**: Monolith React app vs Micro-frontends — when to choose each? (Monolith: small team, fast iteration, easier debugging. MFE: multiple teams, independent deployment needs, org > 100 engineers.)
- **Q107**: When would you choose React over Vue/Svelte for a new project? (React: large ecosystem, talent pool, concurrent features. Vue: gentler learning curve. Svelte: smallest bundle, compile-time.)
- **Q108**: How do you handle backward compatibility in a shared component library? (Semantic versioning, deprecation warnings, codemod scripts for breaking changes, change detection in CI.)
- **Q109**: When is Redux still the right choice in 2025? (Audit history/time-travel debugging, very large team needing strict patterns, existing Redux codebase with RTK.)
- **Q110**: How do you manage breaking changes between major React versions? (React codemods, gradual migration with StrictMode to surface warnings, version pinning in component library.)

## Senior-Only Topics
- **Q111**: Explain Algebraic Effects and how Suspense is related. (Suspense is an approximation — throwing Promises as "effects" with Suspense as the "handler".)
- **Q112**: How does React's scheduler prioritize using MessageChannel? (Yields to browser between fiber processing by scheduling continuation via `MessageChannel.port1.postMessage`.)
- **Q113**: What is the "work loop" in React's Fiber reconciler? (`workLoopSync` / `workLoopConcurrent` — the loop that calls `performUnitOfWork` on each fiber until no more work or time expires.)
- **Q114**: How does React implement error boundaries using `getDerivedStateFromError`? (Thrown error propagates up fiber tree; React sets error state on nearest class component with `getDerivedStateFromError`.)
- **Q115**: What are Lanes in React's scheduler and how do they handle priority inversion? (Bitmask lanes assigned per update; higher-priority lanes always processed first; EntangledLanes ensure related updates batch together.)
- **Q116**: How does React handle Suspense in a concurrent tree — what is "retrying"? (React keeps rendering the suspended subtree until Promise resolves, then replays from root of Suspense boundary.)
- **Q117**: What is `ReactDOM.flushSync` at the implementation level? (Forces scheduler to synchronously flush all pending work inside the callback — bypasses concurrent scheduling.)
- **Q118**: How does the Context API avoid prop drilling without Zustand? (Tree-position-based subscription — any component in tree can access context without prop threading. Performance cost: all consumers re-render.)
- **Q119**: What is "interleaved updates" in React 18 and how does it differ from batching? (Concurrent updates from different sources interleaved by scheduler priority — higher priority updates jump ahead of in-progress transitions.)
- **Q120**: How would you design a React rendering pipeline for a game engine UI? (60fps requires: no allocation in render loop, canvas for WebGL, Web Workers for physics, React only for UI overlay with strict memoization.)

---

# COMPARISON TABLES REFERENCE

## Hook Decision Matrix
| Need | Hook |
|---|---|
| UI state (toggle, counter) | `useState` |
| Complex state transitions | `useReducer` |
| DOM reference | `useRef` |
| Instance variable (no re-render) | `useRef` |
| Side effect after render | `useEffect` |
| DOM mutation before paint | `useLayoutEffect` |
| Computed value (expensive) | `useMemo` |
| Stable function reference | `useCallback` |
| Shared app state (read) | `useContext` |
| External store subscription | `useSyncExternalStore` |
| Deferred low-priority update | `useDeferredValue` or `useTransition` |
| Stable ID (SSR-safe) | `useId` |

## State Management Decision Matrix
| Scenario | Tool |
|---|---|
| Component-local UI state | `useState` / `useReducer` |
| Shared UI state (2-3 levels) | `useContext` |
| Server data (fetch, cache, sync) | React Query / SWR |
| Complex global UI state | Zustand |
| Large team, strict patterns | Redux Toolkit |
| Fine-grained atom subscriptions | Jotai / Recoil |
| Form state | React Hook Form |
| URL state | React Router `useSearchParams` |

## When to Use Each Rendering Strategy
| Content Type | Strategy |
|---|---|
| Marketing pages, blog posts | SSG |
| Product catalog (updates hourly) | ISR |
| User dashboard (personalized) | SSR or CSR |
| Admin panels, internal tools | CSR |
| E-commerce product pages | ISR |
| Social feed (real-time) | CSR + WebSocket |
| Documentation | SSG |
| Authentication flows | SSR (for redirect logic) |

---

*End of Handbook — 120+ Questions Covered*
*For each question in production interviews, focus on: the trade-off, the production implication, and at least one concrete example from your own experience.*
