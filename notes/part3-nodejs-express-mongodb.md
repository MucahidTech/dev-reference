# Part 3 — Programming a Server with Node.js and Express

> Reference for building a REST API with Node.js, Express, MongoDB (Mongoose), deployment to the cloud, validation, and ESLint.
> Assumes prior knowledge. Focus: canonical patterns, correct HTTP semantics, and production-ready structure.

## 📑 Index

1. [Project Setup](#1-project-setup)
2. [Raw Node HTTP Server (Baseline)](#2-raw-node-http-server-baseline)
3. [Express Server](#3-express-server)
4. [REST API Conventions](#4-rest-api-conventions)
5. [Middleware](#5-middleware)
6. [HTTP Method Semantics](#6-http-method-semantics)
7. [Logging with Morgan](#7-logging-with-morgan)
8. [Testing Endpoints (.rest / Postman / curl)](#8-testing-endpoints-rest--postman--curl)
9. [Deployment](#9-deployment)
10. [CORS vs. Vite Proxy](#10-cors-vs-vite-proxy)
11. [MongoDB + Mongoose](#11-mongodb--mongoose)
12. [CRUD with Mongoose](#12-crud-with-mongoose)
13. [Error Handling Middleware](#13-error-handling-middleware)
14. [Environment Variables (dotenv)](#14-environment-variables-dotenv)
15. [Validation with Mongoose](#15-validation-with-mongoose)
16. [ESLint](#16-eslint)
17. [Project Structure](#17-project-structure)
18. [Key Takeaways](#18-key-takeaways)

---

## 1. Project Setup

```bash
npm init # creates package.json (main: index.js)
npm install express mongoose dotenv cors morgan
npm install --save-dev eslint @eslint/js globals @stylistic/eslint-plugin
```

```json
// package.json scripts
{
  "scripts": {
    "start": "node index.js",
    "dev": "node --watch index.js",
    "lint": "eslint ."
  }
}
```

- `.gitignore` **must** contain: `node_modules/`, `.env`, `dist/`.
- `npm start` and `npm test` don't require `run`; custom scripts do (`npm run dev`).
- `node --watch index.js` restarts the server on file changes (**replaces nodemon** in Node 22+).

**Backend uses CommonJS** (`require` / `module.exports`) by default. ES modules are supported but with limitations in the Node ecosystem — stick to CommonJS.

---

## 2. Raw Node HTTP Server (Baseline)

Provided for context only — **never use this in production**.

```js
const http = require("http");

const app = http.createServer((request, response) => {
  response.writeHead(200, { "Content-Type": "application/json" });
  response.end(JSON.stringify(notes));
});

const PORT = 3001;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

- `writeHead(status, headers)` — set status + headers.
- `end(body)` — must be a string or Buffer. Hence `JSON.stringify`.
- `listen(port)` — start listening.

---

## 3. Express Server

```js
const express = require("express");
const app = express();

app.get("/", (req, res) => {
  res.send("<h1>Hello World!</h1>");
});

app.get("/api/notes", (req, res) => {
  res.json(notes); // sets Content-Type: application/json automatically
});

const PORT = process.env.PORT || 3001;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

- `res.send(body)` → `text/html` by default; status defaults to `200`.
- `res.json(obj)` → sets `Content-Type: application/json` and serializes automatically. **Never call `JSON.stringify` yourself.**
- `req.params` — path parameters (from `:id`).
- `req.query` — query string parameters.
- `req.body` — parsed body (requires `express.json()` middleware).

---

## 4. REST API Conventions

Every resource has a **unique URL**. Resources are identified by their id within a collection.

| URL          | Verb   | Meaning                       |
| ------------ | ------ | ----------------------------- |
| `/notes`     | GET    | Fetch all resources           |
| `/notes/:id` | GET    | Fetch one resource            |
| `/notes`     | POST   | Create a new resource         |
| `/notes/:id` | DELETE | Remove a resource             |
| `/notes/:id` | PUT    | Replace the entire resource   |
| `/notes/:id` | PATCH  | Partially update the resource |

**Fetching a single resource with existence check:**

```js
app.get("/api/notes/:id", (req, res) => {
  const id = req.params.id;
  const note = notes.find((n) => n.id === id);
  if (note) {
    res.json(note);
  } else {
    res.status(404).end(); // 404 Not Found
  }
});
```

**Creating a resource:**

```js
app.use(express.json()); // required to populate req.body

const generateId = () => {
  const maxId =
    notes.length > 0 ? Math.max(...notes.map((n) => Number(n.id))) : 0;
  return String(maxId + 1);
};

app.post("/api/notes", (req, res) => {
  const body = req.body;
  if (!body.content) {
    return res.status(400).json({ error: "content missing" }); // 400 Bad Request
  }
  const note = {
    content: body.content,
    important: body.important ?? false,
    id: generateId(),
  };
  notes = notes.concat(note);
  res.json(note);
});
```

- `return` before `res.status(...)` prevents the rest of the handler from running.
- Never trust the client — always validate the body server-side.

**Deleting:**

```js
app.delete("/api/notes/:id", (req, res) => {
  const id = req.params.id;
  notes = notes.filter((n) => n.id !== id);
  res.status(204).end(); // 204 No Content
});
```

---

## 5. Middleware

Middleware = functions that process `req`/`res`. Applied with `app.use()`. **Order matters — they run in the order registered.**

```js
const requestLogger = (req, res, next) => {
  console.log("Method:", req.method);
  console.log("Path: ", req.path);
  console.log("Body: ", req.body);
  console.log("---");
  next(); // pass control to the next middleware/route
};

app.use(requestLogger);
```

**Unknown endpoint handler** — must be **after** all routes, before the error handler:

```js
const unknownEndpoint = (req, res) => {
  res.status(404).send({ error: "unknown endpoint" });
};
app.use(unknownEndpoint);
```

**Canonical middleware order:**

```js
app.use(express.static("dist")); // serve built frontend
app.use(express.json()); // parse JSON bodies — must come early
app.use(requestLogger); // custom logging

app.use("/api/notes", notesRouter); // routes

app.use(unknownEndpoint); // catch-all 404
app.use(errorHandler); // error handler — ALWAYS LAST
```

---

## 6. HTTP Method Semantics

| Property       | Rule                                                                                                          |
| -------------- | ------------------------------------------------------------------------------------------------------------- |
| **Safe**       | A method is _safe_ if it has no side effects. `GET` and `HEAD` are safe.                                      |
| **Idempotent** | Repeating a request yields the same result. `GET`, `HEAD`, `PUT`, `DELETE` are idempotent. `POST` is **not**. |
| **HEAD**       | Returns only status code and headers, no body.                                                                |

**Status codes to memorize:**

| Code | Name                  | When to use                |
| ---- | --------------------- | -------------------------- |
| 200  | OK                    | Successful GET/PUT         |
| 201  | Created               | Successful POST            |
| 204  | No Content            | Successful DELETE          |
| 400  | Bad Request           | Client sent invalid data   |
| 404  | Not Found             | Resource doesn't exist     |
| 500  | Internal Server Error | Unhandled server exception |

**Note:** When the endpoint exists but the **id is malformed** (e.g. wrong type), `400` is often more appropriate than `404`.

---

## 7. Logging with Morgan

```bash
npm install morgan
```

```js
const morgan = require("morgan");

// Basic
app.use(morgan("tiny")); // GET /api/notes 200 231 - 5.022 ms

// Custom token to log request body
morgan.token("body", (req) => JSON.stringify(req.body));

app.use(
  morgan(":method :url :status :res[content-length] - :response-time ms :body"),
);
```

- `morgan.token(name, fn)` — define a custom placeholder usable as `:name` in the format string.
- Log the request body only when necessary — it can contain sensitive data.

---

## 8. Testing Endpoints (.rest / Postman / curl)

Create a `requests/` directory at the project root. Files end with `.rest`.

```rest

### get all notes

GET http://localhost:3001/api/notes

### create a note

POST http://localhost:3001/api/notes
Content-Type: application/json

{
"content": "A new note",
"important": true
}
```

- Separate requests with `###`.
- The **REST Client** VS Code extension lets you send requests inline and view responses side-by-side.
- When the request header looks empty during debugging, add `console.log(req.headers)` to verify what the client actually sent.

---

## 9. Deployment

### Port binding for cloud platforms

```js
const PORT = process.env.PORT || 3001;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

Cloud providers set `PORT`; locally it falls back to `3001`.

### Serving the frontend from Express

```bash

# 1. Build the frontend

cd ../frontend && npm run build # produces dist/

# 2. Copy dist/ into the backend (or configure your build script)

```

```js
app.use(express.static("dist")); // serve static files first
```

Requests hit `dist/` first; `/api/*` routes fall through to the API.

### One-command build + copy (optional)

```json
// package.json scripts
{
  "build:ui": "rm -rf dist && cd ../frontend && npm run build && cp -r dist ../backend",
  "deploy:full": "npm run build:ui && git add . && git commit -m uibuild && git push"
}
```

**Note:** On Windows, unix shell commands require Git Bash:

```bash
npm config set script-shell "C:\\Program Files\\git\\bin\\bash.exe"
```

**Hosting platforms (2025):**

- **Render** — free tier, most common in the course.
- **Fly.io** — free tier, requires CLI setup.
- Heroku — paid since Nov 2022.

---

## 10. CORS vs. Vite Proxy

### Option A — CORS (when frontend and backend are on different origins)

```bash
npm install cors
```

```js
const cors = require("cors");
app.use(cors()); // allow all origins
// app.use(cors({ origin: 'https://example.com' })) // restrict in production
```

### Option B — Vite Proxy (recommended for local dev)

```js
// vite.config.js
export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      "/api": {
        target: "http://localhost:3001",
        changeOrigin: true,
      },
    },
  },
});
```

- Requests to `http://localhost:5173/api/*` are transparently forwarded to `http://localhost:3001/api/*`.
- **No CORS needed** because the browser sees only one origin.
- When using the proxy, remove `cors`: `npm remove cors`.

**Same-origin policy:** browsers block cross-origin requests unless the response includes an appropriate `Access-Control-Allow-Origin` header.

---

## 11. MongoDB + Mongoose

MongoDB is a **document database (NoSQL)**. Mongoose is an ODM that enforces a schema.

```bash
npm install mongoose
```

### Connection

```js
const mongoose = require("mongoose");
mongoose.set("strictQuery", false);
mongoose
  .connect(process.env.MONGODB_URI, { family: 4 })
  .then(() => console.log("connected to MongoDB"))
  .catch((err) => console.log("error connecting to MongoDB:", err.message));
```

### Schema + Model

```js
const noteSchema = new mongoose.Schema({
content: String,
important: Boolean,
})

// Clean up Mongo-specific fields in every JSON response
noteSchema.set('toJSON', {
transform: (document, returnedObject) => {
returnedObject.id = returnedObject.\_id.toString()
delete returnedObject.\_id
delete returnedObject.\_\_v
},
})

module.exports = mongoose.model('Note', noteSchema)
```

- Model name `Note` → Mongoose creates collection `notes` (lowercased, pluralized).
- `Note` → `notes`, `Person` → `people`, `Blog` → `blogs`.
- `toJSON` transform runs **automatically** on every serialization — no need to repeat per route.
- `_id` is an ObjectId, not a string — always convert with `.toString()`.

---

## 12. CRUD with Mongoose

### Fetch all

```js
app.get("/api/notes", (req, res) => {
  Note.find({}).then((notes) => res.json(notes));
});
```

`find({})` → all documents. `find({ important: true })` → filtered.

### Fetch by id

```js
app.get("/api/notes/:id", (req, res, next) => {
  Note.findById(req.params.id)
    .then((note) => {
      if (note) res.json(note);
      else res.status(404).end();
    })
    .catch((error) => next(error));
});
```

### Create

```js
app.post("/api/notes", (req, res, next) => {
  const body = req.body;
  if (!body.content) {
    return res.status(400).json({ error: "content missing" });
  }

  const note = new Note({
    content: body.content,
    important: body.important || false,
  });

  note
    .save()
    .then((savedNote) => res.json(savedNote))
    .catch((error) => next(error));
});
```

### Delete

```js
app.delete("/api/notes/:id", (req, res, next) => {
  Note.findByIdAndDelete(req.params.id)
    .then(() => res.status(204).end())
    .catch((error) => next(error));
});
```

### Update

```js
app.put("/api/notes/:id", (req, res, next) => {
  const { content, important } = req.body;

  Note.findById(req.params.id)
    .then((note) => {
      if (!note) return res.status(404).end();
      note.content = content;
      note.important = important;
      return note.save().then((updatedNote) => res.json(updatedNote));
    })
    .catch((error) => next(error));
});
```

- `note.save()` persists the changes.
- `findByIdAndUpdate` exists, but updating via `save()` is recommended — it runs validators.

---

## 13. Error Handling Middleware

Centralize error handling in a single middleware — **always the last one registered**.

```js
const errorHandler = (error, req, res, next) => {
  console.error(error.message);

  if (error.name === "CastError") {
    return res.status(400).send({ error: "malformatted id" });
  }
  if (error.name === "ValidationError") {
    return res.status(400).json({ error: error.message });
  }

  next(error); // delegate to default Express handler (500)
};

// ... after all routes and unknownEndpoint
app.use(errorHandler);
```

**In route handlers**, pass errors to the handler with `next`:

```js
.catch(error => next(error))
```

**Why last?** Express executes middleware in order. Anything registered after the error handler won't have its errors caught.

---

## 14. Environment Variables (dotenv)

```bash
npm install dotenv
```

```env
// .env — MUST be in .gitignore
MONGODB_URI=mongodb+srv://user:pass@cluster.mongodb.net/dbname
PORT=3001
```

```js
require("dotenv").config(); // at the top of index.js
const PORT = process.env.PORT;
```

Call `.config()` **before** any file that reads from `process.env`.

**On cloud platforms:** set env vars through the provider's dashboard. For Fly.io, add `.dockerignore` with `.env` inside.

---

## 15. Validation with Mongoose

Never trust client input — validate server-side.

### Built-in validators

```js
const noteSchema = new mongoose.Schema({
  content: {
    type: String,
    minLength: 5,
    required: true,
  },
  important: Boolean,
});
```

### Custom validators

```js
number: {
type: String,
validate: {
validator: function (v) {
return /^\d{2,3}-\d{5,}$/.test(v)
    },
    message: (props) =>
      `${props.value} is not a valid phone number! Use format: XX-XXXXXXX or XXX-XXXXXXXX`,
  },
}
```

Failed validations throw a `ValidationError`— handled in`errorHandler`.

---

## 16. ESLint

```bash
npm install --save-dev eslint @eslint/js globals @stylistic/eslint-plugin
npx eslint --init # choose: syntax, CommonJS, Node, JS, no TS
```

```js
// eslint.config.mjs
import globals from "globals";
import js from "@eslint/js";
import stylisticJs from "@stylistic/eslint-plugin";

export default [
  js.configs.recommended, // recommended rules — include first
  {
    files: ["**/*.js"],
    languageOptions: {
      sourceType: "commonjs",
      globals: { ...globals.node },
      ecmaVersion: "latest",
    },
    plugins: {
      "@stylistic/js": stylisticJs,
    },
    rules: {
      "@stylistic/js/indent": ["error", 2],
      "@stylistic/js/linebreak-style": ["error", "unix"],
      "@stylistic/js/quotes": ["error", "single"],
      "@stylistic/js/semi": ["error", "never"],
      eqeqeq: "error",
      "no-trailing-spaces": "error",
      "object-curly-spacing": ["error", "always"],
      "arrow-spacing": ["error", { before: true, after: true }],
      "no-console": "off",
    },
  },
  {
    ignores: ["dist/**"],
  },
];
```

```bash

# Run

npx eslint index.js # single file
npm run lint # all files (script: eslint .)
```

**Notes:**

- **Custom `plugins`** let you define and load your own rules.
- **`eqeqeq`** enforces `===` over `==`.
- **`no-console: 'off'`** disables the default warning against `console.log` (useful during development).
- **Airbnb config** is a common industry standard: `eslint-config-airbnb`.
- **VS Code ESLint extension** shows lint errors live while coding.
- **CRLF warning on Windows:** configure Git to use LF via `core.autocrlf` or convert the file — see the [Stack Overflow answer referenced in the course](https://stackoverflow.com/questions/48692741/how-can-i-make-all-line-endings-eols-in-all-files-in-visual-studio-code-unix).

---

## 17. Project Structure

```text
project/
├── index.js # entry point — app.listen
├── app.js # Express app (exported for tests)
├── package.json
├── .env # NOT committed
├── .gitignore # node_modules, .env, dist
├── eslint.config.mjs
├── requests/ # \*.rest files
├── models/
│ └── note.js # Mongoose schema + model
├── controllers/
│ └── notes.js # route handlers
├── utils/
│ ├── config.js # reads process.env
│ ├── logger.js # console.log wrapper
│ └── middleware.js # requestLogger, unknownEndpoint, errorHandler
└── dist/ # built frontend (from part 3 of Part 3)
```

**Best practices:**

- Move Mongoose connection + schema to `models/<resource>.js`.
- Move route handlers to `controllers/<resource>.js` in larger apps.
- Move `requestLogger`, `unknownEndpoint`, `errorHandler` into `utils/middleware.js`.
- Centralize `process.env` reads in `utils/config.js`.

---

## 18. Key Takeaways

| Principle           | Rule                                                                             |
| ------------------- | -------------------------------------------------------------------------------- |
| Trust nothing       | Validate all client input server-side.                                           |
| HTTP semantics      | Safe = GET/HEAD. Idempotent = GET/HEAD/PUT/DELETE (not POST).                    |
| Status codes        | 200/201/204 for success; 400/404/500 for failures.                               |
| Middleware order    | `express.json()` early; `unknownEndpoint` after routes; `errorHandler` **last**. |
| Error delegation    | `.catch(error => next(error))` in every async route.                             |
| Mongoose            | `toJSON` transform to clean `_id` and `__v`.                                     |
| Env vars            | `.env` always in `.gitignore`; read via `process.env`.                           |
| Static assets       | `express.static('dist')` before API routes.                                      |
| CORS vs Proxy       | Prefer Vite proxy in dev; use CORS only when cross-origin is real.               |
| `findByIdAndUpdate` | Prefer `findById` + `save()` — runs validators.                                  |
| ESLint              | Enforce `===`, LF endings, 2-space indent, no trailing whitespace.               |
| Logging             | Keep Morgan on; custom `:body` token for debugging POSTs.                        |

> _"The HTTP GET request should be safe."_
> _"When in doubt, `console.log(req.body)` — but never trust the client."_
