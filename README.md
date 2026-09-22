# 📚 Dev Reference

A personal, engineer-grade reference library. Curated notes, canonical patterns, and quick-decision guides for the tools and concepts I use in day-to-day full-stack development.

Not tutorials. Not a learning log. **A lookup table for experienced developers.**

---

## 🎯 Purpose

This repository answers three questions fast:

1. **"How do I use X correctly?"** — canonical code snippets for the most common tools and patterns.
2. **"Which tool should I choose for Y?"** — decision tables for recurring architecture choices.
3. **"Where do I find Z?"** — a single index across every topic I've documented.

If a note doesn't help me ship faster or debug quicker, it doesn't belong here.

---

## 📁 Structure

```
dev-reference/
├── README.md ← you are here
└── notes/
├── README.md ← full index, cross-cutting topics, decision guide
├── part0-fundamentals-of-web-apps.md
├── part1-react-foundations.md
├── part2-communicating-with-server.md
├── part3-nodejs-express-mongodb.md
├── part4-testing-express-user-auth.md
├── part5-testing-react-router-mui.md
├── part6-advanced-state-management.md
└── part7-custom-hooks-vite-internals.md
```

> **Start here:** [`notes/README.md`](notes/README.md) — the full index with cross-cutting tables, a quick-decision guide, and search tips.

---

## 🧭 How to Navigate

| You want to…                           | Open                                                        |
| -------------------------------------- | ----------------------------------------------------------- |
| Find a specific topic across all notes | [`notes/README.md`](notes/README.md) → Cross-Cutting Topics |
| Answer "which tool should I use?"      | [`notes/README.md`](notes/README.md) → Quick Decision Guide |
| Review a full part end-to-end          | The corresponding `part*.md` file                           |
| Add a new topic                        | See "Contributing" below                                    |

---

## 🛠️ Technologies Covered

**Frontend:** React, React Router, Zustand, TanStack Query, MaterialUI, styled-components, Vite

**Backend:** Node.js, Express, MongoDB, Mongoose, JWT, bcrypt

**Testing:** `node:test`, Supertest, Vitest, React Testing Library, Playwright

**Tooling:** ESLint, Prettier, esbuild, Rollup, GitHub Actions, npm workspaces

**Concepts:** REST, SPA architecture, state management, error boundaries, SSR, microservices

---

## 📐 Conventions

- **Language:** English (for international portability of the notes).
- **Section structure:** every file opens with an Index, ends with a Key Takeaways table.

---

## ✍️ Contributing (to future me)

When adding a new note:

1. Create the file under `notes/` with a descriptive name.
2. Add an **Index** at the top with anchor links to every section.
3. End with a **Key Takeaways** table.
4. Update `notes/README.md` → the main table and any relevant cross-cutting row.
5. Commit with a scoped message: `docs(notes): add <topic> cheat sheet`.

When a topic is referenced by **3+ files** or grows past ~50 lines in its home file, split it into its own note.

---

## 📜 License

Personal reference material. Free to use, adapt, or fork — no attribution required.

---

_"The best documentation is the one you actually consult."_
