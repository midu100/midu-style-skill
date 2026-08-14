---
name: midu-style
description: Build React.js, Next.js, and Node/Express/MongoDB code in midu100's (Kazi Mridul's) exact personal style and file structure. Use whenever creating or scaffolding a frontend (React+Vite or Next.js App Router) or a backend (Express + Mongoose), writing components, routing, data fetching, props, Mongoose schemas, Express routes/controllers/middleware, image/file uploads with multer + Cloudinary, JWT cookie auth, Redux Toolkit / RTK Query data layers, socket.io real-time features, or wiring a full-stack feature.
---

# midu-style — Build in the user's own coding style

Encodes the personal conventions of **midu100** (GitHub: Kazi Mridul), extracted from his
real repositories: `Air_Bnb`, `E-commece-FullStack`, `Kazir_Haat_Client`/`Kazir_Haat_server`,
`ChatWebApplication`, `Ecommerce-Next.js-`.

When writing React.js, Next.js, or Node/Express/MongoDB code, follow these so output looks
self-written.

## Universal rules
- **JavaScript only (no TS).** React components `.jsx`; utilities/config/store `.js`.
  `jsconfig.json`, never `tsconfig.json`.
- **`const Name = () => {}` arrow functions**, then `export default Name` at the bottom.
  Exceptions he actually makes: Mongoose `pre`/`methods` hooks use `async function () {}`
  (need `this`), and Next.js `page`/`layout`/`loading` entry files use
  `export default function Name()`.
- `import React from 'react'` is written at the top of nearly every component, even on React 19.
- Prefer `const`, rarely `let`, never `var`. camelCase vars/functions, PascalCase
  React components/files.
- **Quotes are NOT strictly single.** Backend leans single; frontend mixes freely. Don't
  fight the surrounding file — match it.
- `async/await` in `try/catch`. Backend catch is usually `catch (error) { console.log(error) }`.
  `.then()` only for the DB-connect line.
- Section comments: `// ====== Section name` (also `// ============= section =========`,
  and `{/* ====== Section ====== */}` in JSX). **Comment text is always English** — see
  the Language rule below.
- **Error-read idiom (frontend):** `err?.data?.message || err?.message || 'Fallback'`
  (RTK Query) or `err?.response?.data?.message || 'Fallback'` (axios).
- **Defensive reads everywhere:** `data?.products || []`, `discountPrice > 0 ? discountPrice : price`.
- Stack he is actually on: **React 19, Express 5, Mongoose 9, Tailwind v4, Next 16**,
  `bcrypt` (not bcryptjs), `node --watch` (not nodemon).

## Language rule — English only (hard rule)
Everything **written by the developer for other developers** must be in plain English.
Never Bangla (বাংলা script) and never Banglish (Bengali words typed in Latin letters —
`// ekhane user check kortesi`, `fix korlam`, `data ta ashe na`). No exceptions, even in
throwaway or scratch code.

Applies to:
- **Code comments** — inline, block, section banners (`// ====== Section name`), JSX
  comments, and JSDoc.
- **Commit messages** — subject and body. Write imperative English:
  `add product pagination`, `fix cart total on quantity change`, `refactor auth middleware`.
  Not `product er pagination add korlam`.
- **README.md and all docs** — headings, prose, setup steps, code-block captions.
- **Variable / function / file names**, branch names, PR titles and descriptions,
  `console.log` debug strings, and error messages thrown in code.

**The one exception — user-facing UI text.** Product copy rendered to the end user *may*
stay Bengali when the app is a Bengali product (`font-bangla`, `৳{price}`, Bengali labels
and headings). That is content, not code. The rule above governs code and repo artifacts only.

If existing code in a file already has Bangla/Banglish comments, translate them to English
while editing that file rather than leaving a mix.

## API response contract — pick one, then be consistent
Two shapes exist in his repos. **Default to A** unless editing a file that already uses B.

