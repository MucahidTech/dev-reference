# Part 5 — Testing React Apps, React Router, UI Frameworks

> Reference for testing React components and applications (Vitest + RTL + Playwright), implementing JWT authentication on the frontend, and building polished UIs with React Router and MaterialUI.
> Assumes prior knowledge. Focus: canonical patterns, correct testing workflow, and idiomatic frontend architecture.

## 📑 Index

1. [Frontend Login Form](#1-frontend-login-form)
2. [Persisting the Session](#2-persisting-the-session)
3. [Extracting the Login Form](#3-extracting-the-login-form)
4. [Togglable Component & props.children](#4-togglable-component--propschildren)
5. [Lifting State Up](#5-lifting-state-up)
6. [Component Refs & useImperativeHandle](#6-component-refs--useimperativehandle)
7. [ESLint for Frontend](#7-eslint-for-frontend)
8. [Vitest + React Testing Library Setup](#8-vitest--react-testing-library-setup)
9. [Basic Component Tests](#9-basic-component-tests)
10. [Query Methods (getBy, findBy, queryBy)](#10-query-methods-getby-findby-queryby)
11. [Event Simulation with user-event](#11-event-simulation-with-user-event)
12. [Testing Togglable Components](#12-testing-togglable-components)
13. [Testing Forms](#13-testing-forms)
14. [Coverage & Snapshots](#14-coverage--snapshots)
15. [End-to-End Testing with Playwright](#15-end-to-end-testing-with-playwright)
16. [Playwright Locators & Assertions](#16-playwright-locators--assertions)
17. [Test Database Reset Endpoint](#17-test-database-reset-endpoint)
18. [Playwright Helpers](#18-playwright-helpers)
19. [Playwright Debugging](#19-playwright-debugging)
20. [React Router](#20-react-router)
21. [MaterialUI](#21-materialui)
22. [styled-components](#22-styled-components)
23. [Key Takeaways](#23-key-takeaways)

---

## 1. Frontend Login Form

**Flow:** user submits credentials → `POST /api/login` → server returns `{ token, username, name }` → token stored and attached to subsequent requests.

```js
// src/services/login.js
import axios from "axios";
const baseUrl = "/api/login";

const login = async (credentials) => {
  const response = await axios.post(baseUrl, credentials);
  return response.data;
};

export default { login };
```

```jsx
// In App
const [username, setUsername] = useState("");
const [password, setPassword] = useState("");
const [user, setUser] = useState(null);

const handleLogin = async (event) => {
  event.preventDefault();
  try {
    const user = await loginService.login({ username, password });
    setUser(user);
    setUsername("");
    setPassword("");
  } catch {
    setErrorMessage("wrong credentials");
    setTimeout(() => setErrorMessage(null), 5000);
  }
};
```

**Conditional rendering — login form only when not logged in:**

```jsx
{
  !user && loginForm();
}
{
  user && noteForm();
}
```

```jsx
// Show user name alongside content
{
  user && (
    <div>
      <p>{user.name} logged in</p>
      {noteForm()}
    </div>
  );
}
```

**Accessibility:** wrap inputs inside `<label>` — this labels the input without needing extra `id`/`for` attributes and works well with screen readers.

---

## 2. Persisting the Session

**Token attachment to requests:**

```js
// src/services/notes.js
import axios from "axios";
const baseUrl = "/api/notes";

let token = null;

const setToken = (newToken) => {
  token = `Bearer ${newToken}`;
};

const create = async (newObject) => {
  const config = { headers: { Authorization: token } };
  const response = await axios.post(baseUrl, newObject, config);
  return response.data;
};

export default { getAll, create, update, setToken };
```

**Call `setToken` right after login:**

```jsx
const user = await loginService.login({ username, password });
noteService.setToken(user.token);
setUser(user);
```

**Persist to localStorage** (survives refresh):

```jsx
window.localStorage.setItem("loggedNoteappUser", JSON.stringify(user));
```

**localStorage API:**

| Call                  | Purpose                          |
| --------------------- | -------------------------------- |
| `setItem(key, value)` | Store a value (must be a string) |
| `getItem(key)`        | Read a value                     |
| `removeItem(key)`     | Delete a key                     |
| `clear()`             | Wipe all keys                    |

**Important:** localStorage stores **strings only**. Serialize objects with `JSON.stringify` / restore with `JSON.parse`.

**Check for existing session on mount:**

```jsx
useEffect(() => {
  const loggedUserJSON = window.localStorage.getItem("loggedNoteappUser");
  if (loggedUserJSON) {
    const user = JSON.parse(loggedUserJSON);
    setUser(user);
    noteService.setToken(user.token);
  }
}, []);
```

**Security caveat:** storing JWTs in localStorage exposes them to XSS. Keep expiry short; prefer HttpOnly cookies for sensitive apps.

---

## 3. Extracting the Login Form

Extract into its own component. Pass all state + handlers via props.

```jsx
// src/components/LoginForm.jsx
const LoginForm = ({
  handleSubmit,
  handleUsernameChange,
  handlePasswordChange,
  username,
  password,
}) => (
  <div>
    <h2>Login</h2>
    <form onSubmit={handleSubmit}>
      <div>
        <label>
          username
          <input value={username} onChange={handleUsernameChange} />
        </label>
      </div>
      <div>
        <label>
          password
          <input
            type="password"
            value={password}
            onChange={handlePasswordChange}
          />
        </label>
      </div>
      <button type="submit">login</button>
    </form>
  </div>
);
export default LoginForm;
```

**Rule of thumb:** the child owns nothing stateful — state stays in the parent.

**Conditional visibility with a dedicated state:**

```jsx
const [loginVisible, setLoginVisible] = useState(false);

const hideWhenVisible = { display: loginVisible ? "none" : "" };
const showWhenVisible = { display: loginVisible ? "" : "none" };
```

---

## 4. Togglable Component & props.children

**`props.children`** — React auto-passes any element wrapped between a component's opening and closing tags.

```jsx
// src/components/Togglable.jsx
import { useState } from "react";

const Togglable = ({ buttonLabel, children }) => {
  const [visible, setVisible] = useState(false);

  const hideWhenVisible = { display: visible ? "none" : "" };
  const showWhenVisible = { display: visible ? "" : "none" };

  return (
    <div>
      <div style={hideWhenVisible}>
        <button onClick={() => setVisible(true)}>{buttonLabel}</button>
      </div>
      <div style={showWhenVisible}>
        {children}
        <button onClick={() => setVisible(false)}>cancel</button>
      </div>
    </div>
  );
};
export default Togglable;
```

**Usage:**

```jsx
<Togglable buttonLabel="login">
  <LoginForm
    username={username}
    password={password}
    handleUsernameChange={({ target }) => setUsername(target.value)}
    handlePasswordChange={({ target }) => setPassword(target.value)}
    handleSubmit={handleLogin}
  />
</Togglable>
```

**Note:** `children` is an array of React elements, added automatically to every component. It's empty (`[]`) if the component self-closes.

---

## 5. Lifting State Up

**Official rule:** when two components need to share or synchronize state, move that state to their **closest common ancestor** and pass it down via props.

**Example — moving `newNote` from `App` into `NoteForm`:**

```jsx
// src/components/NoteForm.jsx — owns its own state
import { useState } from "react";

const NoteForm = ({ createNote }) => {
  const [newNote, setNewNote] = useState("");

  const addNote = (event) => {
    event.preventDefault();
    createNote({ content: newNote, important: true });
    setNewNote("");
  };

  return (
    <form onSubmit={addNote}>
      <input value={newNote} onChange={(e) => setNewNote(e.target.value)} />
      <button type="submit">save</button>
    </form>
  );
};
```

**The parent only provides the mutation handler:**

```jsx
// App
const addNote = (noteObject) => {
  noteService.create(noteObject).then((returnedNote) => {
    setNotes(notes.concat(returnedNote));
  });
};

const noteForm = () => (
  <Togglable buttonLabel="new note">
    <NoteForm createNote={addNote} />
  </Togglable>
);
```

**Rule:** keep state as low as possible, as high as necessary.

---

## 6. Component Refs & useImperativeHandle

To trigger a child's behavior from the parent **without lifting its state up**, use refs.

```jsx
import { useRef } from 'react'
const noteFormRef = useRef()

<Togglable buttonLabel="new note" ref={noteFormRef}>
  <NoteForm createNote={addNote} />
</Togglable>
```

**Inside `Togglable`**, expose specific behaviors via `useImperativeHandle`:

```jsx
import { useState, useImperativeHandle } from "react";

const Togglable = ({ buttonLabel, children, ref }) => {
  const [visible, setVisible] = useState(false);

  const toggleVisibility = () => setVisible(!visible);

  useImperativeHandle(ref, () => ({ toggleVisibility }));

  // ...
};
```

**Call from parent:**

```jsx
noteFormRef.current.toggleVisibility();
```

**Notes:**

- A ref remains tied to the same component instance across re-renders.
- Each `<Togglable>` instance with its own ref keeps its own state — no cross-talk.
- This is the one area where class components were arguably cleaner than function components.

---

## 7. ESLint for Frontend

Vite scaffolds ESLint by default. Configure `eslint.config.js`:

```js
import js from "@eslint/js";
import globals from "globals";
import reactHooks from "eslint-plugin-react-hooks";
import reactRefresh from "eslint-plugin-react-refresh";
import eslintConfigPrettier from "eslint-config-prettier";

export default [
  { ignores: ["dist"] },
  {
    files: ["**/*.{js,jsx}"],
    languageOptions: {
      ecmaVersion: 2020,
      globals: globals.browser,
      parserOptions: {
        ecmaVersion: "latest",
        ecmaFeatures: { jsx: true },
        sourceType: "module",
      },
    },
    plugins: {
      "react-hooks": reactHooks,
      "react-refresh": reactRefresh,
    },
    rules: {
      ...js.configs.recommended.rules,
      ...reactHooks.configs.recommended.rules,
      "no-unused-vars": ["error", { varsIgnorePattern: "^[A-Z_]" }],
      "react-refresh/only-export-components": [
        "warn",
        { allowConstantExport: true },
      ],
      indent: ["error", 2],
      "linebreak-style": ["error", "unix"],
      quotes: ["error", "single"],
      semi: ["error", "never"],
      eqeqeq: "error",
      "no-trailing-spaces": "error",
      "object-curly-spacing": ["error", "always"],
      "arrow-spacing": ["error", { before: true, after: true }],
      "no-console": "off",
    },
  },
  {
    files: ["**/*.test.{js,jsx}"],
    languageOptions: { globals: { ...globals.vitest } },
  },
  eslintConfigPrettier, // must be last
];
```

**`eslintConfigPrettier`** disables rules that conflict with Prettier formatting.

---

## 8. Vitest + React Testing Library Setup

```bash
npm install --save-dev vitest jsdom @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

```json
// package.json scripts
{
  "scripts": { "test": "vitest run" }
}
```

```js
// testSetup.js
import { afterEach } from "vitest";
import { cleanup } from "@testing-library/react";
import "@testing-library/jest-dom/vitest";

afterEach(() => {
  cleanup();
});
```

```js
// vite.config.js
export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    globals: true, // no need to import describe/test/expect
    setupFiles: "./testSetup.js",
  },
});
```

**Test file placement** — two conventions:

- **Co-located:** `src/components/Note.test.jsx`
- **Separate:** `src/tests/Note.test.jsx` (course preference)

---

## 9. Basic Component Tests

```jsx
import { render, screen } from "@testing-library/react";
import Note from "./Note";

test("renders content", () => {
  const note = {
    content: "Component testing is done with react-testing-library",
    important: true,
  };

  render(<Note note={note} />);

  const element = screen.getByText(
    "Component testing is done with react-testing-library",
  );
  expect(element).toBeDefined();
});
```

**Key facts:**

- `render` renders into a virtual DOM (jsdom), not the browser.
- `screen` is the entry point for querying that virtual DOM.
- If `getByText` fails, the test fails automatically — no need for `expect(...).toBeDefined()`.

---

## 10. Query Methods (getBy, findBy, queryBy)

| Method                              | Behavior                                              | Returns          |
| ----------------------------------- | ----------------------------------------------------- | ---------------- |
| `getByText(text)`                   | Finds exactly one matching element; throws if missing | Element          |
| `getByText(text, { exact: false })` | Substring match                                       | Element          |
| `findByText(text)`                  | Async — waits for element to appear                   | Promise<Element> |
| `queryByText(text)`                 | Returns `null` if not found (no throw)                | Element \| null  |
| `getByRole(role, { name })`         | By ARIA role (button, textbox, …)                     | Element          |
| `getByLabelText('label')`           | By associated `<label>`                               | Element          |
| `getByPlaceholderText('...')`       | By placeholder                                        | Element          |
| `getByTestId(id)`                   | By `data-testid` attribute                            | Element          |
| `getAllByRole(role)`                | Array of all matches                                  | Element[]        |

**Prefer user-facing queries** (text, role, label) over CSS selectors. CSS is subject to refactoring and yields brittle tests.

**`container.querySelector`** — only as a last resort:

```jsx
const { container } = render(<Note note={note} />);
const div = container.querySelector(".note");
expect(div).toHaveTextContent(
  "Component testing is done with react-testing-library",
);
```

**Debug helpers:**

```jsx
screen.debug(); // prints whole render
screen.debug(element); // prints one element
```

---

## 11. Event Simulation with user-event

```bash
npm install --save-dev @testing-library/user-event
```

```jsx
import userEvent from "@testing-library/user-event";

const mockHandler = vi.fn();
render(<Note note={note} toggleImportance={mockHandler} />);

const user = userEvent.setup();
const button = screen.getByText("make not important");
await user.click(button);

expect(mockHandler.mock.calls).toHaveLength(1);
```

**Notes:**

- `vi.fn()` creates a mock function that records calls in `mock.calls`.
- Always call `userEvent.setup()` before interacting.
- Prefer `expect(mock).toHaveBeenCalledTimes(2)` over inspecting `mock.calls.length`.

---

## 12. Testing Togglable Components

```jsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import Togglable from "./Togglable";

describe("<Togglable />", () => {
  beforeEach(() => {
    render(
      <Togglable buttonLabel="show...">
        <div>togglable content</div>
      </Togglable>,
    );
  });

  test("renders its children", () => {
    screen.getByText("togglable content");
  });

  test("at start the children are not displayed", () => {
    const element = screen.getByText("togglable content");
    expect(element).not.toBeVisible();
  });

  test("after clicking the button, children are displayed", async () => {
    const user = userEvent.setup();
    await user.click(screen.getByText("show..."));

    expect(screen.getByText("togglable content")).toBeVisible();
  });

  test("toggled content can be closed", async () => {
    const user = userEvent.setup();
    await user.click(screen.getByText("show..."));
    await user.click(screen.getByText("cancel"));

    expect(screen.getByText("togglable content")).not.toBeVisible();
  });
});
```

**Patterns:** `beforeEach` renders; `not.toBeVisible()` (not `not.toBeDefined()`) is the correct assertion — the element exists in the DOM but is hidden.

---

## 13. Testing Forms

```jsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import NoteForm from "./NoteForm";

test("<NoteForm /> updates parent state and calls onSubmit", async () => {
  const createNote = vi.fn();
  const user = userEvent.setup();

  render(<NoteForm createNote={createNote} />);

  const input = screen.getByRole("textbox");
  const sendButton = screen.getByText("save");

  await user.type(input, "testing a form...");
  await user.click(sendButton);

  expect(createNote).toHaveBeenCalledTimes(1);
  expect(createNote.mock.calls[0][0].content).toBe("testing a form...");
});
```

**Preferred assertions:**

```jsx
expect(createNote).toHaveBeenCalledWith({ content: "testing a form..." });
```

**Input query shortcuts:** `getByRole('textbox')`, `getByLabelText('content')`, `getByPlaceholderText('...')`.

---

## 14. Coverage & Snapshots

```bash
npm test -- --coverage
# On first run, Vitest offers to install @vitest/coverage-v8
```

Produces a `coverage/` directory with per-file line/branch coverage.

Add to `.gitignore`:

```text
coverage/
```

**Snapshot testing** — Vitest captures the component's rendered HTML and compares it on subsequent runs. Useful for detecting unintended DOM changes; weak for verifying behavior. Prefer explicit assertions.

---

## 15. End-to-End Testing with Playwright

**E2E tests exercise the full system** (frontend + backend + DB) through a real browser.

**Tools:** Playwright (out-of-browser Node driver) vs Cypress (runs inside the browser). Course uses Playwright.

**Trade-offs of E2E:** hardest to write, slowest to run (hours on real suites), occasionally flaky. But they catch integration bugs unit tests can't.

**Setup:**

```bash
mkdir e2e-tests && cd e2e-tests
npm init -y
npm init playwright@latest
# JS, "tests", add GitHub Actions: yes, install browsers: yes
```

**Limit projects in `playwright.config.js`** if some browsers aren't supported:

```js
projects: [
  { name: "chromium", use: { ...devices["Desktop Chrome"] } },
  { name: "firefox", use: { ...devices["Desktop Firefox"] } },
  // { name: 'webkit', ... }   // comment out if unsupported
];
```

**Scripts:**

```json
{
  "scripts": {
    "test": "playwright test",
    "test:report": "playwright show-report"
  }
}
```

**Disable parallelism** for apps backed by a single DB:

```js
export default defineConfig({
  timeout: 3000,
  fullyParallel: false,
  workers: 1,
});
```

---

## 16. Playwright Locators & Assertions

```js
const { test, expect } = require("@playwright/test");

test("front page can be opened", async ({ page }) => {
  await page.goto("http://localhost:5173");

  await expect(page.getByText("Notes")).toBeVisible();
  await expect(page.getByText("Note app, ...")).toBeVisible();
});
```

**Common locators:**

| Locator                                       | Purpose                    |
| --------------------------------------------- | -------------------------- |
| `page.getByRole('button', { name: 'login' })` | By accessible role + name  |
| `page.getByText('...')`                       | By visible text            |
| `page.getByLabel('username')`                 | By associated label        |
| `page.getByTestId('...')`                     | By `data-testid`           |
| `page.locator('.error')`                      | CSS selector (last resort) |

**Handling multiple matches:**

```js
// First / last
await page.getByRole("textbox").first().fill("mluukkai");
await page.getByRole("textbox").last().fill("salainen");

// All as array
const textboxes = await page.getByRole("textbox").all();
await textboxes[0].fill("mluukkai");
await textboxes[1].fill("salainen");
```

**Preferred approach: `getByLabel`** — labels are unique by nature.

**Assertion examples:**

```js
await expect(page.getByText("wrong credentials")).toBeVisible();
await expect(page.locator(".error")).toContainText("wrong credentials");
await expect(page.locator(".error")).toHaveCSS("border-style", "solid");
await expect(page.locator(".error")).toHaveCSS("color", "rgb(255, 0, 0)");
await expect(page.getByText("Matti logged in")).not.toBeVisible();
```

**Skip flaky waiters — prefer web-first assertions:**

- `page.waitForSelector()` is discouraged in SPA tests (React/Vue). Prefer `expect(...)` which auto-retries until timeout.
- Use `await page.getByText(content).waitFor()` only when you need to guarantee data has arrived before the next step.

**Dialog handling** (alert/confirm/prompt) — register **before** the trigger:

```js
page.on("dialog", (dialog) => dialog.accept());
```

**Accessibility-friendly locators include `getByRole` — this improves the app, not just the test.**

---

## 17. Test Database Reset Endpoint

E2E tests need a clean DB per run. Add a **test-only** endpoint:

```js
// controllers/testing.js
const router = require("express").Router();
const Note = require("../models/note");
const User = require("../models/user");

router.post("/reset", async (req, res) => {
  await Note.deleteMany({});
  await User.deleteMany({});
  res.status(204).end();
});

module.exports = router;
```

```js
// app.js — register only in test mode
if (process.env.NODE_ENV === "test") {
  const testingRouter = require("./controllers/testing");
  app.use("/api/testing", testingRouter);
}
```

**Backend test script:**

```json
"start:test": "cross-env NODE_ENV=test node --watch index.js"
```

**Reset + seed before each Playwright test:**

```js
describe("Note app", () => {
  beforeEach(async ({ page, request }) => {
    await request.post("http://localhost:3001/api/testing/reset");
    await request.post("http://localhost:3001/api/users", {
      data: {
        name: "Matti Luukkainen",
        username: "mluukkai",
        password: "salainen",
      },
    });
    await page.goto("http://localhost:5173");
  });
  // ...
});
```

**Cleaner setup — set `baseURL` in `playwright.config.js`:**

```js
export default defineConfig({
  use: { baseURL: "http://localhost:5173" },
});
```

Then use `/api/testing/reset`, `/`, etc.

---

## 18. Playwright Helpers

Extract repeated interaction sequences:

```js
// e2e-tests/helper.js
const loginWith = async (page, username, password) => {
  await page.getByRole("button", { name: "login" }).click();
  await page.getByLabel("username").fill(username);
  await page.getByLabel("password").fill(password);
  await page.getByRole("button", { name: "login" }).click();
};

const createNote = async (page, content) => {
  await page.getByRole("button", { name: "new note" }).click();
  await page.getByRole("textbox").fill(content);
  await page.getByRole("button", { name: "save" }).click();
  await page.getByText(content).waitFor();
};

module.exports = { loginWith, createNote };
```

**Nested `describe` with shared `beforeEach`** keeps scenarios focused:

```js
describe("when logged in", () => {
  beforeEach(async ({ page }) => {
    await loginWith(page, "mluukkai", "salainen");
  });

  test("a new note can be created", async ({ page }) => {
    await createNote(page, "a note created by playwright");
    await expect(page.getByText("a note created by playwright")).toBeVisible();
  });
});
```

---

## 19. Playwright Debugging

| Tool                                        | Purpose                                          |
| ------------------------------------------- | ------------------------------------------------ |
| `npm test -- -g "test name"`                | Run a single test by name                        |
| `npm test -- --project chromium`            | Run against one browser                          |
| `npm test -- --ui`                          | Visual test runner                               |
| `--debug` flag                              | Step-through with inspector                      |
| `await page.pause()`                        | Pause mid-test and open inspector                |
| `npx playwright codegen <url>`              | Record browser actions → auto-generate test code |
| `--trace on` + `npx playwright show-report` | Post-mortem trace of a failed run                |
| VS Code Playwright extension                | Locator picker, test explorer                    |

**`test.only`** — run one test. **Always remove before committing.**

---

## 20. React Router

```bash
npm install react-router-dom
```

**Basic setup — move routing into `main.jsx` so `App` isn't the router root:**

```jsx
// main.jsx
import { BrowserRouter as Router } from "react-router-dom";

ReactDOM.createRoot(document.getElementById("root")).render(
  <Router>
    <App />
  </Router>,
);
```

```jsx
// App.jsx
import {
  Routes,
  Route,
  Link,
  useParams,
  useMatch,
  useNavigate,
} from "react-router-dom";

const App = () => (
  <div>
    <Link to="/">home</Link>
    <Link to="/notes">notes</Link>
    <Link to="/create">new note</Link>

    <Routes>
      <Route path="/notes" element={<NoteList notes={notes} />} />
      <Route path="/notes/:id" element={<NoteDetail notes={notes} />} />
      <Route path="/create" element={<NoteForm createNote={addNote} />} />
      <Route path="/" element={<Home />} />
      <Route path="*" element={<NotFound />} />
    </Routes>
  </div>
);
```

**Key concepts:**

- `<Link>` produces internal navigation; the server is never reloaded.
- `<Route>` matches a path and renders its `element`.
- `:id` is a route parameter.

**`useParams`** — read path parameters:

```jsx
const id = useParams().id;
const note = notes.find((n) => n.id === id);
```

Passing all notes to every detail route is wasteful; prefer `useMatch`.

**`useMatch`** — access path params inside `App` and pass only the matched resource:

```jsx
const match = useMatch("/notes/:id");
const note = match ? notes.find((n) => n.id === match.params.id) : null;
```

**Guard against rendering before data loads:**

```jsx
if (!note) return null;
```

**`useNavigate`** — programmatic redirect (only usable inside a Router):

```jsx
const navigate = useNavigate();
const handleDelete = () => {
  if (window.confirm(`Delete note "${note.content}"?`)) {
    deleteNote(id);
    navigate("/notes");
  }
};
```

**Testing wrappers:** any component that uses router hooks must be rendered inside a `BrowserRouter`:

```jsx
const renderWithRouter = (component) =>
  render(<BrowserRouter>{component}</BrowserRouter>);
```

---

## 21. MaterialUI

```bash
npm install @mui/material @emotion/react @emotion/styled
```

**Wrap the app in `<Container>`:**

```jsx
import { Container } from "@mui/material";

const App = () => <Container>{/* ... */}</Container>;
```

**Table:**

```jsx
import {
  Table,
  TableBody,
  TableCell,
  TableContainer,
  TableHead,
  TableRow,
  Paper,
} from "@mui/material";

<TableContainer component={Paper}>
  <Table>
    <TableHead>
      <TableRow>
        <TableCell>content</TableCell>
        <TableCell>user</TableCell>
        <TableCell>important</TableCell>
      </TableRow>
    </TableHead>
    <TableBody>
      {notes.map((note) => (
        <TableRow key={note.id}>
          <TableCell>
            <Link to={`/notes/${note.id}`}>{note.content}</Link>
          </TableCell>
          <TableCell>{note.user.name}</TableCell>
          <TableCell>{note.important ? "yes" : ""}</TableCell>
        </TableRow>
      ))}
    </TableBody>
  </Table>
</TableContainer>;
```

**Form:**

```jsx
import { TextField, Button } from '@mui/material'

<TextField label="note content" value={newNote} onChange={(e) => setNewNote(e.target.value)} />
<Button type="submit" variant="contained">save</Button>
```

**Alert / Notification:**

```jsx
import { Alert } from "@mui/material";

const Notification = ({ notification }) => {
  if (!notification) return null;
  return (
    <Alert
      severity={notification.type}
      style={{ marginTop: 10, marginBottom: 10 }}
    >
      {notification.text}
    </Alert>
  );
};
```

- `severity`: `'success' | 'error' | 'warning' | 'info'`.

**AppBar (nav bar):**

```jsx
import { AppBar, Toolbar, Button } from '@mui/material'
import { Link } from 'react-router-dom'

const style = { '&:hover': { bgcolor: 'rgba(255,255,255,0.3)' } }

<AppBar position="static">
  <Toolbar>
    <Button color="inherit" component={Link} to="/" sx={style}>home</Button>
    <Button color="inherit" component={Link} to="/notes" sx={style}>notes</Button>
    <Button color="inherit" component={Link} to="/create" sx={style}>new note</Button>
  </Toolbar>
</AppBar>
```

- `position="static"` keeps the bar at the top of the page.
- `sx` accepts responsive, theme-aware styles — equivalent to inline styles but richer.

---

## 22. styled-components

Alternative styling approach — CSS-in-JS via tagged template literals.

```bash
npm install styled-components
```

```jsx
import styled from "styled-components";

const Button = styled.button`
  background: Bisque;
  font-size: 1em;
  margin: 1em;
  padding: 0.25em 1em;
  border: 2px solid Chocolate;
  border-radius: 3px;
`;

const Input = styled.input`
  margin: 0.25em;
  width: 300px;
`;

const Page = styled.div`
  padding: 1em;
  background: papayawhip;
`;
```

Every `<Button>`, `<Input>`, `<Page>` in the app uses these styles. Styles are scoped to the styled component — no class name collisions.

---

## 23. Key Takeaways

| Principle           | Rule                                                                       |
| ------------------- | -------------------------------------------------------------------------- |
| Auth flow           | Login → token in state → `service.setToken()` → persist to localStorage.   |
| Persist safely      | localStorage stores strings only; parse/stringify explicitly.              |
| `props.children`    | Auto-passed array of elements between component tags.                      |
| Lifting state       | Move state to the nearest common ancestor.                                 |
| Refs                | `useRef` + `useImperativeHandle` to expose child methods to parent.        |
| Frontend tests      | Vitest + RTL + jsdom + user-event; test as the user would.                 |
| Query priority      | `getByRole` > `getByLabelText` > `getByText` > CSS.                        |
| Async queries       | `findBy*` returns a Promise; `queryBy*` for absence.                       |
| Event simulation    | `userEvent.setup()` then `await user.click(...)` / `user.type(...)`.       |
| Coverage            | `vitest run --coverage` → add `coverage/` to gitignore.                    |
| E2E                 | Playwright + real browser; slowest but highest confidence.                 |
| E2E DB reset        | Add `/api/testing/reset` under `NODE_ENV === 'test'`.                      |
| Playwright locators | Prefer `getByLabel` and `getByRole`; `getByTestId` for edge cases.         |
| Waiters             | Prefer web-first assertions over `waitForSelector`.                        |
| Dialogs             | Register `page.on('dialog', ...)` before the trigger.                      |
| React Router        | `<Link>` navigates; `<Route>` matches; `useMatch` reads params inside App. |
| Router in tests     | Wrap tested components in `<BrowserRouter>`.                               |
| MUI                 | Wrap in `<Container>`; use `sx` for scoped styling.                        |
| styled-components   | Template-literal CSS-in-JS; component-scoped styles.                       |

> _"Write tests that resemble the way software is used."_ — Kent C. Dodds
