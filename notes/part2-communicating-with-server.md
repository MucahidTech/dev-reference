# Part 2 — Communicating with the Server

> Reference for rendering collections, controlled forms, fetching/mutating server data, service modules, error handling, and styling in React.
> Assumes prior knowledge. Focus: correct patterns, API semantics, and common pitfalls.

## 📑 Index

1. [Rendering Collections](#1-rendering-collections)
2. [Array Methods for React](#2-array-methods-for-react)
3. [Module Structure](#3-module-structure)
4. [Forms & Controlled Inputs](#4-forms--controlled-inputs)
5. [Filtering Displayed Data](#5-filtering-displayed-data)
6. [Fetching Data: JSON Server + Axios](#6-fetching-data-json-server--axios)
7. [Promises](#7-promises)
8. [useEffect](#8-useeffect)
9. [Mutating Server Data (POST / PUT / PATCH / DELETE)](#9-mutating-server-data-post--put--patch--delete)
10. [Service Modules (API Layer)](#10-service-modules-api-layer)
11. [Error Handling](#11-error-handling)
12. [Styling](#12-styling)
13. [State Initialization Gotcha](#13-state-initialization-gotcha)
14. [Key Takeaways](#14-key-takeaways)

---

## 1. Rendering Collections

React renders arrays of elements by mapping data to JSX. Every generated element **must have a stable, unique `key`**.

```jsx
<ul>
  {notes.map((note) => (
    <Note key={note.id} note={note} />
  ))}
</ul>
```

**Rules & rationale:**

- `key` must be **unique among siblings** and **stable across renders**.
- **Never use array index as key** unless the list is static and never reordered. Index keys break reconciliation when items are inserted/deleted/reordered.
- `key` belongs to the **outermost element returned by `map`**, not necessarily a native `<li>` — for a component `<Note />`, the key sits on the component.

---

## 2. Array Methods for React

| Method              | Returns                | Mutates? | Use case in React                            |
| ------------------- | ---------------------- | -------- | -------------------------------------------- |
| `.map(fn)`          | New array              | ❌       | Render lists                                 |
| `.filter(fn)`       | New array              | ❌       | Conditional rendering                        |
| `.find(fn)`         | Element or `undefined` | ❌       | Locate one item by id                        |
| `.reduce(fn, init)` | Single value           | ❌       | Sums, aggregations                           |
| `.some(fn)`         | `boolean`              | ❌       | Existence check                              |
| `.includes(x)`      | `boolean`              | ❌       | Primitive membership (array or string)       |
| `.concat(x)`        | New array              | ❌       | **Preferred** for state append               |
| `[...arr, x]`       | New array              | ❌       | Same as `concat`, idiomatic                  |
| `.push(x)`          | Length                 | ✅       | ❌ Never on state                            |
| `.unshift(x)`       | Length                 | ✅       | ❌ Never on state                            |
| `.splice(i, 0, x)`  | Removed items          | ✅       | ❌ Never on state                            |
| `.sort(cmp)`        | Same array (sorted)    | ✅       | ❌ For state — copy first: `[...arr].sort()` |

**Common canonical forms:**

```jsx
const total = parts.reduce((sum, p) => sum + p.exercises, 0);

const filtered = persons.filter((p) =>
  p.name.toLowerCase().includes(search.toLowerCase()),
);

const sortedDesc = [...blogs].sort((a, b) => b.likes - a.likes);
```

**Equality in filters:** prefer `===` (strict) over `==`.

---

## 3. Module Structure

Split components into their own files under `src/components/`. Each file exports default.

```jsx
// src/components/Note.jsx
const Note = ({ note }) => <li>{note.content}</li>;
export default Note;

// Usage
import Note from "./components/Note";
```

- Relative paths are required for local imports.
- `.jsx` extension may be omitted.
- **Do not define components inside other components.**

**Debugging first-step:** add `console.log('App works...')` at the top of `App` to confirm the app renders at all. Then log deeper, component by component. Common root causes: wrong prop type or a prop name mismatch — check with `console.log(props)`.

---

## 4. Forms & Controlled Inputs

A **controlled input** has its `value` bound to state and its `onChange` synchronized back.

```jsx
const App = (props) => {
  const [notes, setNotes] = useState(props.notes);
  const [newNote, setNewNote] = useState("");

  const addNote = (event) => {
    event.preventDefault(); // stop default full-page submit
    const noteObject = {
      content: newNote,
      important: Math.random() > 0.5,
      id: String(notes.length + 1),
    };
    setNotes(notes.concat(noteObject));
    setNewNote("");
  };

  const handleNoteChange = (event) => {
    setNewNote(event.target.value);
  };

  return (
    <form onSubmit={addNote}>
      <input value={newNote} onChange={handleNoteChange} />
      <button type="submit">save</button>
    </form>
  );
};
```

**Key points:**

- `event.preventDefault()` is required in `onSubmit` to avoid a full page reload.
- The `onChange` handler does **not** need `preventDefault`.
- Never mutate state — use `concat` or `[...state, item]`.

---

## 5. Filtering Displayed Data

Derive the displayed list in the render body — do not store it as separate state.

```jsx
const notesToShow = showAll
? notes
: notes.filter(note => note.important)

// ...

<ul>
  {notesToShow.map(note => <Note key={note.id} note={note} />)}
</ul>
<button onClick={() => setShowAll(!showAll)}>
  show {showAll ? 'important' : 'all'}
</button>
```

Short inline arrow handlers are fine for one-line toggles.

---

## 6. Fetching Data: JSON Server + Axios

### JSON Server (mock backend)

```bash
npm install json-server --save-dev

# db.json at project root

{
"notes": [
{ "id": "1", "content": "HTML is easy", "important": true }
]
}

# package.json scripts

"server": "json-server -p 3001 db.json"

npm run server # http://localhost:3001/notes
```

### Axios

```bash
npm install axios
```

---

## 7. Promises

A **Promise** represents the eventual completion or failure of an asynchronous operation.

| State       | Meaning               |
| ----------- | --------------------- |
| `pending`   | Not yet settled       |
| `fulfilled` | Resolved with a value |
| `rejected`  | Failed with a reason  |

```jsx
axios
  .get("http://localhost:3001/notes")
  .then((response) => {
    const notes = response.data;
    // ...
  })
  .catch((error) => {
    // handle failure
  });
```

`axios.get` returns a promise. Chain `.then` for success and `.catch` for failure — no need to store the promise in a variable.

---

## 8. useEffect

`useEffect` connects a component to external systems (network, DOM, timers).

```jsx
useEffect(() => {
  noteService.getAll().then((initialNotes) => {
    setNotes(initialNotes);
  });
}, []);
```

**Signature:** `useEffect(fn, deps)`

| `deps` value | Effect runs when…                               |
| ------------ | ----------------------------------------------- |
| `[]`         | After the first render only                     |
| `[a, b]`     | After first render + whenever `a` or `b` change |
| omitted      | After every render                              |

- Running the effect and calling its setter triggers a re-render — that's how data flows into the UI.
- **Rules of Hooks:** `useEffect` and `useState` may only be called at the top level of a React function component (never in loops, conditions, or nested functions).

---

## 9. Mutating Server Data (POST / PUT / PATCH / DELETE)

### POST — create

```jsx
axios.post("http://localhost:3001/notes", noteObject).then((response) => {
  setNotes(notes.concat(response.data));
});
```

- The server generates the `id` — let it.
- Axios auto-sets `Content-Type: application/json` for JS objects.
- Update local state immediately from `response.data` instead of re-fetching.

### PUT vs PATCH — update

- **PUT** replaces the entire resource.
- **PATCH** updates specific fields.

```jsx
const toggleImportanceOf = (id) => {
  const url = `http://localhost:3001/notes/${id}`;
  const note = notes.find((n) => n.id === id);
  const changedNote = { ...note, important: !note.important };

  noteService.update(id, changedNote).then((returnedNote) => {
    setNotes(notes.map((n) => (n.id === id ? returnedNote : n)));
  });
};
```

**Why create a new object with `{ ...note, important: ... }`?** Because the note lives inside the state array — directly mutating it violates React immutability.

### DELETE

```js
const remove = (id) => {
  const request = axios.delete(`${baseUrl}/${id}`);
  return request.then((response) => response.data);
};
```

- `delete` is a reserved word — never use it as a variable name.
- DELETE returns no body; pass the id in the URL.

---

## 10. Service Modules (API Layer)

Extract all HTTP calls into `src/services/<resource>.js`. One module per resource.

```js
// src/services/notes.js
import axios from "axios";
const baseUrl = "http://localhost:3001/notes";

const getAll = () => {
  const request = axios.get(baseUrl);
  return request.then((response) => response.data);
};

const create = (newObject) => {
  const request = axios.post(baseUrl, newObject);
  return request.then((response) => response.data);
};

const update = (id, newObject) => {
  const request = axios.put(`${baseUrl}/${id}`, newObject);
  return request.then((response) => response.data);
};

const remove = (id) => {
  const request = axios.delete(`${baseUrl}/${id}`);
  return request.then((response) => response.data);
};

export default { getAll, create, update, remove };
```

**Shorthand:** when keys and values match, `export default { getAll, create, update, remove }` is enough.

**Consuming the service:**

```jsx
import noteService from "./services/notes";

useEffect(() => {
  noteService.getAll().then((initialNotes) => setNotes(initialNotes));
}, []);

const addNote = (event) => {
  event.preventDefault();
  noteService
    .create({ content: newNote, important: Math.random() > 0.5 })
    .then((returnedNote) => {
      setNotes(notes.concat(returnedNote));
      setNewNote("");
    });
};
```

---

## 11. Error Handling

Attach `.catch` at the **end of a promise chain** — it catches any rejection in the chain.

```jsx
noteService
  .update(id, changedNote)
  .then((returnedNote) => {
    setNotes(notes.map((n) => (n.id === id ? returnedNote : n)));
  })
  .catch((error) => {
    alert(`The note '${note.content}' was already deleted from server`);
    setNotes(notes.filter((n) => n.id !== id));
  });
```

`catch` is not only for reporting — it's a legitimate place to **reconcile UI state** with server truth.

---

## 12. Styling

### Global CSS

```jsx
// src/main.jsx or src/App.jsx
import "./index.css";
```

Use `className` (not `class`) in JSX.

### Notification Component Pattern

```jsx
// src/components/Notification.jsx
const Notification = ({ message }) => {
  if (message === null) return null;
  return <div className="error">{message}</div>;
};
export default Notification;
```

```jsx
// In App
const [errorMessage, setErrorMessage] = useState(null)

// On failure
setErrorMessage(`Note '${note.content}' was already removed from server`)
setTimeout(() => setErrorMessage(null), 5000)

// In render
<Notification message={errorMessage} />
```

```css
.error {
  color: red;
  background: lightgrey;
  font-size: 20px;
  border-style: solid;
  border-radius: 5px;
  padding: 10px;
  margin-bottom: 10px;
}
```

### Inline Styles

```jsx
const footerStyle = {
color: 'green',
fontStyle: 'italic',
}

<div style={footerStyle}>...</div>
```

- Values must be quoted strings (or numbers for pixel values).
- Multi-word CSS properties use **camelCase** (`fontStyle`, not `font-style`).
- Inline styles can clash with class-based rules; use sparingly.

**React philosophy:** co-locate structure, behavior, and styling inside the component — the opposite of the older "separate HTML/CSS/JS" school.

---

## 13. State Initialization Gotcha

**Match the initial state type to the value type it will hold.**

```jsx
const [notes, setNotes] = useState(null); // ❌
// later: notesToShow.map(...) → crash: Cannot read 'map' of null

const [notes, setNotes] = useState([]); // ✅
// map on an empty array is safe
```

The empty-array initializer **masks** the "data not yet fetched" state. If you truly cannot define a safe initial value, gate rendering:

```jsx
if (!notes) return null;
```

This is the fallback pattern when the initial shape is genuinely unknown.

---

## 14. Key Takeaways

| Principle              | Rule                                                             |
| ---------------------- | ---------------------------------------------------------------- |
| Single source of truth | Keep state at the highest common ancestor; pass down via props.  |
| Presentation vs logic  | Child components display; parents coordinate.                    |
| Immutability           | Never mutate state; use `concat`, `[...arr]`, `{ ...obj }`.      |
| Service layer          | All HTTP calls live in `src/services/*.js`.                      |
| Promises               | `.then` for success, `.catch` for errors at the chain's end.     |
| useEffect              | `[]` = run once on mount; `[dep]` = run on dep change.           |
| Keys                   | Unique + stable; never array index on dynamic lists.             |
| Controlled inputs      | `value` bound to state, `onChange` updates it.                   |
| State type             | Initialize with the type you'll actually use (`[]`, not `null`). |
| CSS in JSX             | `className` for classes, camelCase for inline styles.            |

> _"The only way to go fast, is to go well."_ — Robert C. Martin
> _"The best way to debug is to keep the console open and log everything."_ — Part 2, Full Stack Open
