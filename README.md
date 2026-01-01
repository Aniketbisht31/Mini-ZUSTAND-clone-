



# Mini Zustand Clone

A lightweight, academic-grade re-implementation of modern state management internals inspired by Zustand, Recoil, and Jotai.

This repository is intentionally designed to explain *how state managers work internally*, not just how to use them.

---

## Architectural Overview

This library is built around **two complementary state models**:

1. **External Global Store (Zustand-style)**
2. **Atom & Derived Graph (Recoil/Jotai-style)**

Both models are **React-agnostic at the core** and integrate with React via `useSyncExternalStore`.

---

## High-Level Architecture Diagram

```
┌───────────────────────────┐
│        React UI           │
│  (Components & Hooks)     │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│   useSyncExternalStore    │
│  (Concurrency Safe API)   │
└─────────────┬─────────────┘
              │
   ┌──────────┴───────────┐
   │                      │
   ▼                      ▼
┌───────────────┐   ┌─────────────────┐
│ Global Store  │   │   Atom Graph     │
│ (Zustand)     │   │ (Recoil/Jotai)   │
└──────┬────────┘   └───────┬─────────┘
       │                    │
       ▼                    ▼
┌───────────────┐   ┌─────────────────┐
│ Middleware    │   │ Derived Atoms   │
│ (Time Travel) │   │ Dependency DAG  │
└───────────────┘   └─────────────────┘
```

---

## Project Structure

```
mini-zustand-clone/
├─ src/
│  ├─ store.ts        # Core store engine
│  ├─ useStore.ts     # React binding
│  ├─ create.ts       # Zustand-like create()
│  ├─ middleware.ts   # Example middleware
│  └─ index.ts        # Public API
├─ package.json
├─ tsconfig.json
├─ README.md
└─ LICENSE
```

---

## Core Concepts

### 1. External Store Model (Zustand)

```
state ──► setState ──► notify subscribers
  ▲                        │
  └──────── getState ◄─────┘
```

* State lives **outside React**
* React subscribes, not owns
* No provider or context tree

---

### 2. Selector-Based Subscription

```
Full State
   │
   ▼
Selector(state)
   │
   ▼
Component re-render (only if changed)
```

This ensures **O(1)** re-render cost per subscriber.

---

### 3. Time-Travel State Model

```
S0 ──► S1 ──► S2 ──► S3
        ▲     │
       undo   redo
```

* Immutable snapshots
* Linear history (no branching)
* Deterministic replay

---

### 4. Atom Dependency Graph

```
 Atom A      Atom B
    │          │
    └────┬─────┘
         ▼
    Derived Atom C
```

* Push-based recomputation
* Dependency invalidation
* Fine-grained updates

---

* External store (no React context)
* Fine-grained subscriptions via selectors
* React 18 compatible using `useSyncExternalStore`
* Middleware support

---

## Core Store Engine

```ts
// src/store.ts
export type Listener<T> = (state: T, prev: T) => void

export function createStore<T>(initialState: T) {
  let state = initialState
  const listeners = new Set<Listener<T>>()

  const getState = () => state

  const setState = (
    partial: Partial<T> | ((s: T) => Partial<T>),
    replace = false
  ) => {
    const prev = state
    const next = typeof partial === 'function' ? partial(state) : partial
    state = replace ? (next as T) : Object.assign({}, state, next)
    if (prev !== state) listeners.forEach(l => l(state, prev))
  }

  const subscribe = (listener: Listener<T>) => {
    listeners.add(listener)
    return () => listeners.delete(listener)
  }

  return { getState, setState, subscribe }
}
```

---

## React Binding

```ts
// src/useStore.ts
import { useSyncExternalStore, useRef } from 'react'

export function useStore<T, U>(
  store: { getState: () => T; subscribe: (l: any) => () => void },
  selector: (s: T) => U,
  equalityFn = Object.is
) {
  const sliceRef = useRef<U>()

  return useSyncExternalStore(
    store.subscribe,
    () => {
      const next = selector(store.getState())
      if (sliceRef.current !== undefined && equalityFn(sliceRef.current, next)) {
        return sliceRef.current
      }
      sliceRef.current = next
      return next
    }
  )
}
```

---

## Zustand-like create()

```ts
// src/create.ts
import { createStore } from './store'
import { useStore } from './useStore'

export function create<T>(
  initializer: (set: any, get: () => T) => T
) {
  const store = createStore({} as T)
  const initialState = initializer(store.setState, store.getState)
  store.setState(initialState, true)

  const useBoundStore = <U>(selector: (s: T) => U = s => s as any) =>
    useStore(store, selector)

  return Object.assign(useBoundStore, store)
}
```

---

## Middleware Example (Logger)

```ts
// src/middleware.ts
export const logger = (config: any) => (set: any, get: any) =>
  config((args: any) => {
    console.log('prev', get())
    set(args)
    console.log('next', get())
  }, get)
```

---

## Time-Travel Middleware (History Store)

```ts
// src/timeTravel.ts
type SetState<T> = (fn: Partial<T> | ((s: T) => Partial<T>), replace?: boolean) => void

export function timeTravel<T>(config: any) {
  return (set: SetState<T>, get: () => T) => {
    const history: T[] = []
    let pointer = -1

    const travelSet: SetState<T> = (fn, replace) => {
      const prev = get()
      const next = typeof fn === 'function' ? { ...prev, ...fn(prev) } : { ...prev, ...fn }

      history.splice(pointer + 1)
      history.push(next)
      pointer++

      set(next, true)
    }

    const state = config(travelSet, get)

    return {
      ...state,
      __history: history,
      undo: () => {
        if (pointer <= 0) return
        pointer--
        set(history[pointer], true)
      },
      redo: () => {
        if (pointer >= history.length - 1) return
        pointer++
        set(history[pointer], true)
      }
    }
  }
}
```

