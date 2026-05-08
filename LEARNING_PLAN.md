# React Mastery — Learning Plan & Repo Guide

> Prepared for Przemyslaw on 2026-05-06. Revised 2026-05-07.
> Goals: master React internals · understand how modern JS libraries are built · market positioning · contribute to open source (Jotai/Zustand).

---

## Timeline at a Glance

| Phase | Hours | At 5–6h/week | At 2–3h/week |
|---|---|---|---|
| Phase 1 — JS Foundations (8 modules) | ~28h | 4–5 weeks | 9–14 weeks |
| Phase 2 — Full Source Reading (12 units) | ~45h | 7–9 weeks | 15–22 weeks |
| Phase 2.5 — RSC + React Compiler (4 units) | ~18h | 3–4 weeks | 6–9 weeks |
| Phase 3 — Internalize Patterns (10 sessions) | ~14h | 2–3 weeks | 5–7 weeks |
| Phase 4 — Build React From Scratch | ~28h | 4–5 weeks | 9–14 weeks |
| **Total** | **~133h** | **~5–6 months** | **~11–14 months** |

**Milestone checkpoints:**
- End of Phase 1: you can explain any hook's mechanism at source level
- End of Phase 2: you can navigate the full React codebase without getting lost
- End of Phase 2.5: you can explain RSC boundaries and the compiler's purpose in an interview
- End of Phase 4: you have a public GitHub repo as a portfolio artifact

---

## How to Use This Plan With Claude

At the start of every study session, tell Claude: **"Let's continue the learning plan."**
Claude will create a task list for the session using TodoWrite, track your progress through each module, and mark items complete as you go. This gives you a clear view of what's done, what's in progress, and what's next — both within a session and across the full plan.

Each module is a natural session boundary. If a module runs long, stop mid-module and Claude will pick up exactly where you left off next time.

After every two modules, do a **review session** before moving on. No new material — just self-explanation of the two completed modules. Cover: what the concept is, where React uses it, and why it matters. If you can explain both modules without hesitation, move on. If not, revisit the weaker one first.

At the end of every session, run a **quiz** before closing. The quiz must cover all source-level concepts touched that session — not just definitions, but mechanism. Questions should require the student to explain *why* something works, trace through code, or predict what would happen in an edge case. Do not proceed to the next session until the quiz is passed. If an answer is wrong or shallow, ask a follow-up until the concept is solid.

---

## Table of Contents

