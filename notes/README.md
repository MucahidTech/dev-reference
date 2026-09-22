# 📓 Full Stack Open — Personal Notes

Curated reference sheets for every part of the course. Not tutorials — canonical patterns, gotchas, and the "what to use when" decisions that matter in real projects.

## 🗂️ How to Use These Notes

Every file follows the same structure:

- **Index** at the top with anchored links.
- **One concept per section** — copy-paste ready snippets.
- **Key Takeaways** table at the end summarizing the essential rules.
- Code is inside ```` blocks (not ` `` `) for cleaner clipboard handling.

Search tips:

- Use your editor's "Find in files" (`Ctrl+Shift+F` / `Cmd+Shift+F`) across the `notes/` folder.
- Common search keys: `useMemo`, `populate`, `JWT`, `CORS`, `useShallow`, `memo`.

---

## 📚 Course Parts

| File                                                                         | Focus                                                                                                    | When to open                                                             |
| ---------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| [part0-fundamentals-of-web-apps.md](part0-fundamentals-of-web-apps.md)       | HTTP, AJAX, SPA, DOM, DevTools, Mermaid                                                                  | Reviewing the mental model of client-server, writing a sequence diagram  |
| [part1-react-foundations.md](part1-react-foundations.md)                     | Components, JSX, props, state, events, hooks rules                                                       | Setting up a new React project, reviewing `useState` patterns            |
| [part2-communicating-with-server.md](part2-communicating-with-server.md)     | Collections, controlled forms, Axios, promises, `useEffect`, service modules                             | Building a CRUD app with REST, debugging `useEffect` deps                |
| [part3-nodejs-express-mongodb.md](part3-nodejs-express-mongodb.md)           | Express, REST semantics, middleware, MongoDB + Mongoose, ESLint, deployment                              | Building or extending a backend API, adding validation                   |
| [part4-testing-express-user-auth.md](part4-testing-express-user-auth.md)     | Node test runner, Supertest, bcrypt, JWT, populate, `beforeEach` patterns                                | Writing integration tests, adding auth to a backend                      |
| [part5-testing-react-router-mui.md](part5-testing-react-router-mui.md)       | Vitest + RTL, Playwright E2E, React Router, MaterialUI, styled-components                                | Testing frontend, wiring routing, styling with MUI                       |
| [part6-advanced-state-management.md](part6-advanced-state-management.md)     | Zustand, TanStack Query, Context API, testing stores                                                     | Choosing a state library, writing async actions, testing global state    |
| [part7-custom-hooks-vite-internals.md](part7-custom-hooks-vite-internals.md) | `useMemo`/`useCallback`/`React.memo`, custom hooks, Vite + esbuild, error boundaries, monorepo, security | Optimizing renders, understanding Vite internals, structuring a monorepo |

---

## 🎯 Cross-Cutting Topics — Where to Find What

Some topics span multiple parts. Use this table to jump straight to the right file.

| Topic                              | Primary File                         | Also See                            |
| ---------------------------------- | ------------------------------------ | ----------------------------------- |
| **HTTP methods & status codes**    | Part 0                               | Part 3 (REST semantics)             |
| **Component design**               | Part 1                               | Part 7 (memoization)                |
| **Forms (controlled)**             | Part 2                               | Part 5 (useField hook)              |
| **Forms (uncontrolled)**           | Part 6                               | —                                   |
| **useEffect**                      | Part 2                               | Part 4 (test DB reset)              |
| **Promises / async-await**         | Part 2                               | Part 3 (Express 5 auto-handling)    |
| **Service modules (API layer)**    | Part 2                               | Part 6 (Zustand async actions)      |
| **Node test runner (`node:test`)** | Part 4                               | Part 6 (Vitest for stores)          |
| **Supertest**                      | Part 4                               | —                                   |
| **RTL + user-event**               | Part 5                               | Part 6 (renderHook + act)           |
| **Playwright**                     | Part 5                               | —                                   |
| **React Router**                   | Part 5                               | Part 7 (monorepo routing)           |
| **MaterialUI**                     | Part 5                               | —                                   |
| **Zustand**                        | Part 6                               | Part 7 (still relevant in monorepo) |
| **TanStack Query**                 | Part 6                               | —                                   |
| **Context API**                    | Part 6                               | —                                   |
| **useMemo / useCallback / memo**   | Part 7                               | —                                   |
| **Custom hooks**                   | Part 7                               | Part 5 (`useField`)                 |
| **Error boundaries**               | Part 7                               | —                                   |
| **JWT authentication**             | Part 4 (backend) + Part 5 (frontend) | —                                   |
| **bcrypt**                         | Part 4                               | —                                   |
| **Mongoose populate**              | Part 4                               | —                                   |
| **CORS vs Vite proxy**             | Part 3                               | Part 7 (monorepo)                   |
| **Environment variables**          | Part 3 (backend `.env`)              | Part 7 (Vite `VITE_` prefix)        |
| **ESLint**                         | Part 3 (backend)                     | Part 5 (frontend + Vitest globals)  |
| **GitHub Actions / CI**            | Part 7                               | —                                   |
| **Mermaid diagrams**               | Part 0                               | —                                   |

