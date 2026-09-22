# Part 7 — Custom Hooks, Vite Internals, and Full-Stack Patterns

> Reference for advanced React hooks (`useMemo`, `useCallback`, `React.memo`, custom hooks), Vite's internals (esbuild vs Rollup), class components & error boundaries, monorepo patterns, and modern web architecture trends.
> Assumes prior knowledge. Focus: performance patterns, tooling internals, and production-ready architecture.

## 📑 Index

1. [Built-in Hooks Overview](#1-built-in-hooks-overview)
2. [useMemo](#2-usememo)
3. [React.memo](#3-reactmemo)
4. [useCallback](#4-usecallback)
5. [When NOT to Optimize](#5-when-not-to-optimize)
6. [Custom Hooks](#6-custom-hooks)
7. [useField Hook](#7-usefield-hook)
8. [useLocalStorage Hook](#8-uselocalstorage-hook)
9. [Bundling & Vite's Two Modes](#9-bundling--vites-two-modes)
10. [esbuild in Practice](#10-esbuild-in-practice)
11. [Vite Configuration](#11-vite-configuration)
12. [Environment Variables in Vite](#12-environment-variables-in-vite)
13. [Vite Plugins & Polyfills](#13-vite-plugins--polyfills)
14. [Class Components (Legacy)](#14-class-components-legacy)
15. [Error Boundaries](#15-error-boundaries)
16. [Frontend & Backend in One Repository](#16-frontend--backend-in-one-repository)
17. [React Project Folder Structure](#17-react-project-folder-structure)
18. [Real-Time Updates: Polling vs WebSockets](#18-real-time-updates-polling-vs-websockets)
19. [Security](#19-security)
20. [Modern Web Architecture](#20-modern-web-architecture)
21. [Key Takeaways](#21-key-takeaways)

---

## 1. Built-in Hooks Overview

React offers **18 built-in hooks**. Those used throughout the course:

| Hook                  | Purpose                                               |
| --------------------- | ----------------------------------------------------- |
| `useState`            | Local component state                                 |
| `useEffect`           | Side effects (fetch, subscribe)                       |
| `useContext`          | Read from a Context                                   |
| `useRef`              | Mutable reference persisting across renders           |
| `useImperativeHandle` | Expose child methods to parent via ref                |
| `useMemo`             | Memoize an expensive computed value                   |
| `useCallback`         | Memoize a function reference                          |
| `useReducer`          | Complex state via reducer (alternative to `useState`) |

**Rules (never negotiable):**

- Call hooks **only at the top level** of a function component or a custom hook.
- **Never** inside loops, conditions, or nested functions.
- ESLint plugin `eslint-plugin-react-hooks` enforces this — keep it enabled.

---

## 2. useMemo

**Purpose:** cache the result of an expensive computation across renders.

```jsx
import { useState, useMemo } from "react";

const FilteredList = () => {
  const [filter, setFilter] = useState("");
  const [darkMode, setDarkMode] = useState(false);

  const filtered = useMemo(() => {
    console.log("filtering...");
    return ITEMS.filter((item) => {
      expensiveCalculation();
      return item.includes(filter);
    });
  }, [filter]);

  return (
    <div style={{ background: darkMode ? "#333" : "#fff" }}>{/_ ... _/}</div>
  );
};
```

**Without `useMemo`:** toggling dark mode re-runs the filter → wasted CPU.
**With `useMemo`:** the filter is recomputed only when `filter` changes.

**Dependency array semantics:** identical to `useEffect` — React compares values across renders. If all match → use cache. Otherwise → recompute.

**Also useful for stabilizing object/array references** passed as props:

```jsx
const options = useMemo(() => ({ caseSensitive: false, filter }), [filter]);
return <SearchResults options={options} />;
```

**Rule:** `useMemo` is a **performance optimization, not a semantic one**. Add it only after measuring — it adds complexity and can hurt readability.

---

## 3. React.memo

**Higher-order component** that memoizes a component's rendered output.

```jsx
const MyComponent = React.memo(({ value }) => {
  console.log("rendered");
  return <div>{value}</div>;
});
```

**Behavior:** skips re-rendering the component when its **props** haven't changed since the last render.

**Important caveats:**

- `React.memo` compares **props only**. If the component uses state or context that changes, it still re-renders.
- If a prop is a **new object/array/function reference on every parent render**, `React.memo` is defeated — use `useMemo`/`useCallback` to stabilize the prop.

**Rule:** memoize only leaf components with heavy render cost. Wrapping everything makes the tree harder to reason about.

---

## 4. useCallback

**Purpose:** preserve a function's identity across renders.

### Two use cases

1. Passing a callback to a component wrapped in `React.memo`.
2. A function used as a dependency in `useEffect` / `useMemo`.

### The problem

```jsx
const App = () => {
  const [notes, setNotes] = useState([]);
  const [newNote, setNewNote] = useState("");

  const handleDelete = (id) => {
    // ❌ New reference every render
    setNotes((notes) => notes.filter((n) => n.id !== id));
  };

  return (
    <div>
      <input value={newNote} onChange={(e) => setNewNote(e.target.value)} />
      <NoteList notes={notes} onDelete={handleDelete} />
    </div>
  );
};

const NoteList = memo(({ onDelete, notes }) => {
  /_ ... _/;
});
```

Every keystroke re-renders `App` → `handleDelete` is a new function → `onDelete` prop changes → `NoteList` re-renders despite being memoized.

### The fix

```jsx
const handleDelete = useCallback((id) => {
  setNotes((notes) => notes.filter((n) => n.id !== id));
}, []);
```

Empty deps → the function never changes. `NoteList` now skips re-renders.

**Rule:** use `useCallback` **only** when a specific memo/effect depends on stable references. Sprinkling it everywhere hurts readability without measurable gain.

---

## 5. When NOT to Optimize

**The order of operations:**

1. **Measure first** — use the React DevTools Profiler.
2. **Optimize** only if a specific subtree renders too often or too slowly.
3. **Prefer structural fixes** (splitting components, moving state down) over memoization.

**Anti-patterns:**

- Wrapping every component in `React.memo` "just in case".
- `useCallback` on functions not passed as props or used in deps.
- `useMemo` around trivial computations (`a + b`).

---

## 6. Custom Hooks

**Definition:** a function whose name starts with `use` and that calls other hooks internally.

**When to build one:** the same logic is being copy-pasted across components.

**Rules:**

- Name **must** start with `use`.
- Follow all hook rules inside.
- Each call site gets its **own independent state** — hooks are not singletons.

### Counter example

```jsx
const useCounter = () => {
  const [value, setValue] = useState(0);

  const increase = () => setValue(value + 1);
  const decrease = () => setValue(value - 1);
  const zero = () => setValue(0);

  return { value, increase, decrease, zero };
};

const App = () => {
  const left = useCounter();
  const right = useCounter();

  return (
    <div>
      {left.value}
      <button onClick={left.increase}>left</button>
      <button onClick={right.increase}>right</button>
      {right.value}
    </div>
  );
};
```

Two calls → two completely separate counters.

### When a custom hook triggers re-renders

Think of the hook as inlined into the component:

- **State inside the hook changes** → re-render.
- **Context the hook subscribes to changes** → re-render.
- **Sending new args to the hook** → no re-render by itself.
- **Args used as effect/memo deps that trigger state updates** → re-render (indirect).

---

## 7. useField Hook

**Purpose:** manage form input state; reuse across all inputs.

```jsx
// hooks/useField.js
import { useState } from "react";

export const useField = (type) => {
  const [value, setValue] = useState("");

  const onChange = (event) => setValue(event.target.value);
  const reset = () => setValue("");

  return { type, value, onChange, reset };
};
```

**Usage with spread:**

```jsx
const App = () => {
  const name = useField("text");
  const born = useField("date");
  const height = useField("number");

  return (
    <form>
      name: <input {...name} />
      birthdate: <input {...born} />
      height: <input {...height} />
    </form>
  );
};
```

**Caveat:** spreading `{...name}` passes `reset` to the `<input>`, producing a React warning (`Invalid value for prop reset`). Fix:

```jsx
// Option 1 — destructure out reset
const { reset: \_reset, ...nameProps } = useField('text')
<input {...nameProps} />

// Option 2 — separate what's passed to <input>
export const useField = (type) => {
const [value, setValue] = useState('')
const onChange = (event) => setValue(event.target.value)
const reset = () => setValue('')
const inputProps = { type, value, onChange }
return { inputProps, reset }
}

<input {...field.inputProps} />
```

---

## 8. useLocalStorage Hook

**Purpose:** persistent state backed by localStorage.

```js
import { useState } from "react";

export const useLocalStorage = (key, initialValue) => {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      return initialValue;
    }
  });

  const setValue = (value) => {
    try {
      setStoredValue(value);
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch (error) {
      console.error(error);
    }
  };

  return [storedValue, setValue];
};
```

**Lazy initializer** `useState(() => ...)` — the function runs only on the first render, avoiding repeated localStorage reads.

---

## 9. Bundling & Vite's Two Modes

### Why bundling exists

Modern apps split code into ES modules. Older runtimes (and even some modern browsers) can't resolve module graphs efficiently. **Bundlers** concatenate, transpile, and optimize modules into a small set of files the browser can load in one shot.

```bash
npm run build # produces dist/
```

```text
dist/
├── assets/
│ ├── index-d526a0c5.css
│ ├── index-e92ae01e.js
│ └── react-35ef61ed.svg
├── index.html
└── vite.svg
```

### Vite's two radically different modes

**Development** (`npm run dev`):

- Vite does **not bundle**. It starts a dev server and lets the browser handle ES module imports directly.
- Third-party dependencies from `node_modules` are pre-bundled by **esbuild** for speed.
- Result: near-instant startup + HMR (Hot Module Replacement).

**Production** (`npm run build`):

- Uses **Rollup** for bundling.
- Uses **esbuild** for transformations (JSX, TypeScript).
- **Tree-shaking** removes unused exports from dependencies.
- Result: optimized, small bundle.

**Division of labor:** _esbuild for speed, Rollup for bundle quality_ — this is central to Vite's design.

**Why not esbuild for production?** esbuild's output quality is weaker in some cases (less code splitting support, chunk strategy less refined). Vite's team committed to switching once those gaps close.

---

## 10. esbuild in Practice

### Minimal React build from scratch

```bash
npm install react react-dom
npm install --save-dev esbuild
```

```json
// package.json
{
  "scripts": {
    "build": "esbuild src/main.jsx --bundle --outfile=dist/main.js --jsx=automatic",
    "serve": "npx serve dist"
  }
}
```

```html
<!-- dist/index.html -->
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <title>esbuild app</title>
  </head>
  <body>
    <div id="root"></div>
    <script src="./main.js"></script>
  </body>
</html>
```

```jsx
// src/main.jsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";

ReactDOM.createRoot(document.getElementById("root")).render(<App />);
```

### Flags worth knowing

| Flag              | Purpose                                             |
| ----------------- | --------------------------------------------------- |
| `--bundle`        | Resolve and concatenate all imports                 |
| `--outfile=`      | Output path                                         |
| `--jsx=automatic` | Transpile JSX (no `import React` required)          |
| `--minify`        | Shrink output (~1.1 MB → ~190 KB for a Hello World) |
| `--sourcemap`     | Emit `.map` for readable stack traces               |
| `--servedir=`     | Serve the output directory                          |
| `--watch`         | Rebuild on file changes                             |

**Source maps in production:** emit them for error-monitoring tools, but **never commit them to public static hosting** — they expose your full source.

---

## 11. Vite Configuration

Minimal config:

```js
// vite.config.js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
});
```

### Dev server options

```js
export default defineConfig({
  plugins: [react()],
  server: {
    port: 3000,
    open: true,
    proxy: {
      "/api": {
        target: "http://localhost:3001",
        changeOrigin: true,
      },
    },
  },
});
```

**Proxy** — forwards `/api/*` to the backend during development, avoiding CORS without touching the backend.

### Build options

```js
export default defineConfig({
  build: {
    sourcemap: true, // ⚠ exposes source in the browser
  },
});
```

---

## 12. Environment Variables in Vite

```env

# .env — development defaults

VITE_BACKEND_URL=http://localhost:3001/api/notes

# .env.production — production

VITE_BACKEND_URL=https://myapp.fly.dev/api/notes
```

**Rules:**

- Only variables prefixed with `VITE_` are exposed to the browser. Any other variable stays server-side — deliberate safety feature.
- Access in code via `import.meta.env`:

```js
const notes = useNotes(import.meta.env.VITE_BACKEND_URL);
```

- Vite loads the appropriate `.env` file based on mode (`.env`, `.env.development`, `.env.production`).
- **Add `.env*` files containing secrets to `.gitignore`.** Commit a `.env.example` documenting required variables.

---

## 13. Vite Plugins & Polyfills

Common plugins:

| Plugin                     | Purpose                        |
| -------------------------- | ------------------------------ |
| `@vitejs/plugin-react`     | JSX + Fast Refresh             |
| `@vitejs/plugin-legacy`    | Older browser support (Babel)  |
| `vite-plugin-svgr`         | Import SVG as React components |
| `rollup-plugin-visualizer` | Bundle size analysis           |

```js
import legacy from "@vitejs/plugin-legacy";

export default defineConfig({
  plugins: [react(), legacy({ targets: ["defaults", "not IE 11"] })],
});
```

**Polyfills** teach older browsers how to implement missing APIs. `plugin-legacy` injects polyfills + generates a separate bundle for legacy targets. Check compatibility at [caniuse.com](https://caniuse.com).

**CSS handling:**

- `import './index.css'` — global styles.
- `foo.module.css` — CSS Modules, imported as an object:

```jsx
import styles from "./App.module.css";

<div className={styles.container}>hello vite</div>;
```

- `.scss` files — install `sass`, then use directly. No config required.

---

## 14. Class Components (Legacy)

React's pre-16.8 way to own state. Still relevant for reading legacy code and **error boundaries** (no hook equivalent yet).

### State as a single object

```jsx
class App extends React.Component {
constructor(props) {
super(props)
this.state = {
anecdotes: [],
current: 0,
}
}

componentDidMount = () => {
axios.get('http://localhost:3001/anecdotes').then((response) => {
this.setState({ anecdotes: response.data })
})
}

handleClick = () => {
const current = Math.floor(Math.random() \* this.state.anecdotes.length)
this.setState({ current })
}

render() {
if (this.state.anecdotes.length === 0) return <div>no anecdotes...</div>

    return (
      <div>
        <h1>anecdote of the day</h1>
        <div>{this.state.anecdotes[this.state.current].content}</div>
        <button onClick={this.handleClick}>next</button>
      </div>
    )

}
}
```

**Key differences from function components:**

| Aspect       | Class                                                             | Function                              |
| ------------ | ----------------------------------------------------------------- | ------------------------------------- |
| State shape  | Single object (`this.state`)                                      | Multiple independent `useState` calls |
| Update       | `this.setState({...})`                                            | Individual setters                    |
| Lifecycle    | `componentDidMount`, `componentDidUpdate`, `componentWillUnmount` | `useEffect` with dependency array     |
| Hooks        | N/A                                                               | Available                             |
| Modern React | Legacy — avoid in new code                                        | Recommended                           |

The function-component equivalent of the above:

```jsx
const App = () => {
const [anecdotes, setAnecdotes] = useState([])
const [current, setCurrent] = useState(0)

useEffect(() => {
axios.get('http://localhost:3001/anecdotes').then((response) => {
setAnecdotes(response.data)
})
}, [])

const handleClick = () => {
setCurrent(Math.round(Math.random() \* (anecdotes.length - 1)))
}

if (anecdotes.length === 0) return <div>no anecdotes...</div>

return (
<div>
<h1>anecdote of the day</h1>
<div>{anecdotes[current].content}</div>
<button onClick={handleClick}>next</button>
</div>
)
}
```

---

## 15. Error Boundaries

**Class-only feature** — no hook equivalent yet.

Catches JS errors anywhere in a subtree and shows a fallback UI instead of crashing the whole app.

```jsx
import React from "react";

class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, info) {
    console.error("ErrorBoundary caught an error", error, info);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div>
          <h2>Something went wrong.</h2>
          <p>{this.state.error.message}</p>
          <button
            onClick={() => this.setState({ hasError: false, error: null })}
          >
            try again
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}

export default ErrorBoundary;
```

**Usage — wrap boundaries around independent regions:**

```jsx
const App = () => (
  <div>
    <ErrorBoundary>
      <Notes />
    </ErrorBoundary>
    <ErrorBoundary>
      <Persons />
    </ErrorBoundary>
  </div>
);
```

A crash in `<Notes>` won't kill `<Persons>`.

**Prefer the `react-error-boundary` library** — it wraps the class implementation in a function-component API.

---

## 16. Frontend & Backend in One Repository

Standard monorepo layout:

```
app/
├── package.json # root — scripts only
├── client/ # Vite + React
│ ├── package.json
│ ├── vite.config.js
│ └── src/
└── server/ # Express
├── package.json
└── index.js
```

**Express serves API in dev and static frontend in production:**

```js
// server/index.js
const express = require('express')
const path = require('path')

const app = express()
app.use(express.json())

app.get('/api/ping', (req, res) => {
res.json({ message: 'pong', time: new Date().toISOString() })
})

if (process.env.NODE_ENV === 'production') {
app.use(express.static(path.join(**dirname, '../client/dist')))
app.get('/\*splat', (req, res) => {
res.sendFile(path.join(**dirname, '../client/dist/index.html'))
})
}

const PORT = process.env.PORT || 3001
app.listen(PORT, () => console.log(`server running on port ${PORT}`))
```

**Vite dev proxy to backend:**

```js
// client/vite.config.js
export default defineConfig({
  plugins: [react()],
  server: { proxy: { "/api": "http://localhost:3001" } },
});
```

**Root `package.json` orchestrates both:**

```json
{
  "scripts": {
    "dev": "concurrently \"npm run dev --prefix server\" \"npm run dev --prefix client\"",
    "build": "npm run build --prefix client",
    "start": "NODE_ENV=production npm start --prefix server"
  },
  "devDependencies": {
    "concurrently": "^8.0.0"
  }
}
```

**`--prefix` flag** runs a script in a subdirectory: `npm run dev --prefix server` ≡ `cd server && npm run dev`.

**Install dependencies explicitly per side:**

```bash
npm install axios --prefix client
npm install mongoose --prefix server
```

---

## 17. React Project Folder Structure

### Small apps — split by type

```
src/
├── App.jsx
├── components/
│ ├── Blog.jsx
│ ├── BlogList.jsx
│ ├── LoginForm.jsx
│ └── Notification.jsx
├── hooks/
│ └── useField.js
├── services/
│ ├── blogs.js
│ └── users.js
└── stores/
├── blogStore.js
└── notificationStore.js
```

### With routing — add `pages/` (or `views/`)

```
src/
├── App.jsx
├── pages/
│ ├── HomePage.jsx
│ ├── BlogPage.jsx
│ └── UserPage.jsx
├── components/
├── hooks/
├── services/
└── stores/
```

### Large apps — split by feature (Feature-Sliced Design)

```
src/
├── App.jsx
├── features/
│ ├── blogs/
│ │ ├── Blog.jsx
│ │ ├── BlogList.jsx
│ │ ├── blogService.js
│ │ └── blogStore.js
│ ├── users/
│ │ ├── UserList.jsx
│ │ └── userService.js
│ └── notifications/
│ ├── Notification.jsx
│ └── notificationStore.js
└── hooks/
└── useField.js
```

**No universal "best" structure.** It depends on app size and team preference. Feature-based scales better; type-based is faster to navigate in small codebases.

---

## 18. Real-Time Updates: Polling vs WebSockets

### Polling

Frontend requests new data at a fixed interval.

```js
useEffect(() => {
  const interval = setInterval(() => {
    noteService.getAll().then(setNotes);
  }, 5000);
  return () => clearInterval(interval);
}, []);
```

**Pros:** trivial to implement.
**Cons:** expensive — most requests return unchanged data.

### WebSockets

Persistent bidirectional channel between client and server. The server **pushes** updates the moment they happen.

```js
// client
import { io } from "socket.io-client";
const socket = io("http://localhost:3001");

socket.on("note:created", (note) => {
  setNotes((prev) => [...prev, note]);
});
```

**Pros:** efficient, instant, real-time.
**Cons:** more infrastructure, harder to scale, requires persistent connections.

**Library:** `socket.io` — handles reconnection, fallbacks, and namespacing.

**Rule of thumb:** polling for low-frequency updates; WebSockets for chat, notifications, collaborative editing.

---

## 19. Security

### XSS and Mongoose

Mongoose sanitizes injected fields by default — user-submitted scripts stored in the DB won't execute on the frontend because React escapes them. Still, **validate and sanitize server-side**.

### Dependency hygiene

```bash
npm outdated --depth 0 # check outdated packages
npm install -g npm-check-updates
ncu -u # update all package.json versions
npm install # apply updates

npm audit # check for vulnerabilities
npm audit fix # safe fixes only
npm audit fix --force # ⚠ upgrades breaking majors (often breaks the app)
```

### Golden rule

> **Never trust the client.** Hiding a button is not security. Every mutation must be authorized **server-side**.

### Hardening Express

- **Helmet** — sets security headers (CSP, HSTS, X-Frame-Options, etc.).
- **eslint-plugin-security** — flags risky patterns like `eval`, `child_process` with user input.

---

## 20. Modern Web Architecture

### Typed JavaScript

**TypeScript** — compile-time type checking; catches bugs before runtime. Now standard in modern production apps.

### Server-Side Rendering & React Server Components

**SSR:** server renders the initial HTML, sends it complete to the client. Improves SEO and first-contentful-paint (browsers and crawlers see content without executing JS).

**React Server Components (RSC):** components render on the server and stream to the client as HTML. Interactive parts remain client components. Server components never ship JS to the browser.

**Next.js** is the dominant framework in this space — the default for SEO-heavy or full-stack React apps.

### Microservices

Each service owns a distinct API surface and possibly its own database. Services communicate over the network.

Example: a blogging platform could split into `posts-service` and `users-service`, each with its own DB.

Popularized by Jeff Bezos at Amazon (2002): every team owns its code end-to-end, with no shared entry point.

### Serverless

Cloud executes your code on demand — no server to provision or patch. You define functions, and the cloud handles scaling, routing, and lifecycle.

"Serverless" ≠ no server — it means the runtime is managed by the cloud provider. Ideal for sporadic workloads; less predictable for steady high throughput.

### 404 Route

```jsx
<Route path="\*" element={<NotFound />} />
```

Place it **last** among routes.

---

## 21. Key Takeaways

| Principle          | Rule                                                                        |
| ------------------ | --------------------------------------------------------------------------- |
| `useMemo`          | Cache expensive computations; dependency array = `useEffect` semantics.     |
| `React.memo`       | Skip re-render if props unchanged; defeated by new references.              |
| `useCallback`      | Stabilize callbacks passed to memoized components or used in deps.          |
| Optimization order | Measure → optimize → re-measure. Never preemptively memoize.                |
| Custom hooks       | Extract repeated logic; each call site gets its own state.                  |
| `useField`         | Standard input hook; watch out for the `reset` prop leak with spread.       |
| `useLocalStorage`  | Lazy initializer + JSON round-trip; catch parse errors.                     |
| Vite dev           | No bundling — dev server + browser-native ESM.                              |
| Vite prod          | Rollup bundling + esbuild transform + tree-shaking.                         |
| esbuild            | Fast transform/minify; source maps in dev only.                             |
| Env vars           | `VITE_` prefix required to expose to browser.                               |
| Class components   | Legacy; still required for error boundaries.                                |
| Error boundaries   | Wrap independent subtrees; prefer `react-error-boundary` for API.           |
| Monorepo           | `client/` + `server/`, root `package.json` orchestrates via `concurrently`. |
| Folder structure   | Type-based for small apps; feature-based for large.                         |
| Real-time          | Polling is easy but wasteful; WebSockets for push updates.                  |
| Security           | Never trust the client; use Helmet + audit dependencies.                    |
| Modern stack       | TypeScript, SSR/RSC via Next.js, microservices, serverless.                 |

> _"Measure, then optimize. Guessing at performance is itself a performance problem."_
