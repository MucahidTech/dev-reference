# Part 1 — Introduction to React

> Quick reference for React fundamentals, JSX, props, state, and event handling.
> Not a tutorial — assumes prior knowledge. Focus: correct patterns and gotchas.

## 📑 Index

1. [Project Setup (Vite)](#1-project-setup-vite)
2. [Components & JSX](#2-components--jsx)
3. [Props](#3-props)
4. [JavaScript Essentials for React](#4-javascript-essentials-for-react)
5. [State & Event Handling](#5-state--event-handling)
6. [Rules of Hooks](#6-rules-of-hooks)
7. [Debugging](#7-debugging)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. Project Setup (Vite)

Vite is the modern, fastest way to scaffold a React app.

```bash
npm create vite@latest        # scaffold project
cd <project>
npm install                   # install dependencies
npm run dev                   # start dev server (default port: 5173)
```

**Minimal React entry point** — keep `main.jsx` as thin as possible so the root of rendering stays obvious:

```jsx
// src/main.jsx
import ReactDOM from "react-dom/client";
import App from "./App";

ReactDOM.createRoot(document.getElementById("root")).render(<App />);
```

**Notes:**

- `index.html` contains only `<div id="root"></div>` — the mount point.
- `src/assets/`, `App.css`, `index.css` can be removed if unused (keep `index.css` for global styles).
- If port 5173 is busy, Vite picks the next available one.

---

## 2. Components & JSX

**Component = a JS function that returns JSX.** It can contain any JS logic, but only what it `return`s is rendered.

```jsx
const App = () => {
  const now = new Date();
  const a = 10;
  const b = 20;
  return (
    <div>
      <p>Hello, it is {now.toString()}</p>
      <p>
        {a} plus {b} is {a + b}
      </p>
    </div>
  );
};
export default App;
```

### JSX Rules

- **Every tag must be closed**, including void tags: `<br />`, `<img />`.
- **Must return a single root element.** Use a fragment `<>...</>` to avoid extra DOM nodes.
- **JS inside JSX must be wrapped in `{}`**. Only primitive values (string, number) can be rendered directly inside braces.
- **Objects cannot be rendered directly** → `<p>{obj}</p>` throws `Objects are not valid as a React child`.

```jsx
// ❌ Wrong — object rendered directly
<p>{friends[0]}</p>

// ✅ Correct — render its primitive fields
<p>{friends[0].name} {friends[0].age}</p>
```

### Component Naming

- **First letter must be uppercase.** Lowercase is treated as a native HTML element.
- **Never define a component inside another component** → it creates a new component identity on every render, breaking reconciliation and hooks.

### Module Exports

- Each component file must `export default ComponentName`.
- Multiple small components can live in the same file if only used locally.

### How JSX Is Compiled

Babel transpiles JSX into nested `React.createElement(...)` calls. Curly braces become function arguments. Vanilla JS is possible but never recommended.

---

## 3. Props

Props are how data flows **parent → child** (unidirectional).

```jsx
const Hello = ({ name, age }) => (
  <div>
    <p>
      Hello {name}, you are {age} years old
    </p>
  </div>
);

const App = () => {
  const name = "Peter";
  const age = 10;
  return (
    <div>
      <Hello name="Maya" age={26 + 10} />
      <Hello name={name} age={age} />
    </div>
  );
};
```

**Notes:**

- Props values must be wrapped in `{}` when they come from JS.
- **Destructuring in the parameter list** is the idiomatic way: `({ name, age }) => ...`.
- Debugging: `const Hello = (props) => { console.log(props); ... }` to inspect the shape of props.
- **Naming convention for event props:** `onSomething` (e.g., `onClick`, `onSubmit`); the handler function is `handleSomething`.

---

## 4. JavaScript Essentials for React

### Arrays

| Method                  | Behavior                                               |
| ----------------------- | ------------------------------------------------------ |
| `.push(x)`              | Mutates array, adds to end ❌ (never use on state)     |
| `.concat(x)`            | Returns **new** array with `x` appended ✅             |
| `[...arr, x]`           | Same as `concat`, idiomatic ✅                         |
| `.map(fn)`              | Returns **new** array of transformed elements ✅       |
| `.filter(fn)`           | Returns new array of elements passing the predicate ✅ |
| `.forEach(fn)`          | Side-effect iteration, returns `undefined`             |
| `.join(sep)`            | Joins elements into one string                         |
| `[a, b, ...rest] = arr` | Destructuring assignment                               |

**Note:** `const` prevents reassignment of the binding, not mutation of contents. `arr.push()` works but is **forbidden for React state** — always create a new array.

### Objects

```js
const obj = {
  name: { first: "Dan", last: "Abramov" },
  grades: [2, 3, 5, 3],
  size: 5,
};

obj.name.first; // dot access
obj["name"]; // bracket access
obj["secret number"] = 1; // brackets required for invalid identifiers
obj.address = "Helsinki"; // add new property on the fly

const { first, last } = obj.name; // destructuring
```

### Functions

```js
// Arrow — preferred in React
const sum = (a, b) => a + b;
const square = (p) => p * p; // single param, single expression

// Old-style
function product(a, b) {
  return a * b;
}
const avg = function (a, b) {
  return (a + b) / 2;
};
```

**`this` pitfall:** Arrow functions do not bind `this`. Use `.bind(obj)` when passing methods as callbacks (e.g., `setTimeout(arto.greet.bind(arto), 1000)`). In React, arrow functions remove the need for `.bind` in most cases.

### Classes (Legacy)

JavaScript "classes" are syntactic sugar over prototype-based inheritance. All instances are `typeof === 'object'`. **Not used in modern React** (hooks replaced class components).

### Useful Built-ins

```js
new Date().getFullYear();
```

---

## 5. State & Event Handling

### `useState`

```jsx
import { useState } from "react";

const Counter = () => {
  const [counter, setCounter] = useState(0);
  return (
    <div>
      <p>{counter}</p>
      <button onClick={() => setCounter(counter + 1)}>+1</button>
    </div>
  );
};
```

**Key rules:**

- Two-element array: `[value, setter]`.
- Calling the setter **triggers a re-render**.
- **The re-render uses the old value first**; the new value is visible on the _next_ render. Log inside a `useEffect` or the render body to observe.

### Event Handlers

- `onClick`, `onChange`, `onSubmit`, etc.
- **Handler must be a function (or a reference to one).**

```jsx
// ❌ Runs immediately → infinite re-render
<button onClick={setCounter(counter + 1)}>

// ✅ Correct
<button onClick={() => setCounter(counter + 1)}>
<button onClick={handleClick}>
```

- **Naming:** props that carry handlers are `onX`; the functions that implement them are `handleX`.

### Passing State Down

Extract a reusable button:

```jsx
const Button = ({ onClick, text }) => <button onClick={onClick}>{text}</button>;
```

The prop name is arbitrary when the child is a custom component.

### Complex State (Objects / Arrays)

- **Never mutate state directly.** Always produce a new object/array.
- Use spread to preserve unchanged fields:

```jsx
const [clicks, setClicks] = useState({ left: 0, right: 0 });

const handleLeftClick = () => setClicks({ ...clicks, left: clicks.left + 1 });

const handleRightClick = () =>
  setClicks({ ...clicks, right: clicks.right + 1 });
```

- Arrays: use `concat` or spread, **never `push`**:

```jsx
const [allClicks, setAll] = useState([]);

const handleLeftClick = () => {
  setAll(allClicks.concat("L")); // ✅ new array
  // setAll(allClicks.push('L'))  // ❌ mutation
};
```

### "Function Returning Function" Pattern

When a handler must compute a value from current state at click time:

```jsx
const setToValue = (newValue) => () => {
  setValue(newValue)
}
// ...
<button onClick={setToValue(1000)}>thousand</button>
<button onClick={() => setToValue(0)}>reset</button>
```

Both approaches work; the arrow-wrapped version is more common in modern code.

---

## 6. Rules of Hooks

- **Only call hooks** (`useState`, `useEffect`, …) **at the top level of a React function component**.
- **Never** call hooks inside loops, conditions, or nested functions.
- **Never define components inside other components** — this re-creates them on every render.

---

## 7. Debugging

- **Keep the browser console open at all times.** First rule of web development.
- `console.log(value)` — prefer `console.log('label', obj)` over string concatenation to see full object contents.
- **`debugger`** — pauses execution in DevTools; also can be set via breakpoints in the Sources tab.
- **React DevTools** browser extension — inspect component tree, props, state, and hooks.
- Log inside a component body to see the render order and current prop/state values.

---

## 8. Key Takeaways

| Principle           | Rule                                                                   |
| ------------------- | ---------------------------------------------------------------------- |
| Immutability        | Never mutate state directly; always create new objects/arrays.         |
| Single root         | JSX must return one root element (use fragments to avoid extra DOM).   |
| Hooks placement     | Top level of function components only.                                 |
| Component placement | Define components at module top level, never inside another component. |
| Event handlers      | Must be functions or references; never call them during render.        |
| Console             | Keep it open; log early, log often.                                    |
| Naming              | `onX` for event props, `handleX` for handler functions.                |
| Debugging           | `debugger` + breakpoints + React DevTools.                             |

> _"The only way to go fast, is to go well."_ — Robert C. Martin
> _"It is forbidden in React to mutate state directly."_
