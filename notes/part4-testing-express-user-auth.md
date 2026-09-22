# Part 4 — Testing Express Servers & User Administration

> Reference for structuring Express backends, integration testing with Node's test runner + Supertest, user administration with bcrypt, and JWT-based token authentication.
> Assumes prior knowledge. Focus: canonical patterns, correct testing workflow, and secure auth.

## 📑 Index

1. [Backend Project Structure](#1-backend-project-structure)
2. [Splitting app.js and index.js](#2-splitting-appjs-and-indexjs)
3. [Router Modules](#3-router-modules)
4. [Logger and Config Modules](#4-logger-and-config-modules)
5. [Middleware Module](#5-middleware-module)
6. [Node's Built-in Test Runner](#6-nodes-built-in-test-runner)
7. [Assertions: strictEqual vs deepStrictEqual](#7-assertions-strictequal-vs-deepstrictequal)
8. [Integration Testing with Supertest](#8-integration-testing-with-supertest)
9. [Environment-Based Configuration](#9-environment-based-configuration)
10. [Test Lifecycle: beforeEach / after / Promise.all](#10-test-lifecycle-beforeeach--after--promiseall)
11. [Test Helpers](#11-test-helpers)
12. [Async/Await in Express 5](#12-asyncawait-in-express-5)
13. [User Model & References](#13-user-model--references)
14. [Password Hashing with bcrypt](#14-password-hashing-with-bcrypt)
15. [Unique Constraints](#15-unique-constraints)
16. [Populate (Mongoose Join)](#16-populate-mongoose-join)
17. [Token Authentication with JWT](#17-token-authentication-with-jwt)
18. [Protecting Routes with Token Extraction](#18-protecting-routes-with-token-extraction)
19. [Token Expiration & Server-Side Sessions](#19-token-expiration--server-side-sessions)
20. [Testing Auth-Protected Routes](#20-testing-auth-protected-routes)
21. [Key Takeaways](#21-key-takeaways)

---

## 1. Backend Project Structure

```text
project/
├── controllers/ # route handlers (one per resource)
│ ├── notes.js
│ ├── users.js
│ └── login.js
├── models/ # Mongoose schemas (one per collection)
│ ├── note.js
│ └── user.js
├── utils/
│ ├── config.js # reads process.env
│ ├── logger.js # console wrapper
│ └── middleware.js # requestLogger, unknownEndpoint, errorHandler
├── tests/
│ ├── test_helper.js
│ ├── note_api.test.js
│ └── user_api.test.js
├── app.js # Express app (exported for tests)
├── index.js # entry — app.listen only
├── .env # NOT committed
├── .gitignore
├── eslint.config.mjs
├── package.json
└── dist/ # built frontend
```

**Why this matters:** small apps tolerate flat structure; larger apps don't. Separating responsibilities per file means a bug in one feature touches the fewest files possible. Rails enforces this pattern; Express doesn't — but it's still best practice.

---

## 2. Splitting app.js and index.js

**Purpose:** `app.js` exports the Express app; `index.js` starts the server. This separation lets Supertest import the app **without binding a port**.

```js
// app.js
const express = require("express");
const mongoose = require("mongoose");
const config = require("./utils/config");
const logger = require("./utils/logger");
const middleware = require("./utils/middleware");
const notesRouter = require("./controllers/notes");

const app = express();

logger.info("connecting to", config.MONGODB_URI);
mongoose
  .connect(config.MONGODB_URI, { family: 4 })
  .then(() => logger.info("connected to MongoDB"))
  .catch((error) =>
    logger.error("error connection to MongoDB:", error.message),
  );

app.use(express.static("dist"));
app.use(express.json());
app.use(middleware.requestLogger);

app.use("/api/notes", notesRouter);

app.use(middleware.unknownEndpoint);
app.use(middleware.errorHandler);

module.exports = app;
```

```js
// index.js
const app = require("./app");
const config = require("./utils/config");
const logger = require("./utils/logger");

app.listen(config.PORT, () => {
  logger.info(`Server running on port ${config.PORT}`);
});
```

**Why DB connection lives in `app.js`:** models don't own the connection — the app does. A model file should only define its schema.

---

## 3. Router Modules

Each router is an Express `Router`, exported as a module and mounted at a base path.

```js
// controllers/notes.js
const notesRouter = require("express").Router();
const Note = require("../models/note");

notesRouter.get("/", async (req, res) => {
  const notes = await Note.find({});
  res.json(notes);
});

notesRouter.post("/", async (req, res) => {
  const note = new Note(req.body);
  const savedNote = await note.save();
  res.status(201).json(savedNote);
});

// ... GET /:id, PUT /:id, DELETE /:id

module.exports = notesRouter;
```

```js
// app.js
const notesRouter = require("./controllers/notes");
app.use("/api/notes", notesRouter);
```

- Inside the router, paths are relative (`'/'`, `'/:id'`) — the mounting path is defined once in `app.js`.
- A single router file should only contain routes that share the same base path.

**Best practice — export shape:**

```js
// Multiple exports
module.exports = { info, error };

// Single export
module.exports = notesRouter;

// Consumption
const logger = require("./utils/logger"); // namespace
logger.info("...");

const { info, error } = require("./utils/logger"); // destructured
info("...");
```

Prefer the namespace form when using many functions; destructure when using 1–2.

**VS Code tip:** right-click a function → _Find All References_ before renaming to avoid breaking imports.

---

## 4. Logger and Config Modules

```js
// utils/logger.js
const info = (...params) => {
  if (process.env.NODE_ENV !== "test") console.log(...params);
};
const error = (...params) => {
  if (process.env.NODE_ENV !== "test") console.error(...params);
};
module.exports = { info, error };
```

```js
// utils/config.js
require("dotenv").config();

const PORT = process.env.PORT;
const MONGODB_URI =
  process.env.NODE_ENV === "test"
    ? process.env.TEST_MONGODB_URI
    : process.env.MONGODB_URI;

module.exports = { MONGODB_URI, PORT };
```

**Silencing logs in test mode** is important: Morgan and default console output clutter the test runner's output.

---

## 5. Middleware Module

```js
// utils/middleware.js
const logger = require("./logger");

const requestLogger = (req, res, next) => {
  logger.info("Method:", req.method);
  logger.info("Path: ", req.path);
  logger.info("Body: ", req.body);
  logger.info("---");
  next();
};

const unknownEndpoint = (req, res) => {
  res.status(404).send({ error: "unknown endpoint" });
};

const errorHandler = (error, req, res, next) => {
  logger.error(error.message);

  if (error.name === "CastError") {
    return res.status(400).send({ error: "malformatted id" });
  }
  if (error.name === "ValidationError") {
    return res.status(400).json({ error: error.message });
  }
  if (
    error.name === "MongoServerError" &&
    error.message.includes("E11000 duplicate key error")
  ) {
    return res.status(400).json({ error: "expected `username` to be unique" });
  }
  if (error.name === "JsonWebTokenError") {
    return res.status(401).json({ error: "token invalid" });
  }
  if (error.name === "TokenExpiredError") {
    return res.status(401).json({ error: "token expired" });
  }

  next(error);
};

module.exports = { requestLogger, unknownEndpoint, errorHandler };
```

**Order in `app.js` (never change it):**

1. `express.static`
2. `express.json()`
3. `requestLogger`
4. Routes
5. `unknownEndpoint`
6. `errorHandler` (always last)

---

## 6. Node's Built-in Test Runner

No external test framework needed — Node 22 ships with `node:test` and `node:assert`.

```json
// package.json
{
  "scripts": {
    "test": "node --test"
  }
}
```

Node automatically runs any file under `tests/` ending in `.test.js`.

**Test file anatomy:**

```js
const { test, describe } = require("node:test");
const assert = require("node:assert");

const reverse = require("../utils/for_testing").reverse;

test("reverse of react", () => {
  assert.strictEqual(reverse("react"), "tcaer");
});

describe("average", () => {
  test("of one value is the value itself", () => {
    assert.strictEqual(average([1]), 1);
  });
  test("of many is calculated correctly", () => {
    assert.strictEqual(average([1, 2, 3, 4]), 2.5);
  });
});
```

**Naming:** describe behavior, not implementation.
`should [result] when [condition]` — e.g. `"creation fails with 400 if username already taken"`.

**Alternatives:** Mocha → Jest → Vitest. Vitest is the modern successor; the course uses `node:test` to avoid extra dependencies.

---

## 7. Assertions: strictEqual vs deepStrictEqual

| Function                       | Compares                           |
| ------------------------------ | ---------------------------------- |
| `assert.strictEqual(a, b)`     | Same reference / strict `===`      |
| `assert.deepStrictEqual(a, b)` | Same shape and content (recursive) |

Use `deepStrictEqual` for **objects and arrays**; use `strictEqual` for primitives.

```js
assert.deepStrictEqual(response.body, expectedObject);
assert.strictEqual(response.body.length, 2);
```

---

## 8. Integration Testing with Supertest

```bash
npm install --save-dev supertest
```

Supertest boots the app on an ephemeral port and provides a fluent HTTP client — no need to call `app.listen` yourself.

```js
// tests/note_api.test.js
const { test, after, beforeEach } = require("node:test");
const assert = require("node:assert");
const mongoose = require("mongoose");
const supertest = require("supertest");
const app = require("../app");

const api = supertest(app);

test("notes are returned as json", async () => {
  await api
    .get("/api/notes")
    .expect(200)
    .expect("Content-Type", /application\/json/);
});

after(async () => {
  await mongoose.connection.close();
});
```

- Use a **regex** for `Content-Type` — a string requires an exact match.
- `await` is only allowed inside `async` functions.
- Always close the Mongo connection in `after`.

**Supertest vs Axios:** Supertest boots the app itself and provides `.expect()` — preferred for HTTP integration tests.

---

## 9. Environment-Based Configuration

Different configurations per run mode:

```json
// package.json
{
  "scripts": {
    "start": "cross-env NODE_ENV=production node index.js",
    "dev": "cross-env NODE_ENV=development node --watch index.js",
    "test": "cross-env NODE_ENV=test node --test",
    "lint": "eslint ."
  }
}
```

**On Windows**, plain `NODE_ENV=...` doesn't work in `cmd`. Use `cross-env`:

```bash
npm install cross-env
```

```env

# .env

TEST_MONGODB_URI=mongodb+srv://user:pass@cluster.mongodb.net/testNoteApp
MONGODB_URI=mongodb+srv://user:pass@cluster.mongodb.net/noteApp
```

Use a **separate test database** so integration tests don't corrupt development data.

---

## 10. Test Lifecycle: beforeEach / after / Promise.all

### Resetting state between tests

```js
const { test, after, beforeEach } = require("node:test");

beforeEach(async () => {
  await Note.deleteMany({});
  await Note.insertMany(helper.initialNotes);
});
```

**Why `beforeEach` and not `before`?** Tests must not affect each other. Resetting before each guarantees a known starting state.

### The forEach / Promise.all pitfall

```js
// ❌ BAD — forEach does not return a promise to await
beforeEach(async () => {
  await Note.deleteMany({});
  helper.initialNotes.forEach(async (note) => {
    await new Note(note).save(); // runs in parallel, may not finish before tests
  });
});
```

```js
// ✅ GOOD — Promise.all waits for all saves
beforeEach(async () => {
  await Note.deleteMany({});
  const noteObjects = helper.initialNotes.map((n) => new Note(n));
  await Promise.all(noteObjects.map((n) => n.save()));
});
```

**Caveat:** `Promise.all` runs operations in **parallel** — order is not guaranteed.

**If order matters:**

```js
for (const note of helper.initialNotes) {
  await new Note(note).save();
}
```

**Best option — Mongoose's `insertMany`:**

```js
beforeEach(async () => {
  await Note.deleteMany({});
  await Note.insertMany(helper.initialNotes);
});
```

---

## 11. Test Helpers

Shared utilities live in `tests/test_helper.js`.

```js
// tests/test_helper.js
const Note = require('../models/note')
const User = require('../models/user')

const initialNotes = [
{ content: 'HTML is easy', important: false },
{ content: 'Browser can execute only JavaScript', important: true },
]

const nonExistingId = async () => {
const note = new Note({ content: 'willremovethissoon' })
await note.save()
await note.deleteOne()
return note.\_id.toString()
}

const notesInDb = async () => {
const notes = await Note.find({})
return notes.map((note) => note.toJSON())
}

const usersInDb = async () => {
const users = await User.find({})
return users.map((u) => u.toJSON())
}

module.exports = { initialNotes, nonExistingId, notesInDb, usersInDb }
```

**`nonExistingId`** — a valid-format ObjectId that's guaranteed not to exist. Used to test 404/400 paths.

---

## 12. Async/Await in Express 5

Express 5 automatically forwards rejected promises to the error handler. **`.catch(next)` is no longer required.**

```js
// ✅ Express 5 — clean
notesRouter.get("/", async (req, res) => {
  const notes = await Note.find({});
  res.json(notes);
});

notesRouter.post("/", async (req, res) => {
  const savedNote = await new Note(req.body).save();
  res.status(201).json(savedNote);
});
```

**Rule:** never mix `async/await` and `.then()` in the same function — pick one.

---

## 13. User Model & References

Document databases have no native joins — references are manual and defined via Mongoose.

**Two-way reference pattern:**

```js
// models/user.js
const userSchema = new mongoose.Schema({
username: String,
name: String,
passwordHash: String,
notes: [
{
type: mongoose.Schema.Types.ObjectId,
ref: 'Note',
},
],
})

userSchema.set('toJSON', {
transform: (document, returnedObject) => {
returnedObject.id = returnedObject.\_id.toString()
delete returnedObject.\_id
delete returnedObject.\_\_v
delete returnedObject.passwordHash // never expose the hash
},
})

module.exports = mongoose.model('User', userSchema)
```

```js
// models/note.js
const noteSchema = new mongoose.Schema({
  content: { type: String, required: true, minlength: 5 },
  important: Boolean,
  user: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "User",
  },
});
```

**Notes:**

- `mongoose.Schema.Types.ObjectId` + `ref: 'Model'` — Mongoose feature, not MongoDB.
- Store **references on both sides**: user has an array of note ids; each note has one user id.
- Alternative: embed notes directly inside the user document (Mongo-specific). Loses per-note ids — chosen against here.

**Creating a note with a user reference:**

```js
notesRouter.post('/', async (req, res) => {
const { content, important, userId } = req.body
const user = await User.findById(userId)

if (!user) {
return res.status(400).json({ error: 'userId missing or not valid' })
}

const note = new Note({ content, important, user: user.\_id })
const savedNote = await note.save()

user.notes = user.notes.concat(savedNote.\_id)
await user.save()

res.status(201).json(savedNote)
})
```

---

## 14. Password Hashing with bcrypt

**Never store plain-text passwords.**

```bash
npm install bcrypt

# If bcrypt fails to build on Windows: npm install bcryptjs

```

```js
// controllers/users.js
const bcrypt = require("bcrypt");
const usersRouter = require("express").Router();
const User = require("../models/user");

usersRouter.post("/", async (req, res) => {
  const { username, name, password } = req.body;

  const saltRounds = 10;
  const passwordHash = await bcrypt.hash(password, saltRounds);

  const user = new User({ username, name, passwordHash });
  const savedUser = await user.save();

  res.status(201).json(savedUser);
});

module.exports = usersRouter;
```

**Comparing a password:**

```js
const passwordCorrect =
  user === null ? false : await bcrypt.compare(password, user.passwordHash);
```

**Notes:**

- `saltRounds = 10` is the standard trade-off. Higher = slower = safer, but fewer hashes per second.
- Validate password length in the **controller**, not in the Mongoose schema — the schema sees the hash, not the plaintext.

---

## 15. Unique Constraints

Mongoose's `unique: true` creates a **MongoDB unique index** — it's not a validator.

```js
const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
  },
});
```

**Critical caveat:** if duplicate values already exist in the collection, the index **won't be created** — the constraint silently fails. Ensure data is clean before enabling.

**Errors thrown:** `MongoServerError` with code `E11000` — not a `ValidationError`.

```js
// errorHandler
if (
  error.name === "MongoServerError" &&
  error.message.includes("E11000 duplicate key error")
) {
  return res.status(400).json({ error: "expected `username` to be unique" });
}
```

**Test for it:**

```js
test("creation fails with 400 if username already taken", async () => {
  const usersAtStart = await helper.usersInDb();

  const newUser = { username: "root", name: "Superuser", password: "salainen" };

  const result = await api
    .post("/api/users")
    .send(newUser)
    .expect(400)
    .expect("Content-Type", /application\/json/);

  assert(result.body.error.includes("expected `username` to be unique"));

  const usersAtEnd = await helper.usersInDb();
  assert.strictEqual(usersAtEnd.length, usersAtStart.length);
});
```

---

## 16. Populate (Mongoose Join)

`populate` replaces stored ObjectIds with the full referenced documents.

```js
// Users with their notes
usersRouter.get("/", async (req, res) => {
  const users = await User.find({}).populate("notes", {
    content: 1,
    important: 1,
  });
  res.json(users);
});

// Notes with their author
notesRouter.get("/", async (req, res) => {
  const notes = await Note.find({}).populate("user", { username: 1, name: 1 });
  res.json(notes);
});
```

- Second argument limits which fields are returned.
- **Mechanism:** Mongoose issues **separate queries** per collection and stitches the results — it's not a real DB join.
- **Caveat:** data can change between the two queries; the response may reflect a stale snapshot.

---

## 17. Token Authentication with JWT

```bash
npm install jsonwebtoken
```

```js
// controllers/login.js
const jwt = require('jsonwebtoken')
const bcrypt = require('bcrypt')
const loginRouter = require('express').Router()
const User = require('../models/user')

loginRouter.post('/', async (req, res) => {
const { username, password } = req.body

const user = await User.findOne({ username })
const passwordCorrect = user === null
? false
: await bcrypt.compare(password, user.passwordHash)

if (!(user && passwordCorrect)) {
return res.status(401).json({ error: 'invalid username or password' })
}

const userForToken = {
username: user.username,
id: user.\_id,
}

const token = jwt.sign(userForToken, process.env.SECRET)

res.status(200).send({ token, username: user.username, name: user.name })
})

module.exports = loginRouter
```

```js
// app.js
const loginRouter = require("./controllers/login");
app.use("/api/login", loginRouter);
```

**Flow:**

1. Client POSTs credentials to `/api/login`.
2. Server verifies with `bcrypt.compare`.
3. Server signs a payload (`{ username, id }`) with `SECRET`.
4. Client stores the token and sends it with every subsequent request in the `Authorization` header.

**Status code:** `401 Unauthorized` for invalid credentials.

---

## 18. Protecting Routes with Token Extraction

**Bearer scheme:** the header is `Authorization: Bearer <token>`.

```js
// Extract the token from the Authorization header
const getTokenFrom = (request) => {
const authorization = request.get('authorization')
if (authorization && authorization.startsWith('Bearer ')) {
return authorization.replace('Bearer ', '')
}
return null
}

notesRouter.post('/', async (req, res) => {
const decodedToken = jwt.verify(getTokenFrom(req), process.env.SECRET)

if (!decodedToken.id) {
return res.status(401).json({ error: 'token invalid' })
}

const user = await User.findById(decodedToken.id)
if (!user) {
return res.status(400).json({ error: 'userId missing or not valid' })
}

const note = new Note({ ...req.body, user: user.\_id })
const savedNote = await note.save()
user.notes = user.notes.concat(savedNote.\_id)
await user.save()

res.status(201).json(savedNote)
})
```

**Why separate `getTokenFrom`?** Reusable across route handlers; keeps the handler focused.

**Extracting to middleware** (recommended when many routes need it):

```js
// utils/middleware.js
const tokenExtractor = (request, response, next) => {
  const authorization = request.get("authorization");
  if (authorization && authorization.startsWith("Bearer ")) {
    request.token = authorization.replace("Bearer ", "");
  }
  next();
};

const userExtractor = async (request, response, next) => {
  const decodedToken = jwt.verify(request.token, process.env.SECRET);
  request.user = await User.findById(decodedToken.id);
  next();
};
```

```js
// Apply to a whole router
app.use(
  "/api/blogs",
  middleware.tokenExtractor,
  middleware.userExtractor,
  blogsRouter,
);

// Or apply to specific routes
blogsRouter.post("/", userExtractor, async (req, res) => {
  /_ ... _/;
});
blogsRouter.delete("/:id", userExtractor, async (req, res) => {
  /_ ... _/;
});
```

---

## 19. Token Expiration & Server-Side Sessions

**Fundamental limitation:** once issued, a JWT is trusted blindly by the server. If leaked, it's valid until expiry.

### Option 1 — Expiry (`expiresIn`)

```js
const token = jwt.sign(
userForToken,
process.env.SECRET,
{ expiresIn: 60 \* 60 } // 1 hour
)
```

Expired tokens trigger `TokenExpiredError`:

```js
if (error.name === "TokenExpiredError") {
  return res.status(401).json({ error: "token expired" });
}
```

Trade-off: shorter = safer, longer = better UX.

### Option 2 — Server-side sessions (stateful)

Store tokens in a DB / Redis; verify on every request. Gives you revocation but:

- Slower (extra DB lookup per request).
- More complex infrastructure.
- Common pattern: opaque random strings + Key-Value store (Redis), typically transmitted via cookies.

**Note:** HTTPS is mandatory in production. Fly.io upgrades HTTP→HTTPS automatically.

---

## 20. Testing Auth-Protected Routes

Add the token to the request header explicitly:

```js
test('a valid note can be added', async () => {
const user = await User.findOne({ username: 'root' })
const token = jwt.sign(
{ username: user.username, id: user.\_id },
process.env.SECRET,
{ expiresIn: 60 \* 60 }
)

const newNote = { content: 'async/await simplifies making async calls', important: true }

await api
.post('/api/notes')
.set('Authorization', `Bearer ${token}`)
.send(newNote)
.expect(201)
.expect('Content-Type', /application\/json/)
})

test('adding a note without a token fails with 401', async () => {
const newNote = { content: 'no token here', important: true }

await api.post('/api/notes').send(newNote).expect(401)
})
```

---

## 21. Key Takeaways

| Principle        | Rule                                                                                              |
| ---------------- | ------------------------------------------------------------------------------------------------- |
| Structure        | Split `app.js` (Express app) from `index.js` (listen).                                            |
| Routers          | One router file per resource; mounted in `app.js` with a base path.                               |
| Test runner      | `node:test` + `node:assert` — no extra dependency.                                                |
| Assertions       | `deepStrictEqual` for objects/arrays; `strictEqual` for primitives.                               |
| Supertest        | Boots the app automatically — no port needed.                                                     |
| Test DB          | Always use a separate test database via `NODE_ENV=test`.                                          |
| Isolation        | `beforeEach` resets DB; use `insertMany` not `forEach`.                                           |
| Async/await      | Never mix with `.then()`. Express 5 auto-forwards rejections.                                     |
| Passwords        | Never store plaintext; use bcrypt with `saltRounds = 10`.                                         |
| Uniqueness       | `unique: true` creates a DB index, not a validator.                                               |
| Refs             | Two-way: user has array of note ids; note has user id.                                            |
| Populate         | Fake join — Mongoose does separate queries and merges.                                            |
| JWT              | Signed payload; client sends in `Authorization: Bearer <token>`.                                  |
| Errors           | `CastError` → 400, `ValidationError` → 400, `JsonWebTokenError` → 401, `TokenExpiredError` → 401. |
| Middleware order | `errorHandler` **always last**.                                                                   |
| HTTPS            | Mandatory in production for auth traffic.                                                         |

> _"Testing is the only way to know your backend works — every time, on every machine."_