---

## 🧩 Supplementary Topics

Topics that appeared in the original notes but were not strictly part of a given part's material. **Not yet split into separate files** — decide based on your usage pattern:

| Topic                                                                                                   | Current location | Recommendation                                                                                   |
| ------------------------------------------------------------------------------------------------------- | ---------------- | ------------------------------------------------------------------------------------------------ |
| **Array methods (`map`, `filter`, `reduce`, `find`, `some`, `includes`, `concat`, `sort`, `toSorted`)** | Part 2 — §2      | ✅ Keep in Part 2. It's foundational and referenced from every later part.                       |
| **Object essentials (dot/bracket access, destructuring, spread)**                                       | Part 1 — §4      | ✅ Keep in Part 1. Everything downstream assumes it.                                             |
| **Class components (legacy)**                                                                           | Part 7 — §14     | ✅ Keep in Part 7. Only relevant for error boundaries and legacy maintenance.                    |
| **styled-components**                                                                                   | Part 5 — §22     | 🟡 Consider splitting later if used more heavily — currently a minor alternative to MUI.         |
| **Useful libraries & further reading**                                                                  | Not included     | ⚠ Split into `supplementary-libraries.md` if you find yourself needing a curated list regularly. |
| **Modern web architecture (SSR, RSC, microservices, serverless)**                                       | Part 7 — §20     | 🟡 Conceptual only. Could move to `architecture-trends.md` if it grows.                          |
| **Security (Helmet, audit, XSS, "never trust the client")**                                             | Part 7 — §19     | ✅ Keep in Part 7. Referenced from backend work across Parts 3–5.                                |

**When to create a new file:**

- The topic is referenced by **3+ different parts**.
- It's growing beyond ~50 lines in its current home.
- You find yourself searching for the same thing repeatedly.

Until then, keep it in the part file where it first appeared — the **cross-cutting table above** already provides a fast path to it.

---

## 🧠 Quick Decision Guide

Some "which tool?" questions come up repeatedly. Quick answers:

| Question                               | Answer                                                                                                                  |
| -------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Server data or client state?           | **Server → TanStack Query. Client → Zustand.**                                                                          |
| Context or Zustand?                    | Context for **small, rarely-changing** shared state (theme, notification). Zustand for **frequent or complex** updates. |
| `useState` or `useReducer`?            | `useState` for simple scalars/objects. `useReducer` for state machines with many transitions.                           |
| `useMemo` / `useCallback` / `memo`?    | Only after measuring. **None of them are free.**                                                                        |
| Class or function component?           | Function — always. Except `ErrorBoundary` (no hook equivalent yet).                                                     |
| CORS or Vite proxy?                    | **Vite proxy** in dev. CORS only when cross-origin is unavoidable in prod.                                              |
| RTL `getBy*` or CSS selector?          | **`getByRole` > `getByLabelText` > `getByText` > `getByTestId` > CSS.**                                                 |
| `invalidateQueries` or `setQueryData`? | `invalidateQueries` for correctness. `setQueryData` for performance.                                                    |
| `node:test` or Vitest?                 | **Backend → `node:test`. Frontend → Vitest.**                                                                           |
| Playwright or Cypress?                 | Playwright — Cypress was removed from the course in 2026.                                                               |

---

## 🔄 Maintenance

- **Add a new file** when a supplementary topic grows beyond 50 lines in its original home.
- **Update the cross-cutting table** whenever you add a new topic that spans parts.
- **Keep the "Quick Decision Guide"** short — it's a lookup, not a lecture.