- **A — `{ success, message, data }`** (`Kazir_Haat_server`, his cleanest backend). Sent with
  `res.status(code).send({ ... })` — note `.send()`, not `.json()`. List endpoints add a
  sibling `pagination` object. Frontend checks `res?.success` before using `res.data`.
- **B — `res.status(code).send({ message, <namedPayload> })`** with no envelope
  (`ChatWebApplication`, `E-commece-FullStack`). Payload sits under a domain key
  (`user`, `productList`, `properties`).

A `utils/responseHandler.js` helper exists in his repos but is **mostly unused** — controllers
inline the response object by hand. Do the same.

## Routing
- **Backend** → `references/backend.md`
- **React (Vite)** → `createBrowserRouter(createRoutesFromElements(...))`, imports from
  **`react-router`** (v7), *not* `react-router-dom`. Some newer code uses declarative
  `<BrowserRouter><Routes>` instead. → `references/frontend.md`
- **Next.js** → App Router, route groups `(auth)/(admin)/(e-commerce)`, `lib/apiClient.js`
  fetch wrapper, `@/*` alias, middleware file named **`proxy.js`** using `jose`.
  → `references/frontend.md`
- **Full-stack** → build backend route/controller/model first, then the matching frontend
  service + component, keeping the response contract identical on both sides.

## Backend skeleton
Flat root, **no `src/`**: `controllers/ models/ routes/ middleware/ utils/ services/ dbConfig/ uploads/ index.js`
- CommonJS `require`/`module.exports`. `require('dotenv').config()` at the top of `index.js`.
- Mongoose. Conn string `process.env.DB_STRING`; JWT `process.env.JWT_SECRET` (sometimes `JWT_SEC`).
- Models: file `xSchema.js`, the variable `xSchema` **holds the compiled model**, registered
  lowercase-singular: `module.exports = mongoose.model('user', userSchema)`.
- Routers: `const route = express.Router()` … `module.exports = route`; mounted in `routes/index.js`.
- `authMiddleware` reads a **cookie first**, `Authorization: Bearer` as fallback.
  `roleCheck(...roles)` is a curried variadic factory.

## Frontend skeleton
- **React (Vite):** `src/{api,components/{admin,common},pages,layout,assets,store}` + `App.jsx` + `main.jsx`
- **Next.js:** `app/` with `(auth)/(e-commerce)/(admin)` groups,
  `app/components/{admin,auth,ecommerce,shared}`, `lib/apiClient.js`, path alias `@/*`
- **Tailwind v4**, no `tailwind.config.js` — design tokens live in `index.css`/`globals.css`
  inside an `@theme { }` block (`--color-primary`, `--font-bangla`, …). Inline utility classes,
  heavy arbitrary values (`text-[13px]`, `rounded-[32px]`).
- Interactive Next.js components start with `"use client";`.
- **No form library** (no react-hook-form/zod/yup). Controlled `useState`, guard-clause
  validation with early return.
- Icons: `react-icons` (or `lucide-react` in chat apps). Animation: `framer-motion`.
  Toasts: `react-hot-toast` (`position: 'top-center'`) or `react-toastify` (`theme="dark"`).

## Reference files
- `references/backend.md` — Express entry, dbConfig, Mongoose schema with bcrypt pre-hook,
  controller/router patterns, cookie `authMiddleware`, `roleCheck` factory, pagination.
- `references/frontend.md` — React (Vite) component + React Router v7 config + axios service
  object; Next.js App Router page/layout, `lib/apiClient.js`, `proxy.js` middleware.
- `references/image-upload.md` — **multer + Cloudinary**, both storage variants, what gets
  persisted to MongoDB, and the frontend `FormData` pattern. Read this for ANY file/image upload.
- `references/state-and-data.md` — RTK Query (`apiSlice` + `injectEndpoints`), `createSlice`,
  cookie auth helpers, `ProtectedRoute`. Read this before wiring a data layer.
- `references/realtime-socket.md` — socket.io client/server setup, event names, room patterns,
  WebRTC call signaling.
