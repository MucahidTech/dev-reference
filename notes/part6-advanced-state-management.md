# Part 6 — Advanced State Management

> Reference for global state in React: Zustand for client-state, TanStack Query for server-state, and the Context API + useReducer for shared-but-simple state.
> Assumes prior knowledge. Focus: canonical patterns, correct store design, and testing.

## 📑 Index

1. [Why Global State](#1-why-global-state)
2. [Flux Architecture (Background)](#2-flux-architecture-background)
3. [Zustand — Core Concepts](#3-zustand--core-concepts)
4. [Zustand — Store Design Best Practices](#4-zustand--store-design-best-practices)
5. [Zustand — Immutability & Actions](#5-zustand--immutability--actions)
6. [Zustand — Uncontrolled Forms](#6-zustand--uncontrolled-forms)
7. [Zustand — Complex State (Filter Example)](#7-zustand--complex-state-filter-example)
8. [Fetch API](#8-fetch-api)
9. [Async Actions in Zustand](#9-async-actions-in-zustand)
10. [Zustand — Middlewares](#10-zustand--middlewares)
11. [Testing Zustand Stores](#11-testing-zustand-stores)
12. [Testing Custom Hooks with RTL](#12-testing-custom-hooks-with-rtl)
13. [TanStack Query — Server State](#13-tanstack-query--server-state)
14. [TanStack Query — Mutations & Cache Invalidation](#14-tanstack-query--mutations--cache-invalidation)
15. [TanStack Query — Optimizing Performance](#15-tanstack-query--optimizing-performance)
16. [TanStack Query — Custom Hook Pattern](#16-tanstack-query--custom-hook-pattern)
17. [Zustand vs TanStack Query](#17-zustand-vs-tanstack-query)
18. [Context API](#18-context-api)
19. [Context API — Best Practices](#19-context-api--best-practices)
20. [Context API — Custom Hook Guard](#20-context-api--custom-hook-guard)
21. [Key Takeaways](#21-key-takeaways)

---

## 1. Why Global State

**Until now:** state lived at the topmost common ancestor and was passed down via props.

**Problem as the app grows:**

- Deep prop drilling.
- Every state change re-renders a large subtree.
- Component coupling — changes ripple outward.

**Solution:** move state **out of the component tree** into an external store. Components subscribe only to the parts they need.

> "Do not store in state what can be computed." — Persian proverb quoted in Part 6

---

## 2. Flux Architecture (Background)

Facebook's early answer to state management pain.

**Flow:** `Action → Dispatcher → Store → View`

- State lives **outside** React components.
- State cannot be mutated directly — only through actions.
- When the store changes, subscribed views re-render.

**Redux** implemented Flux but suffered from boilerplate. **Redux Toolkit** improved it, but newer alternatives emerged: **MobX, Recoil, Jotai**. The current winner is **Zustand**.

---

## 3. Zustand — Core Concepts

```bash
npm install zustand
```

```js
// store.js
import { create } from "zustand";

const useCounterStore = create((set) => ({
  counter: 0,
  increment: () => set((state) => ({ counter: state.counter + 1 })),
  decrement: () => set((state) => ({ counter: state.counter - 1 })),
  zero: () => set(() => ({ counter: 0 })),
}));
```

**How it works:**

- `create(fn)` — receives a function returning the initial state.
- `set(updater)` — Zustand-provided helper to update the store. `updater` may be a new object or a function receiving the previous state.
- The function returned by `create` is a **React custom hook** (`useCounterStore`).

**Consuming in components:**

```jsx
const counter = useCounterStore((state) => state.counter);
const increment = useCounterStore((state) => state.increment);
```

**Selector function** `state => state.counter` — extracts only the slice the component cares about. The component re-renders **only when that slice changes**.

**The `App` becomes trivial:**

```jsx
const App = () => (
  <div>
    <Display />
    <Controls />
  </div>
);
```

No props, no state, no callbacks — each child subscribes to the store independently.

---

## 4. Zustand — Store Design Best Practices

### Avoid destructuring the whole store

```jsx
// ❌ Re-renders on EVERY change, even unrelated ones
const { increment, decrement, zero } = useCounterStore();

// ✅ Selects only the actions object → no re-renders when counter changes
const { increment, decrement, zero } = useCounterStore(
  (state) => state.actions,
);
```

### Group related values

```js
const useCounterStore = create((set) => ({
  counter: 0,
  actions: {
    increment: () => set((state) => ({ counter: state.counter + 1 })),
    decrement: () => set((state) => ({ counter: state.counter - 1 })),
    zero: () => set(() => ({ counter: 0 })),
  },
}));
```

### Export narrow hooks, not the raw store

```js
// ✅ Preferred export pattern
export const useCounter = () => useCounterStore((state) => state.counter);
export const useCounterControls = () =>
  useCounterStore((state) => state.actions);
```

Consumer code becomes:

```jsx
const counter = useCounter();
const { increment, decrement, zero } = useCounterControls();
```

### Hook naming rules

Everything created by `create` is a **React custom hook**. Same rules apply:

- Name must start with `use`.
- Never call inside loops, conditions, or nested functions.
- Never mutate state directly — always replace it.

---

## 5. Zustand — Immutability & Actions

**Actions must be pure functions:** no side effects, same inputs → same output.

### Appending to an array

```js
// ✅ Both create a NEW array
add: (note) => set((state) => ({ notes: state.notes.concat(note) }));
add: (note) => set((state) => ({ notes: [...state.notes, note] }));

// ❌ NEVER — mutates state
add: (note) =>
  set((state) => {
    state.notes.push(note);
    return state;
  });
```

### Updating an item in an array

```js
toggleImportance: (id) =>
  set((state) => ({
    notes: state.notes.map((note) =>
      note.id === id ? { ...note, important: !note.important } : note,
    ),
  }));
```

**`Array.toSorted()`** — creates a sorted copy without mutating the original (preferred over `.sort()` in state updates).

---

## 6. Zustand — Uncontrolled Forms

For simple forms, skip two-way binding. Use `name` + `e.target.reset()`.

```jsx
const addNote = (e) => {
  e.preventDefault();
  const content = e.target.note.value;
  add({ id: generateId(), content, important: false });
  e.target.reset();
};

<form onSubmit={addNote}>
  <input name="note" />
  <button type="submit">add</button>
</form>;
```

**Limitations:** no live validation, can't disable the submit button based on content, no per-keystroke side effects.

---

## 7. Zustand — Complex State (Filter Example)

**Rule:** keep completely unrelated concerns in separate stores. Related concerns (notes + their filter) belong in the same store.

```js
const useNoteStore = create((set) => ({
  notes: [],
  filter: "all",
  actions: {
    setFilter: (value) => set(() => ({ filter: value })),
  },
}));

export const useFilter = () => useNoteStore((state) => state.filter);
```

### Filter component

```jsx
const VisibilityFilter = () => {
  const { setFilter } = useNoteActions();
  return (
    <div>
      <input
        type="radio"
        name="filter"
        onChange={() => setFilter("all")}
        defaultChecked
      />{" "}
      all
      <input
        type="radio"
        name="filter"
        onChange={() => setFilter("important")}
      />{" "}
      important
      <input
        type="radio"
        name="filter"
        onChange={() => setFilter("nonimportant")}
      />{" "}
      not important
    </div>
  );
};
```

### Where should filtering logic live?

**Option A — In the component** (keeps store lean):

```jsx
const notes = useNotes();
const filter = useFilter();
const notesToShow = notes.filter((n) => {
  if (filter === "important") return n.important;
  if (filter === "nonimportant") return !n.important;
  return true;
});
```

**Option B — In the store hook** (cleaner consumer API):

```js
export const useNotes = () => {
  const notes = useNoteStore((state) => state.notes);
  const filter = useNoteStore((state) => state.filter);
  if (filter === "important") return notes.filter((n) => n.important);
  if (filter === "nonimportant") return notes.filter((n) => !n.important);
  return notes;
};
```

### ⚠️ The infinite-render trap

If you filter **inside a single selector**, Zustand sees a new array on each call → re-render → new array → loop.

**Fix with `useShallow`:**

```js
import { useShallow } from "zustand/react/shallow";

export const useNotes = () =>
  useNoteStore(
    useShallow(({ notes, filter }) => {
      if (filter === "important") return notes.filter((n) => n.important);
      if (filter === "nonimportant") return notes.filter((n) => !n.important);
      return notes;
    }),
  );
```

`useShallow` compares the array **element by element** instead of by reference. If nothing changed, it returns the previous reference → no re-render.

The course prefers Option B (two separate selector calls) — simpler and doesn't require `useShallow`.

---

## 8. Fetch API

Native alternative to Axios. No extra dependency.

### GET

```js
// src/services/notes.js
const baseUrl = "http://localhost:3001/notes";

const getAll = async () => {
  const response = await fetch(baseUrl);

  if (!response.ok) {
    throw new Error("Failed to fetch notes");
  }

  return await response.json();
};

export default { getAll };
```

**Notes:**

- `fetch` does **not** reject on HTTP errors (404, 500) — check `response.ok` (status 200–299).
- `response.json()` returns a Promise — must `await` it.

### POST

```js
const createNew = async (content) => {
  const options = {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ content, important: false }),
  };

  const response = await fetch(baseUrl, options);

  if (!response.ok) {
    throw new Error("Failed to create note");
  }

  return await response.json();
};
```

- `body` must be a string → `JSON.stringify` required.
- The server returns the created resource (with a generated id).
- The `Content-Type: application/json` header tells the server how to parse the body.

### PUT

```js
const update = async (id, note) => {
  const response = await fetch(`${baseUrl}/${id}`, {
    method: "PUT",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(note),
  });

  if (!response.ok) {
    throw new Error("Failed to update note");
  }

  return await response.json();
};
```

### JSON Server (mock backend)

```bash
npm install json-server --save-dev
```

```json
// package.json
{
  "scripts": { "server": "json-server -p 3001 db.json" }
}
```

```bash
npm run server
```

---

## 9. Async Actions in Zustand

**Move all server communication into the store** — components stay as pure presenters.

```js
import { create } from "zustand";
import noteService from "./services/notes";

const useNoteStore = create((set, get) => ({
  notes: [],
  filter: "",
  actions: {
    initialize: async () => {
      const notes = await noteService.getAll();
      set(() => ({ notes }));
    },

    add: async (content) => {
      const newNote = await noteService.createNew(content);
      set((state) => ({ notes: state.notes.concat(newNote) }));
    },

    toggleImportance: async (id) => {
      const note = get().notes.find((n) => n.id === id);
      const updated = await noteService.update(id, {
        ...note,
        important: !note.important,
      });
      set((state) => ({
        notes: state.notes.map((n) => (n.id === id ? updated : n)),
      }));
    },

    setFilter: (value) => set(() => ({ filter: value })),
  },
}));
```

**`get()` vs `useNoteStore.getState()`:**

- Both read the current state from within an action.
- `get()` is idiomatic inside the store; `getState()` is used from outside (e.g., tests).
- Preferred: `get()` — shorter and clearer.

**Component side:**

```jsx
// App
const { initialize } = useNoteActions();
useEffect(() => {
  initialize();
}, [initialize]);

// NoteForm
const { add } = useNoteActions();
const addNote = async (e) => {
  e.preventDefault();
  await add(e.target.note.value);
  e.target.reset();
};
```

**Best practice:** include `initialize` in the `useEffect` deps array even though its reference is stable. ESLint will thank you.

---

## 10. Zustand — Middlewares

Middlewares wrap the store without modifying its logic.

### Custom logger

```js
const logger = (config) => (set, get) =>
  config((...args) => {
    console.log("prev state", get());
    set(...args);
    console.log("next state", get());
  }, get);

const useNoteStore = create(
  logger((set, get) => ({
    // ...
  })),
);
```

### DevTools middleware (Redux DevTools browser extension)

```js
import { devtools } from "zustand/middleware";

const useNoteStore = create(
  devtools((set, get) => ({
    // ...
  })),
);
```

Now store changes are visible in the Redux DevTools tab.

---

## 11. Testing Zustand Stores

```bash
npm install --save-dev vitest
```

**Export the store itself** to enable direct access in tests:

```js
// store.js
export default useCounterStore;
```

```js
// store.test.js
import { beforeEach, describe, expect, it } from "vitest";
import useCounterStore from "./store";

beforeEach(() => {
  useCounterStore.setState({ counter: 0 });
});

describe("counter store", () => {
  it("initial state is 0", () => {
    expect(useCounterStore.getState().counter).toBe(0);
  });

  it("increment increases counter by 1", () => {
    useCounterStore.getState().actions.increment();
    expect(useCounterStore.getState().counter).toBe(1);
  });

  it("decrement decreases counter by 1", () => {
    useCounterStore.getState().actions.decrement();
    expect(useCounterStore.getState().counter).toBe(-1);
  });

  it("zero resets counter to 0", () => {
    const { increment, zero } = useCounterStore.getState().actions;
    increment();
    increment();
    zero();
    expect(useCounterStore.getState().counter).toBe(0);
  });
});
```

**Resetting state between tests:** `useCounterStore.setState({ counter: 0 })` — Zustand also offers a more complex `reset` setup; for simple stores this is enough.

---

## 12. Testing Custom Hooks with RTL

Use when **the hook contains logic worth testing independently** (e.g., a filter hook).

```bash
npm install --save-dev @testing-library/react jsdom
```

```js
// vite.config.js
export default defineConfig({
  plugins: [react()],
  test: { environment: "jsdom" },
});
```

```js
import { beforeEach, describe, expect, it } from "vitest";
import { renderHook, act } from "@testing-library/react";
import useCounterStore, { useCounter, useCounterControls } from "./store";

beforeEach(() => {
  useCounterStore.setState({ counter: 0 });
});

describe("counter hooks", () => {
  it("useCounter returns initial value of 0", () => {
    const { result } = renderHook(() => useCounter());
    expect(result.current).toBe(0);
  });

  it("increment updates counter", () => {
    const { result: counter } = renderHook(() => useCounter());
    const { result: controls } = renderHook(() => useCounterControls());

    act(() => controls.current.increment());

    expect(counter.current).toBe(1);
  });
});
```

### What is `act`?

`act` forces React to flush all queued state updates and side effects **before** the test continues. Without it, a test may read stale state because React batches updates.

RTL wraps many of its own methods in `act` automatically, but direct hook calls usually need it.

### Testing async actions with mocked services

```js
import { describe, it, expect, beforeEach, vi } from "vitest";
import { renderHook, act } from "@testing-library/react";

vi.mock("./services/notes", () => ({
  default: {
    getAll: vi.fn(),
    createNew: vi.fn(),
    update: vi.fn(),
  },
}));

import noteService from "./services/notes";
import useNoteStore, { useNotes, useNoteActions } from "./store";

beforeEach(() => {
  useNoteStore.setState({ notes: [], filter: "" });
  vi.clearAllMocks();
});

describe("useNoteActions", () => {
  it("initialize loads notes from service", async () => {
    const mockNotes = [{ id: 1, content: "Test", important: false }];
    noteService.getAll.mockResolvedValue(mockNotes);

    const { result } = renderHook(() => useNoteActions());

    await act(async () => {
      await result.current.initialize();
    });

    const { result: notesResult } = renderHook(() => useNotes());
    expect(notesResult.current).toEqual(mockNotes);
  });
});
```

**Notes:**

- `vi.mock` is hoisted — place it above imports.
- `vi.clearAllMocks()` resets call history between tests.
- Wrap async `act` with `await act(async () => { ... })`.

**Hint:** hook-based tests via jsdom are slower. If a hook contains no logic (just selects from the store), test the store directly.

---

## 13. TanStack Query — Server State

**For server state specifically** — different from client state.

```bash
npm install @tanstack/react-query
```

**Wrap the app:**

```jsx
// main.jsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

const queryClient = new QueryClient();

createRoot(document.getElementById("root")).render(
  <QueryClientProvider client={queryClient}>
    <App />
  </QueryClientProvider>,
);
```

**Fetch with `useQuery`:**

```jsx
import { useQuery } from "@tanstack/react-query";
import { getNotes } from "./requests";

const App = () => {
  const result = useQuery({
    queryKey: ["notes"],
    queryFn: getNotes,
  });

  if (result.isPending) return <div>loading data...</div>;

  const notes = result.data;
  // ...
};
```

**Result object fields:**

- `isPending` — request in flight.
- `isError` — request failed.
- `data` — the fetched value.

No `useState` / `useEffect` needed for server data.

---

## 14. TanStack Query — Mutations & Cache Invalidation

### `useMutation` for writes

```jsx
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { getNotes, createNote } from "./requests";

const App = () => {
  const queryClient = useQueryClient();

  const newNoteMutation = useMutation({
    mutationFn: createNote,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["notes"] });
    },
  });

  const addNote = (event) => {
    event.preventDefault();
    const content = event.target.note.value;
    event.target.reset();
    newNoteMutation.mutate({ content, important: true });
  };
  // ...
};
```

**Key idea:** `invalidateQueries` marks the `['notes']` query as stale → TanStack refetches → UI updates.

### PUT example

```jsx
const updateNoteMutation = useMutation({
  mutationFn: updateNote,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ["notes"] });
  },
});

const toggleImportance = (note) => {
  updateNoteMutation.mutate({ ...note, important: !note.important });
};
```

**Request functions live in `requests.js`** — one module, pure functions, no React.

---

## 15. TanStack Query — Optimizing Performance

`invalidateQueries` triggers an extra GET. For small apps this is fine; for large apps you may want to update the cache manually:

```jsx
const newNoteMutation = useMutation({
  mutationFn: createNote,
  onSuccess: (newNote) => {
    const notes = queryClient.getQueryData(["notes"]);
    queryClient.setQueryData(["notes"], notes.concat(newNote));
  },
});
```

- `getQueryData` reads the cached array.
- `setQueryData` writes it back with the new item appended.
- No network round-trip.

**Default behaviors to be aware of:**

| Behavior                | Default | Turn off with                 |
| ----------------------- | ------- | ----------------------------- |
| Refetch on window focus | on      | `refetchOnWindowFocus: false` |
| Retry on failure        | 3×      | `retry: false` or `retry: 1`  |

---

## 16. TanStack Query — Custom Hook Pattern

Extract all query + mutation logic into a hook:

```js
// hooks/useNotes.js
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { getNotes, createNote, updateNote } from "../requests";

export const useNotes = () => {
  const queryClient = useQueryClient();

  const result = useQuery({
    queryKey: ["notes"],
    queryFn: getNotes,
    refetchOnWindowFocus: false,
  });

  const newNoteMutation = useMutation({
    mutationFn: createNote,
    onSuccess: (newNote) => {
      const notes = queryClient.getQueryData(["notes"]);
      queryClient.setQueryData(["notes"], notes.concat(newNote));
    },
  });

  const updateNoteMutation = useMutation({
    mutationFn: updateNote,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["notes"] });
    },
  });

  return {
    notes: result.data,
    isPending: result.isPending,
    addNote: (content) => newNoteMutation.mutate({ content, important: true }),
    toggleImportance: (note) =>
      updateNoteMutation.mutate({ ...note, important: !note.important }),
  };
};
```

Component stays clean:

```jsx
const { notes, isPending, addNote, toggleImportance } = useNotes();
if (isPending) return <div>loading data...</div>;
```

---

## 17. Zustand vs TanStack Query

| Aspect                | Zustand                               | TanStack Query                         |
| --------------------- | ------------------------------------- | -------------------------------------- |
| **Domain**            | Client state (UI, forms, filters)     | Server state (async data + cache)      |
| **Async support**     | Yes, but manual                       | Built-in                               |
| **Caching**           | None                                  | Automatic                              |
| **Refetching**        | Manual                                | Automatic (focus, reconnect, interval) |
| **Best for**          | Notifications, user session, UI flags | Blog posts, users, any REST resource   |
| **Can they coexist?** | Yes — common in real apps             |

**Rule of thumb:** server data → TanStack Query. Everything else → Zustand.

**Note:** TanStack Query is **server-state cache**, not a replacement for client-state libraries. Most React apps need both.

---

## 18. Context API

React's built-in solution for **global state without prop drilling**.

```jsx
// CounterContext.jsx
import { createContext } from "react";

const CounterContext = createContext();
export default CounterContext;
```

**Provide via `<Context.Provider value={...}>`:**

```jsx
import CounterContext from "./components/CounterContext";

const App = () => {
  const [counter, setCounter] = useState(0);

  return (
    <CounterContext.Provider value={{ counter, setCounter }}>
      <Panel />
      <Footer />
    </CounterContext.Provider>
  );
};
```

**Consume via `useContext`:**

```jsx
import { useContext } from "react";
import CounterContext from "./CounterContext";

const Display = () => {
  const { counter } = useContext(CounterContext);
  return <div>{counter}</div>;
};
```

No prop drilling, no intermediate components passing state along.

---

## 19. Context API — Best Practices

### Move all context logic into its own file

```jsx
// CounterContext.jsx
import { createContext, useState } from "react";

const CounterContext = createContext();
export default CounterContext;

export const CounterContextProvider = (props) => {
  const [counter, setCounter] = useState(0);

  const increment = () => setCounter(counter + 1);
  const decrement = () => setCounter(counter - 1);
  const zero = () => setCounter(0);

  return (
    <CounterContext.Provider value={{ counter, increment, decrement, zero }}>
      {props.children}
    </CounterContext.Provider>
  );
};
```

### Wrap the app once, at the root

```jsx
// main.jsx
import { CounterContextProvider } from "./CounterContext";

createRoot(document.getElementById("root")).render(
  <CounterContextProvider>
    <App />
  </CounterContextProvider>,
);
```

### Split context by concern

If you expose values and actions separately, components re-render less:

```js
export const useCounterValue = () => useContext(CounterContext).counter;
export const useCounterControls = () => {
  const { increment, decrement, zero } = useContext(CounterContext);
  return { increment, decrement, zero };
};
```

### Custom hook over raw `useContext`

Encapsulate consumer logic in a `useX` hook — this also enables the next pattern.

---

## 20. Context API — Custom Hook Guard

**Context is `undefined` outside its provider.** Guard against it explicitly.

```js
// hooks/useNotification.js
import { useContext } from "react";
import NotificationContext from "../NotificationContext";

const useNotification = () => {
  const context = useContext(NotificationContext);
  if (!context) {
    throw new Error("useNotification must be used within NotificationProvider");
  }
  return context;
};

export default useNotification;
```

- Fails fast with a clear error message instead of a cryptic `undefined` crash deep in the tree.
- Consumers only import the hook — they don't need to know it's Context-backed.

---

## 21. Key Takeaways

| Principle         | Rule                                                                              |
| ----------------- | --------------------------------------------------------------------------------- |
| Store design      | One store per concern; related concerns share a store.                            |
| Selectors         | Always select the smallest slice a component needs.                               |
| Exports           | Export narrow hooks (`useX`, `useXActions`), not the raw store.                   |
| Immutability      | Actions must be pure; never mutate state — replace it.                            |
| Async in Zustand  | Async actions call services, then `set`. Use `get()` inside actions.              |
| Middlewares       | Wrap `create(...)` for logging, DevTools, persistence.                            |
| Testing stores    | `getState()` for reads, `setState()` to reset, `getState().actions.*` to invoke.  |
| Testing hooks     | `renderHook` + `act` from RTL; mock services with `vi.mock`.                      |
| Server state      | TanStack Query: `useQuery` to read, `useMutation` to write.                       |
| Cache sync        | Prefer `setQueryData` for surgical updates; `invalidateQueries` for full refetch. |
| Defaults          | TanStack refetches on window focus and retries 3× — override when needed.         |
| Context           | Provide at root; move logic into the Context file.                                |
| Custom hook guard | Throw if `context` is `undefined` — fail fast.                                    |
| `useShallow`      | Use when selecting a derived array/object to avoid infinite re-renders.           |
| Rule of thumb     | Server data → TanStack Query. Client data → Zustand.                              |

> _"Do not store in state what can be computed."_ — Part 6, Full Stack Open
