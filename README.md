# midu-style — Claude Code Skill

A personal Claude Code skill that makes Claude write **React.js, Next.js, and Node/Express/MongoDB**
code in **midu100 (Kazi Mridul)'s** exact style, file structure, and
conventions — extracted from real repositories: `Air_Bnb`, `E-commece-FullStack`,
`Kazir_Haat_Client` / `Kazir_Haat_server`, `ChatWebApplication`, `Ecommerce-Next.js-`.

---

## What it does

When you ask Claude to scaffold or write code, this skill makes the output match how *you* write:

- **Backend** — Express 5 + Mongoose 9, CommonJS, flat root (no `src/`), the
  `controllers / models / routes / middleware / services / utils / dbConfig` layout,
  `{ success, message, data }` responses sent with `.send()`.
- **React (Vite)** — React Router v7 (imported from `react-router`, not `react-router-dom`),
  RTK Query or centralized axios service objects, Tailwind v4 with `@theme` tokens.
- **Next.js** — App Router with `(auth)/(admin)/(e-commerce)` route groups, `"use client"`,
  `lib/apiClient.js` fetch wrapper, middleware file named `proxy.js` using `jose`.
- **Uploads** — multer + Cloudinary only, hand-built `FormData` in the submit handler.
- **Real-time** — socket.io rooms, `global.io` emits from controllers, WebRTC call signaling.

The skill activates automatically when a request involves React/Next/Express/Mongo work.
You can also invoke it explicitly with `/midu-style`.

---

## Language rule

The skill enforces **English-only** for everything written by the developer for other
developers — no Bangla script and no Banglish:

| Artifact | Language |
|---|---|
| Code comments (inline, block, `// ====== Section`, JSX, JSDoc) | English |
| Commit messages, branch names, PR titles/descriptions | English |
| README and all documentation | English |
| Variable / function / file names, `console.log` strings, thrown errors | English |
| **User-facing UI copy rendered to the end user** | Bengali allowed (`font-bangla`, `৳{price}`) |

That last row is the only exception — it is product content, not code.

---

## Repository layout

```
midu-style/
├── SKILL.md                        ← main skill file (required, has the YAML frontmatter)
├── README.md                       ← this file
└── references/
    ├── backend.md                  ← Express entry, dbConfig, Mongoose + bcrypt pre-hook,
    │                                 controllers, cookie authMiddleware, roleCheck, pagination
    ├── frontend.md                 ← React (Vite) components + React Router v7 config;
    │                                 Next.js App Router pages, apiClient.js, proxy.js
    ├── image-upload.md             ← multer + Cloudinary (both variants), FormData patterns
    ├── state-and-data.md           ← RTK Query, createSlice, cookie auth, ProtectedRoute
    └── realtime-socket.md          ← socket.io client/server, rooms, WebRTC signaling
```

`SKILL.md` stays lean; the `references/` files are loaded on demand when a task needs them.

---

## Installation

A Claude Code skill is just a folder containing a `SKILL.md`. Personal skills live in
`~/.claude/skills/`.

### Install from this repo

```powershell
git clone https://github.com/midu100/midu-style-skill.git "$env:USERPROFILE\.claude\skills\midu-style"
```

```bash
git clone https://github.com/midu100/midu-style-skill.git ~/.claude/skills/midu-style
```

Then **restart Claude Code** (or start a new session) so the skill is picked up.

### Verify it loaded

- Type `/` and look for `midu-style` in the skill list, **or**
- Ask *"Scaffold an Express + Mongo auth API"* and confirm the output uses your structure.

### Update later

```powershell
git -C "$env:USERPROFILE\.claude\skills\midu-style" pull
```

---

## Project-level install (optional)

To apply the skill inside one specific project instead of globally, copy the same folder to
that project's `.claude/skills/` directory and commit it:

```
your-project/
└── .claude/
    └── skills/
        └── midu-style/
            ├── SKILL.md
            └── references/
```

Project skills override personal skills of the same name.

---

## How skills get triggered

- **Automatically** — Claude matches the request against the `description` in the `SKILL.md`
  frontmatter and loads the skill when relevant.
- **Manually** — type `/midu-style`.

Keep the `description` specific and full of trigger words (React, Next.js, Express, Mongoose,
routing, schema, controller, component, upload, socket) — vague descriptions reduce
auto-triggering.

---

## Uninstall

```powershell
Remove-Item -Recurse -Force "$env:USERPROFILE\.claude\skills\midu-style"
```