1. [Phase 1 — JS Foundations via React Source](#1-phase-1)
2. [Phase 2 — Reading the Full Source](#2-phase-2)
3. [Phase 2.5 — RSC + React Compiler](#25-phase-25)
4. [Phase 3 — Internalize Patterns](#3-phase-3)
5. [Phase 4 — Build React From Scratch](#4-phase-4)
6. [Repo Map — What Is Where and Why](#5-repo-map)
7. [Contribution Starter Guide](#6-contribution-guide)

---

## 1. Phase 1 — JS Foundations via React Source

No abstract exercises. Each JS concept is taught through the exact React code where it lives.
Work through these in order — each one unlocks the next.
**Target pace: 3–4 hours per module.**

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

**Self-explanation test:** close the laptop. Explain out loud: "When I call `setState`, how does React know which component to update?" If you can answer without hesitation, move on.

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

**Self-explanation test:** draw the linked list for a component with `useState`, `useEffect`, `useMemo`. Explain what happens on first render vs. second. Then explain to yourself: "why can't hooks be inside an if statement" — at source level, not just the rule.

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

**Self-explanation test:** in a Node REPL:
```js
const Placement = 0b010;
const Update    = 0b100;
const flags = Placement | Update;
console.log((flags & Placement) !== 0); // true
console.log((flags & 0b001) !== 0);     // false
flags & ~Placement                      // clear Placement
```
Then explain: "how does React skip an entire subtree without walking it?"

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

**Self-explanation test:** explain why recursive DFS cannot be interrupted and iterative DFS can. Then explain: "what happens to `workInProgress` when React yields to the browser?"

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
- This pattern (a mutable slot + swappable implementations) appears in virtually every large JS library.

**Self-explanation test:** explain the "Invalid hook call" error at source level. Not "you called a hook outside a component" — explain exactly what is null and why.

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

**Self-explanation test:** implement this from memory. It should be ~30 lines. If you can do it, you understand it.

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
    return false;
  }
  return true;
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
After yielding, React reschedules itself by calling `port.postMessage(null)`. The browser puts this in the task queue. The browser processes its own queue first (paint, input events), then calls `performWorkUntilDeadline`.

**Why `MessageChannel` and not `setTimeout(fn, 0)`?** Browsers enforce a minimum 4ms delay on nested `setTimeout`. `MessageChannel` fires on the next task queue tick with no enforced delay.

**Self-explanation test:** explain "Concurrent React" to a colleague without using the word "concurrent". Then explain why `MessageChannel` beats `setTimeout(fn, 0)`.

---

### Module 8 — `useSyncExternalStore`: the tearing problem and how Zustand works

**The concept:** in Concurrent React, the render phase can be interrupted and resumed. If an external store changes mid-render, different components can read different snapshots of the same store — a visual inconsistency called *tearing*.

**Why this matters:** Zustand's entire subscription model is built on this hook. Understanding it is the bridge between React internals and contributing to Jotai/Zustand.

**Read the source:**
```
packages/react-reconciler/src/ReactFiberHooks.js
```
Search for `mountSyncExternalStore` and `updateSyncExternalStore`. Key behavior:
- On mount: subscribes to the external store, reads the current snapshot
- On update: compares the current snapshot with the stored snapshot — if they differ mid-render, React forces a synchronous re-render to restore consistency
- The subscription fires on every store change, triggering a re-render

**Then look at Zustand's usage:**
```bash
# in a separate Zustand repo or node_modules
grep -n "useSyncExternalStore" node_modules/zustand/src/react.ts
```
You'll see Zustand passes its store's `subscribe` and `getSnapshot` directly into `useSyncExternalStore`. That's the entire React integration — ~5 lines.

**What to notice:**
- Without `useSyncExternalStore`, an external store + concurrent rendering = potential tearing
- `useSyncExternalStore` is React's contract for safe external state. Any library that manages state outside React (Zustand, Jotai, Redux) must use this or accept tearing risk
- Jotai uses it too, but wraps it in atoms — the subscription model is the same underneath

**Self-explanation test:** explain tearing. Then explain why Zustand's core is actually tiny — it delegates all the hard React integration work to `useSyncExternalStore`.

---

## 2. Phase 2 — Reading the Full Source

After Phase 1 you have the vocabulary. Now read larger files in this order.
**Target pace: 3–5 hours per week-unit.**

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
In `ReactChildFiber.js`, start at `reconcileChildrenArray` (line 1171) — the list diffing algorithm. Two passes: first matches by index/key, then uses a `Map` for remaining children.

**After this unit:** self-explanation exercise — "why does moving a list item without a key cause it to remount instead of move?" Answer at source level using `reconcileChildrenArray`.

### Week 7–8: Hooks (you already know the foundation)
```
ReactFiberHooks.js        (~5260 lines) ⭐
```
You have already read `mountWorkInProgressHook`, `mountState`, `dispatchSetState`, `mountSyncExternalStore`.
Now read:
- `mountEffectImpl` (line 2615) and `updateEffectImpl` (line 2632)
- `mountCallback` (line 2896) and `updateCallback` (line 2903) — notice how trivially simple `useCallback` is
- `mountMemo` (line 2917) and `updateMemo` (line 2936)
- `useReducer` — `useState` is literally `useReducer` with `basicStateReducer`

**After this unit:** self-explanation — "when should I reach for `useMemo`?" Answer using what you now know about `updateMemo` — what exactly does it compare, and what's the cost of the comparison itself?

### Week 9–10: Commit phase
```
ReactFiberCommitWork.js    (~5331 lines) ⭐ — DOM mutations
ReactFiberCommitEffects.js               — effect cleanup and fire
```
The commit phase is split into three sub-phases:
1. `commitBeforeMutationEffects` — `getSnapshotBeforeUpdate`
2. `commitMutationEffects` — actual DOM insertions/deletions/updates
3. `commitLayoutEffects` — `useLayoutEffect`, `componentDidMount/DidUpdate`
Then asynchronously: `flushPassiveEffects` — `useEffect`

**After this unit:** self-explanation — "why does `useLayoutEffect` block the browser from painting, but `useEffect` doesn't?" Answer using the commit phase sequence you just read.

### Week 11–12: Scheduler (already know the core)
```
Scheduler.js (forks/)       — task queue + yielding (already familiar)
SchedulerMinHeap.js         — already familiar
ReactFiberRootScheduler.js  — bridge between React and Scheduler
```
`ReactFiberRootScheduler.js` is the glue layer. It translates React lane priorities into Scheduler priorities and calls `scheduleCallback`.

**After this unit:** self-explanation — "what happens when you call `startTransition`?" Walk the full path from `startTransition` → lane assignment → scheduler → work loop → render.

---

## 2.5. Phase 2.5 — RSC + React Compiler

These are not internals-level deep dives. The goal is interview fluency and the ability to advise colleagues — not source-level mastery.
**Target pace: 4–5 hours per week-unit.**

### Week 1–2: React Server Components mental model

**What RSC is (and is not):**
- Server Components execute on the server. They have no hooks, no event handlers, no fiber in the traditional client sense.
- They send a serialized component tree — not HTML — to the client via the React Flight protocol. The client reconciler merges this with the existing tree.
- The boundary between Server and Client is explicit: `"use client"` marks the transition point. Everything above it is server; everything below is client.

**Key concepts to be able to explain:**
- Why RSC reduces bundle size (server component code never ships to the client)
- Why you can `async/await` directly in a Server Component but not a Client Component
- What "interleaving" means — Server Components can render Client Components as children, but not vice versa (without slots/children)
- The difference between SSR (server renders HTML) and RSC (server renders a component tree that the client can update incrementally)

**Where to look in the source:**
```
packages/react-server/          — the server-side renderer
packages/react-client/          — client-side deserialization of Flight payloads
```
Read the READMEs and top-level exports. You don't need to read every line — understand the shape.

**Self-explanation test:** explain RSC to a colleague who knows React well but hasn't used Next.js App Router. No jargon. Then explain: "why can't I use `useState` in a Server Component?" at mechanism level.

### Week 3: React Compiler

**What it does:**
The React Compiler is a Babel plugin that statically analyzes your components and automatically inserts `useMemo` and `useCallback` where it can prove inputs haven't changed. The goal: remove manual memoization from application code entirely.

**What it can't do:**
- It cannot handle components that violate the Rules of React (mutating props, reading external mutable state without `useSyncExternalStore`, etc.)
- It will bail out of optimizing a component if it can't prove it's safe

**Where to look:**
```
compiler/README.md              — start here
compiler/packages/babel-plugin-react-compiler/
```
Look at a few compiler output examples. Run `npx react-compiler-healthcheck` on a codebase if you have one. Read what it flags and why.

**Self-explanation test:** explain why the React Compiler exists — what problem does manual `useMemo` have at scale? Then explain: "if the compiler handles memoization, does `useCallback` become obsolete?" (Answer: mostly yes, for components the compiler can optimize.)

### Week 4: Interview consolidation

No new reading. Spend this week doing self-explanation sessions on the two hardest questions RSC generates:

1. "When would you use a Server Component vs a Client Component?"
2. "What is the React Compiler and would you use it in production today?"

If you can answer both fluently without pausing to think, Phase 2.5 is done.

---

## 3. Phase 3 — Internalize Patterns

After reading the source, these patterns will be obvious across any codebase. The goal of this phase is not more reading — it is **active recall under self-imposed pressure**.

For each pattern: close the laptop. Explain it out loud as if an interviewer just asked. Cover: what the pattern is, where React uses it, when you'd use it in your own code, and what problem it solves that a naive approach wouldn't.

| Pattern | React location | Interview angle |
|---|---|---|
| **Closure for capture** | `mountState` — dispatch closes over fiber+queue | "How does `setState` know which component to update?" |
| **Linked list** | `ReactFiberHooks.js:980` — hook chain | "Why can't hooks be called conditionally?" |
| **Bitmask flags** | `ReactFiberFlags.js`, `ReactFiberLane.js` | "How does React skip subtrees efficiently?" |
| **Iterative DFS with pointer** | `ReactFiberWorkLoop.js:2990` | "How does Concurrent React interrupt rendering?" |
| **Strategy / Dispatcher** | `ReactSharedInternals.H` | "Why does calling a hook outside a component throw?" |
| **Min-heap priority queue** | `SchedulerMinHeap.js` | "Why doesn't a transition block a button click?" |
| **Cooperative multitasking** | `Scheduler.js:447` — shouldYieldToHost | "How does React stay on the main thread without blocking UI?" |
| **Double-buffering** | `ReactFiber.js` — `alternate` field | "What is `workInProgress` and why are there two trees?" |
| **Effect tagging** | Fiber `flags` field | "What is the commit phase and why is it split into three sub-phases?" |
| **External store subscription** | `useSyncExternalStore` | "How does Zustand integrate with React without tearing?" |

**One session per pattern. Ten sessions total. ~1–1.5 hours each.**
You are done with Phase 3 when you can explain all ten without hesitation.

---

## 4. Phase 4 — Build React From Scratch

The most effective verification of understanding. Target: ~500 lines. No libraries.
**Make this repo public on GitHub with a README — it is your primary portfolio artifact.**

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

### README requirements (keep it to one page)

The README is not a blog post. Five sections, each two to four sentences:

1. **What this is** — a minimal React implementation to understand the real one
2. **What's implemented** — JSX factory, fiber tree, work loop, `useState`, `useEffect`
3. **What's deliberately omitted** — concurrent mode, error boundaries, keys, events
4. **Architecture** — one paragraph on the three-phase structure (render → commit → effects) and why the work loop is iterative
5. **How to run** — one command

This README is what a hiring manager or colleague reads. It signals depth without requiring them to read code.

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

Contribution is a secondary goal — pursue it after Phase 4, or opportunistically during Phase 2 if an issue grabs you.

### Primary targets: Jotai and Zustand

Both are maintained by Daishi Kato (pmndrs ecosystem). Small codebases, welcoming maintainer, active issue tracker.

**Read the source first (one session each):**
```
Zustand core:  ~1000 lines — read src/react.ts and src/vanilla.ts
Jotai core:    ~2000 lines — read src/vanilla.ts and src/react.ts
```
After Phase 1 Module 8, Zustand's `useSyncExternalStore` integration will make complete sense.

**Finding issues:**
- GitHub: filter by `good first issue` or `help wanted`
- Look for missing test coverage, edge cases in TypeScript types, documentation gaps
- Daishi responds quickly — a question in a GitHub discussion is a valid first contribution

### React ecosystem (secondary)

If a React-adjacent issue grabs you during your study:
- `packages/eslint-plugin-react-hooks` — small surface area, adding tests is valued
- `packages/react-devtools` — fiber knowledge from Phase 2 maps directly here
- Missing tests in `packages/react-reconciler/src/__tests__/`

### Before submitting a PR

- Every change needs a test in the nearest `__tests__` directory
- `yarn prettier` and `yarn lint` before pushing
- One behavior change per PR — small and focused merges fastest
- Commit prefix convention: `[react-dom] Fix hydration mismatch`

### The Meta internal fork

You will see `// TODO(fb.www):` comments and `forks/*.www.js` files. These are Meta-internal. Ignore them.

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
