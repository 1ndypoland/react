# React Mastery — Learning Plan & Repo Guide

> Prepared for Przemyslaw on 2026-05-06.  
> Goals: become a better programmer · master React · understand how modern JS libraries are built · contribute to open source.

---

## Table of Contents

1. [Phase 1 — JS Foundations via React Source](#1-phase-1)
2. [Phase 2 — Reading the Full Source](#2-phase-2)
3. [Phase 3 — Internalize Patterns](#3-phase-3)
4. [Phase 4 — Build React From Scratch](#4-phase-4)
5. [Repo Map — What Is Where and Why](#5-repo-map)
6. [Contribution Starter Guide](#6-contribution-guide)

---

## 1. Phase 1 — JS Foundations via React Source

No abstract exercises. Each JS concept is taught through the exact React code where it lives.
Work through these in order — each one unlocks the next.

---

### Module 1 — Closures: how `setState` knows which component it belongs to

**The concept:** a function captures variables from the scope where it was *created*, not where it is *called*.

**Open this file:**
```
packages/react-reconciler/src/ReactFiberHooks.js
```

**Read lines 1923–1936 (`mountState`):**
```js
const dispatch = dispatchSetState.bind(null, currentlyRenderingFiber, queue);
queue.dispatch = dispatch;
return [hook.memoizedState, dispatch];
```
The `setState` function you get back from `useState` is `dispatchSetState` partially applied with the specific `fiber` and `queue` for *this component instance*. It doesn't look up the component at call time — the closure carries it.

**Then read lines 3599–3628 (`dispatchSetState`):**
The function signature is `dispatchSetState(fiber, queue, action)`. When *you* call `setState(42)`, you only pass `action`. `fiber` and `queue` come from the closure baked in at mount time.

**What to notice:**
- `dispatch` is recreated on every render but always closes over the *same* `queue` object — that's why `setState` has a stable reference.
- The `bind(null, fiber, queue)` is not React magic; it is plain JS partial application.

**Mini test:** open your browser console and write a function that uses `.bind` to close over a counter. Then explain `dispatchSetState` in your own words.

---

### Module 2 — Linked Lists: why hooks must be called in the same order

**The concept:** a singly linked list where each node has `{ data, next }`. You walk it by following `next` until `null`.

**Open this file:**
```
packages/react-reconciler/src/ReactFiberHooks.js
```

**Read lines 980–998 (`mountWorkInProgressHook`):**
```js
const hook: Hook = {
  memoizedState: null,
  baseState: null,
  baseQueue: null,
  queue: null,
  next: null,        // ← this is the linked list pointer
};

if (workInProgressHook === null) {
  // First hook — becomes the head
  currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
} else {
  // Append to end
  workInProgressHook = workInProgressHook.next = hook;
}
```
Every time you call a hook (`useState`, `useEffect`, etc.), React runs this function and appends a new node. The list lives at `fiber.memoizedState`. First hook = head. Third hook = third node.

**Then read lines 1001–1060 (`updateWorkInProgressHook`):**
On re-render, React walks this same list from the head, reading one node per hook call — `nextCurrentHook = currentHook.next`. It never resets; it just advances the pointer.

**What to notice:**
- If you add a hook call inside an `if`, the list grows on the first render but not on the second. The pointer desynchronizes and every subsequent hook reads the *wrong* node. This is the entire mechanical reason for the Rules of Hooks.
- `fiber.memoizedState` is the linked list head, not an array. There is no index. Only order.

**Mini test:** draw the linked list for a component with `useState`, `useEffect`, `useMemo`. What does the list look like on first render vs. second?

---

### Module 3 — Bitwise Operations: React's priority system

**The concept:** `|` (OR) combines flags, `&` (AND) checks if a flag is set, `~` (NOT) clears a flag. All O(1), no loops.

**Open this file:**
```
packages/react-reconciler/src/ReactFiberLane.js
```

**Read lines 45–56 (the priority constants):**
```js
export const SyncLane:             Lane = 0b0000000000000000000000000000010;
export const InputContinuousLane:  Lane = 0b0000000000000000000000000001000;
export const DefaultLane:          Lane = 0b0000000000000000000000000100000;
// TransitionLane1–14: bits 8–21
// RetryLanes: bits 22–26
// IdleLane:   bit 30
// OffscreenLane: bit 31
```
Each lane is a single bit. Higher urgency = lower bit position. React can represent "I have sync work AND a transition pending" as a single number: `SyncLane | TransitionLane1`.

**Then open:**
```
packages/react-reconciler/src/ReactFiberFlags.js
```
**Read lines 18–40.** Same idea for effect types: `Placement`, `Update`, `ChildDeletion`, `Passive`, `Ref` are all single bits. A fiber's `flags` field is a bitmask of all the work that needs to happen to it.

**What to notice:**
- `fiber.lanes & SyncLane !== 0` means "does this fiber have sync work?". One CPU instruction.
- `subtreeFlags & (Passive | Update)` tells React whether it even needs to walk into a subtree. If the result is 0, the entire subtree is skipped during commit.
- This is why React can traverse millions of nodes fast. It skips entire branches with a single bitwise AND.

**Mini test:** in a Node REPL:
```js
const Placement = 0b010;
const Update    = 0b100;
const flags = Placement | Update;     // set both
console.log((flags & Placement) !== 0); // true — Placement is set
console.log((flags & 0b001) !== 0);     // false — some other flag is NOT set
flags & ~Placement                      // clear Placement
```

---

### Module 4 — Tree Traversal (DFS without recursion): the fiber work loop

**The concept:** depth-first traversal iteratively, using an explicit "current node" pointer instead of the call stack. Go deep (child), then across (sibling), then up (return/parent).

**Open this file:**
```
packages/react-reconciler/src/ReactFiberWorkLoop.js
```

**Read lines 2982–3040 (`workLoopConcurrentByScheduler` + `performUnitOfWork`):**
```js
function workLoopConcurrentByScheduler() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```
`workInProgress` is the "current node" pointer. `performUnitOfWork` calls `beginWork` on the current fiber. If `beginWork` returns a child, that child becomes `workInProgress`. If there is no child, `completeUnitOfWork` is called, which walks to the sibling, or if there's no sibling, back up to the parent.

**Draw this out:**
```
        A
       / \
      B   C
     / \
    D   E

Walk order: A → B → D → (complete D) → E → (complete E) → (complete B) → C → (complete C) → (complete A)
```
`child` pointer goes down. `sibling` pointer goes across. `return` pointer goes up.

**What to notice:**
- Recursive DFS locks the call stack — you cannot pause it midway. Iterative DFS with a pointer can be paused at any point just by returning from the `while` loop. This is the entire mechanical basis for Concurrent React's interruptibility.
- `workInProgress` is a module-level variable. Pausing = returning from `workLoopConcurrentByScheduler` without clearing it. Resuming = calling `workLoopConcurrentByScheduler` again.

---

### Module 5 — Strategy / Dispatcher Pattern: how the same hook does different things on mount vs. update

**The concept:** a shared mutable slot holds the "current implementation". Swap the object in the slot to change behavior without changing callers.

**Open this file:**
```
packages/react/src/ReactHooks.js
```

**Read lines 24–44 (`resolveDispatcher` and `useContext`):**
```js
function resolveDispatcher() {
  const dispatcher = ReactSharedInternals.H;
  // ...
  return dispatcher;
}

export function useState(initialState) {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}
```
`useState` in the public API does nothing itself — it just asks "who is the current dispatcher?" and delegates.

**Now grep for `HooksDispatcherOnMount` in `ReactFiberHooks.js`:**
```bash
grep -n "HooksDispatcherOnMount\b" packages/react-reconciler/src/ReactFiberHooks.js | head -5
```
You'll find an object where `useState: mountState`. On first render, React installs this dispatcher. On update, it installs `HooksDispatcherOnUpdate` where `useState: updateState`. The hook code path changes entirely without touching the public API.

**What to notice:**
- `ReactSharedInternals.H` is null outside of rendering. `resolveDispatcher` will return null, causing your hook call to crash with the "Invalid hook call" error. That error message is not an accident — it is the dispatcher being null.
- This pattern (a mutable slot + swappable implementations) appears in virtually every large JS library. React uses it here, for the async dispatcher (`.A`), and for server vs. client rendering.

---

### Module 6 — Priority Queue (Min-Heap): how React orders scheduled work

**The concept:** always gives you the *most urgent* item in O(1), inserts in O(log n). Internally a binary tree stored as an array.

**Open this file (it is tiny — ~100 lines):**
```
packages/scheduler/src/SchedulerMinHeap.js
```

**Read the whole file.** Key lines:
- `push` (line 17): appends to the array, then `siftUp` bubbles it to the right position
- `pop` (line 27): removes the root (most urgent), puts the last element at root, then `siftDown` sinks it
- `peek` (line 13): reads root without removing — O(1)
- Comparison (line 93): `a.sortIndex - b.sortIndex` — smallest sortIndex = most urgent = closest to top

**How it connects:** `sortIndex` in the scheduler is `expirationTime`. A task that expires sooner has a smaller number, floats to the top, and runs first. This heap is why a `startTransition` update never blocks a button click — they literally have different sortIndexes in the heap.

**Mini test:** implement this from memory. It should be ~30 lines. If you can do it, you understand it.

---

### Module 7 — Cooperative Multitasking: yielding to the browser without threads

**The concept:** voluntarily pause long work every ~5ms to let the browser paint and handle user input. No threads, no Workers — just "I'll give control back and ask to be resumed."

**Open this file:**
```
packages/scheduler/src/forks/Scheduler.js
```

**Read lines 447–460 (`shouldYieldToHost`):**
```js
function shouldYieldToHost(): boolean {
  const timeElapsed = getCurrentTime() - startTime;
  if (timeElapsed < frameInterval) {
    return false;   // still within the budget, keep going
  }
  return true;      // over budget, yield
}
```
`frameInterval` defaults to 5ms (`frameYieldMs`). This is called inside `workLoopConcurrentByScheduler` on every fiber. The moment a frame boundary is crossed, React stops.

**Read lines 532–548 (the `MessageChannel` setup):**
```js
const channel = new MessageChannel();
const port = channel.port2;
channel.port1.onmessage = performWorkUntilDeadline;
schedulePerformWorkUntilDeadline = () => {
  port.postMessage(null);
};
```
After yielding, React reschedules itself by calling `port.postMessage(null)`. The browser puts this in the task queue. The browser processes its own queue first (paint, input events), then calls `performWorkUntilDeadline` — which calls `flushWork` — which calls `workLoopConcurrentByScheduler` again.

**Why `MessageChannel` and not `setTimeout(fn, 0)`?** Browsers enforce a minimum 4ms delay on nested `setTimeout`. `MessageChannel` fires on the next task queue tick with no enforced delay — React gets back control as fast as the browser allows.

**What to notice:**
- React never "runs in the background". It runs, yields, gets resumed. All on the main thread.
- Concurrent React's "interruptibility" is not a thread pause — it is simply the `while` loop in `workLoopConcurrentByScheduler` exiting because `shouldYield()` returned true. The work-in-progress pointer is left where it was and resumed next turn.

---

## 2. Phase 2 — Reading the Full Source

After Phase 1 you have the vocabulary. Now read larger files in this order.

### Week 1–2: Data structures
```
ReactWorkTags.js       (~60 lines)   — all 32 fiber types
ReactFiberFlags.js     (~120 lines)  — effect bitmasks (already familiar)
ReactFiberLane.js      (~1302 lines) — priority bitmasks (already familiar)
ReactInternalTypes.js                — the Fiber type; read every field's comment
ReactFiber.js          (~959 lines)  — fiber constructor; note the `alternate` field
```
The `alternate` field in `ReactFiber.js` is React's **double-buffering**. There are always two fiber trees: `current` (what's on screen) and `workInProgress` (what React is computing). They point to each other via `alternate`. When the render is done, they swap.

### Week 3–4: The render cycle
```
ReactFiberReconciler.js   — public entry: createRoot().render()
ReactFiberRoot.js         — the FiberRoot object (different from the root Fiber)
ReactFiberWorkLoop.js     (~5337 lines) ⭐ — the main loop
```
In `ReactFiberWorkLoop.js`, find these functions and read them in order:
1. `scheduleUpdateOnFiber` (line 916) — called by `setState`, entry into the system
2. `ensureRootIsScheduled` — decides whether to schedule sync or async work
3. `performConcurrentWorkOnRoot` — the async render entry point
4. `renderRootConcurrent` / `renderRootSync` — calls the work loop
5. `commitRoot` (line 3416) — flushes everything to the DOM

### Week 5–6: Reconciliation
```
ReactFiberBeginWork.js    (~4417 lines) — "render" phase per fiber type
ReactFiberCompleteWork.js              — building host (DOM) tree bottom-up
ReactChildFiber.js        (~2251 lines) — the diffing algorithm ⭐
```
In `ReactChildFiber.js`, start at `reconcileChildrenArray` (line 1171) — this is the list diffing algorithm. It does two passes: first tries to match by index/key, then uses a `Map` for remaining children.

### Week 7–8: Hooks (you already know the foundation)
```
ReactFiberHooks.js        (~5260 lines) ⭐
```
You have already read `mountWorkInProgressHook`, `mountState`, `dispatchSetState`.
Now read:
- `mountEffectImpl` (line 2615) and `updateEffectImpl` (line 2632)
- `mountCallback` (line 2896) and `updateCallback` (line 2903) — notice how trivially simple `useCallback` is
- `mountMemo` (line 2917) and `updateMemo` (line 2936)
- `useReducer` — `useState` is literally `useReducer` with `basicStateReducer`

### Week 9–10: Commit phase
```
ReactFiberCommitWork.js    (~5331 lines) ⭐ — DOM mutations
ReactFiberCommitEffects.js               — effect cleanup and fire
```
The commit phase is split into three sub-phases that each walk the tree:
1. `commitBeforeMutationEffects` — `getSnapshotBeforeUpdate`
2. `commitMutationEffects` — actual DOM insertions/deletions/updates
3. `commitLayoutEffects` — `useLayoutEffect`, `componentDidMount/DidUpdate`
Then asynchronously: `flushPassiveEffects` — `useEffect`

### Week 11–12: Scheduler (already know the core)
```
Scheduler.js (forks/)       — task queue + yielding (already familiar)
SchedulerMinHeap.js         — already familiar
ReactFiberRootScheduler.js  — bridge between React and Scheduler
```
`ReactFiberRootScheduler.js` is the glue layer. It translates React lane priorities into Scheduler priorities and calls `scheduleCallback`.

---

## 3. Phase 3 — Internalize Patterns

After reading the source, these patterns will be obvious. They appear in virtually every modern JS library.

| Pattern | React location | General use |
|---|---|---|
| **Closure for capture** | `mountState` — dispatch closes over fiber+queue | Event handlers, callbacks, factories |
| **Linked list** | `ReactFiberHooks.js:980` — hook chain | Any ordered, growable structure without random access |
| **Bitmask flags** | `ReactFiberFlags.js`, `ReactFiberLane.js` | Permissions, feature flags, state machines |
| **Iterative DFS with pointer** | `ReactFiberWorkLoop.js:2990` | Any pauseable tree traversal |
| **Strategy / Dispatcher** | `ReactSharedInternals.H` | Swap implementations at runtime (e.g., dev vs prod, mount vs update) |
| **Min-heap priority queue** | `SchedulerMinHeap.js` | Task queues, event scheduling |
| **Cooperative multitasking** | `Scheduler.js:447` — shouldYieldToHost | Any long-running JS work that must share the main thread |
| **Double-buffering** | `ReactFiber.js` — `alternate` field | Rendering, canvas, any "prepare then swap" pattern |
| **Effect tagging** | Fiber `flags` field | Deferred side effects (mark → collect → flush) |
| **Monorepo with platform forks** | `packages/*/src/forks/` | Build-time code substitution without runtime conditionals |

---

## 4. Phase 4 — Build React From Scratch

The most effective way to verify you understand something is to build it.
Target: ~500 lines. No libraries.

```
Step 1  JSX factory
        createElement(type, props, ...children) → plain JS object { type, props, children }

Step 2  Fiber tree construction
        createFiber(element) → { tag, type, props, child, sibling, return, stateNode }

Step 3  Synchronous work loop (iterative DFS)
        while (workInProgress) { beginWork(); completeWork(); }

Step 4  DOM commit
        Walk the completed tree, call document.createElement / appendChild / etc.

Step 5  useState
        Store state in a linked list on the fiber.
        dispatch → enqueue update → scheduleRender()

Step 6  useEffect
        Tag fibers with a Passive flag during render.
        After commit, walk the tree, fire effects, store cleanup.
```

Finishing this means you built a real (tiny) React. Everything else in the actual codebase is error handling, optimizations, and platform coverage on top of this core.

---

## 5. Repo Map

### Root structure

```
react/
├── packages/          ← all source (monorepo, ~35 packages)
├── scripts/           ← build system (Rollup), CI, release tooling
├── compiler/          ← React Compiler (separate sub-monorepo)
├── fixtures/          ← manual integration test apps
├── flow-typed/        ← Flow type stubs for third-party libs
└── ReactVersions.js   ← single source of truth for version numbers
```

### Key packages

| Package | What it does | Why it exists separately |
|---|---|---|
| `react` | Public API: hooks, `createElement`, `createContext`, `Suspense` | The contract users code against — kept thin on purpose |
| `react-dom` | Binds React to the browser DOM | Renderer is swappable; this is just one target |
| `react-reconciler` | Core engine: fiber tree, work loop, diffing, hooks | Platform-agnostic — can be used to build any custom renderer |
| `scheduler` | Cooperative task scheduler | Standalone; could be used by anything needing priority-based async work |
| `react-dom-bindings` | DOM event system, attribute handling, hydration | Separated for tree-shaking |
| `react-server` | Server-side streaming renderer (RSC/Fizz) | Different execution model from the client reconciler |
| `react-server-dom-webpack` / `-turbopack` / `-parcel` / `-esm` | Bundler-specific RSC wire format | Each bundler needs different module resolution hooks |
| `react-client` | Client half of RSC — deserializes flight payloads | Mirror of server package |
| `react-native-renderer` | React Native renderer | Same reconciler, different host config |
| `react-art` | SVG/canvas renderer | Proves the reconciler is truly host-agnostic |
| `react-noop-renderer` | "Null" renderer used in tests | Best learning tool: reconciler with no DOM |
| `react-devtools` + `react-devtools-*` | Browser DevTools extension | Separate release cycle; hooks into reconciler via internal API |
| `react-refresh` | Hot module replacement | Preserves component state during dev reloads |
| `compiler/` | Babel plugin that auto-memoizes components | New (2024–); compiles away manual `useMemo`/`useCallback` |
| `eslint-plugin-react-hooks` | `rules-of-hooks`, `exhaustive-deps` lint rules | Enforces constraints the runtime can't catch |
| `shared` | Internal utilities shared across packages | Not published to npm |
| `react-is` | Type-checking (`isElement`, `isPortal`, etc.) | Stable, tiny; used by third-party libs |
| `use-sync-external-store` | Polyfill for `useSyncExternalStore` | Redux/Zustand use this for safe external state |

### The `forks/` pattern

Inside many packages there is a `src/forks/` directory. This is how React ships different code per platform (browser / React Native / Meta's internal `www` / test) without runtime `if/else`.

The Rollup build system (in `scripts/rollup/`) substitutes the right fork at build time:

```
packages/scheduler/src/forks/
  Scheduler.js              ← browser (MessageChannel-based)
  SchedulerNative.js        ← React Native
  SchedulerPostTask.js      ← experimental browser Scheduler API
  SchedulerMock.js          ← tests (manual time control via advanceTime)
```

### The `compiler/` sub-repo

A separate Babel/Rollup plugin that statically analyzes your components and inserts memoization automatically. Own `packages/`, own `yarn.lock`, own release cycle. Lives here so the compiler team can co-evolve it with the runtime.

---

## 6. Contribution Guide

### Getting the repo running

```bash
yarn                                           # install all dependencies

yarn build react react-dom --type=NODE         # build specific packages (fast)

yarn test packages/react-reconciler            # test a package

yarn jest ReactFiberHooks                      # run a single test file
yarn jest --testPathPattern=useState           # run tests matching a pattern
```

### Where to find first issues

1. GitHub issues labeled `good first issue` or `help wanted`
2. `packages/eslint-plugin-react-hooks` — small surface area, easy to add tests
3. Missing tests in `packages/react-reconciler/src/__tests__/` — adding edge case tests is valued
4. `compiler/` has its own issue tracker and a contributing guide in `compiler/README.md`

### Before submitting a PR

- Every change needs a test in the nearest `__tests__` directory
- `yarn prettier` and `yarn lint` before pushing
- Significant behavior changes need a Changelog entry and possibly a new `fixtures/` app
- Commit prefix convention: `[react-dom] Fix hydration mismatch`
- One behavior change per PR — small and focused merges fastest

### The Meta internal fork

You will see `// TODO(fb.www):` comments and `forks/*.www.js` files. These are Meta-internal and not your concern as an external contributor. Ignore them.

---

## Quick Reference: The Three Render Phases

```
┌─────────────────────────────────────────────────────────────────┐
│  1. RENDER PHASE  (can be interrupted mid-tree)                 │
│     ReactFiberBeginWork.js    ← "begin" each fiber             │
│     ReactFiberCompleteWork.js ← "complete" each fiber          │
│     ReactChildFiber.js        ← diff children                  │
│  Pure: no side effects. Builds work-in-progress tree.           │
│  Can be thrown away and restarted (Concurrent mode).            │
├─────────────────────────────────────────────────────────────────┤
│  2. COMMIT PHASE  (synchronous, cannot be interrupted)          │
│     BeforeMutation  → getSnapshotBeforeUpdate                  │
│     Mutation        → DOM inserts / deletes / updates          │
│     Layout          → useLayoutEffect, componentDidMount       │
│  After browser paint:                                           │
│     Passive         → useEffect (async, lowest priority)       │
├─────────────────────────────────────────────────────────────────┤
│  3. SCHEDULER  (underneath both phases)                         │
│     SchedulerMinHeap.js  ← min-heap task queue                 │
│     Scheduler.js         ← 5ms time slices, MessageChannel     │
└─────────────────────────────────────────────────────────────────┘
```