---

## Derived State / Atom System (Recoil-style)

```ts
// src/atom.ts
import { createStore } from './store'

export type Atom<T> = {
  get: () => T
  set?: (v: T) => void
  subscribe: (l: () => void) => () => void
}

export function atom<T>(initial: T): Atom<T> {
  const store = createStore(initial)
  return {
    get: store.getState,
    set: (v: T) => store.setState(v as any, true),
    subscribe: (l) => store.subscribe(() => l())
  }
}

export function derived<T>(compute: () => T, deps: Atom<any>[]): Atom<T> {
  let cached = compute()
  const listeners = new Set<() => void>()

  deps.forEach(dep => dep.subscribe(() => {
    cached = compute()
    listeners.forEach(l => l())
  }))

  return {
    get: () => cached,
    subscribe: (l) => {
      listeners.add(l)
      return () => listeners.delete(l)
    }
  }
}
```

---

## React Hook for Atoms

```ts
// src/useAtom.ts
import { useSyncExternalStore } from 'react'
import type { Atom } from './atom'

export function useAtom<T>(atom: Atom<T>) {
  return useSyncExternalStore(atom.subscribe, atom.get)
}
```

---

## Atom Usage Example

```ts
import { atom, derived } from './atom'
import { useAtom } from './useAtom'

const countAtom = atom(0)
const doubleAtom = derived(() => countAtom.get() * 2, [countAtom])

function Counter() {
  const count = useAtom(countAtom)
  const double = useAtom(doubleAtom)

  return (
    <>
      <button onClick={() => countAtom.set!(count + 1)}>+</button>
      <div>{count} / {double}</div>
    </>
  )
}
```

---

## Time-Travel Usage

```ts
import { create } from 'mini-zustand-clone'
import { timeTravel } from './timeTravel'

export const useCounter = create(
  timeTravel((set) => ({
    count: 0,
    inc: () => set(s => ({ count: s.count + 1 })),
    dec: () => set(s => ({ count: s.count - 1 }))
  }))
)

useCounter.getState().undo()
useCounter.getState().redo()
```

---

## Public API

```ts
// src/index.ts
export * from './create'
export * from './store'
export * from './middleware'
```

---

## Usage Example

```ts
import { create } from 'mini-zustand-clone'

export const useCounter = create((set) => ({
  count: 0,
  inc: () => set(s => ({ count: s.count + 1 }))
}))
```

---

## Computational Complexity

### External Store

* **setState**: O(n) subscribers
* **getState**: O(1)
* **subscribe/unsubscribe**: O(1)

### Selector-Based Rendering

* Selector evaluation: O(1)
* Re-render cost: only when selected slice changes

### Time-Travel Debugging

* Snapshot insertion: O(1)
* Undo / Redo: O(1)
* Memory: O(k × |state|), where k = history length

### Atom Graph

* Base atom update: O(d), where d = dependents
* Derived recomputation: minimal invalidation only

---

## Correctness Guarantees

* **No tearing**: Uses `useSyncExternalStore`, ensuring React consistency in concurrent rendering.
* **Deterministic replay**: Time-travel restores exact snapshots with full replacement semantics.
* **Snapshot isolation**: State updates never mutate historical snapshots.
* **Locality of updates**: Components only re-render when their subscribed slice or atom changes.

---

## References

* React RFC: External Store Support (`useSyncExternalStore`)
* Redux DevTools: Time-travel debugging architecture
* Recoil: Atom and selector dependency graph model
* Jotai: Primitive atom-based state management

---

## Why This Matters

This project demonstrates:

* External store architecture
* React concurrent rendering safety
* Selector-based performance optimization
* Middleware composition

Ideal for interviews, learning internals, or extending into devtools/time-travel.

---

## Testing (Optional)

```bash
npm install -D vitest jsdom
```

```ts
// tests/store.test.ts
import { create } from '../src'

test('increments count', () => {
  const useStore = create((set) => ({ count: 0, inc: () => set(s => ({ count: s.count + 1 })) }))
  expect(useStore.getState().count).toBe(0)
  useStore.getState().inc()
  expect(useStore.getState().count).toBe(1)
})
```

---

## package.json

```json
{
  "name": "mini-zustand-clone",
  "version": "0.1.0",
  "description": "Minimal Zustand-like state manager for learning and experimentation",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist"],
  "scripts": {
    "build": "tsc",
    "test": "vitest"
  },
  "peerDependencies": {
    "react": ">=18"
  },
  "devDependencies": {
    "typescript": "^5.3.3"
  },
  "license": "MIT"
}
```

---

## tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2019",
    "module": "ESNext",
    "declaration": true,
    "outDir": "dist",
    "strict": true,
    "moduleResolution": "bundler",
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

---

## Publish & GitHub Setup

```bash
git init
git add .
git commit -m "Initial commit: Zustand-like state manager"

git branch -M main
git remote add origin https://github.com/<your-username>/mini-zustand-clone.git
git push -u origin main
```

---

## License

MIT

---




