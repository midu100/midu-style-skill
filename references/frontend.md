# Frontend reference — React (Vite) + Next.js (midu-style)

JavaScript only. `const Name = () => {}` then `export default Name` at the bottom.
`import React from 'react'` at the top even on React 19. Tailwind v4 inline classes.

Data layer (RTK Query vs axios services) → `state-and-data.md`.
File uploads → `image-upload.md`. Sockets → `realtime-socket.md`.

---

## React (Vite)

### Folder tree
```
src/
├── api/
│   └── index.js            # axios instance + service objects  (older projects)
├── store/                  # RTK Query + slices              (newer projects)
│   ├── store.js
│   ├── apiSlice.js
│   ├── api/                # propertyApi.js, authApi.js, …
│   └── slices/             # authSlice.js, cartSlice.js
├── components/
│   ├── admin/
│   ├── common/             # Services.js (cookie helpers), ProtectedRoute.jsx, cards, modals
│   └── Navbar.jsx …
├── pages/
│   ├── Home.jsx
│   ├── SignIn.jsx
│   └── admin/
├── layout/
│   ├── LayoutOne.jsx
│   └── AdminLayout.jsx
├── assets/
├── App.jsx
├── main.jsx
└── index.css
```
Layout components are numbered: `LayoutOne`, `ButtonOne`, `ButtonTwo`.

### vite.config.js — Tailwind v4 via the Vite plugin, no tailwind.config.js
```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [react(), tailwindcss()],
})
```

### src/index.css — design tokens live in `@theme`
```css
@import "tailwindcss";

/* ===== Custom Theme Configuration ===== */
@theme {
  --color-primary: #1A5D45;
  --color-secondary: #F28B00;
  --font-poppins: 'Poppins', sans-serif;
  --font-bangla: 'Hind Siliguri', sans-serif;
  --color-admin-sidebar: #1e293b;
}
```
Tokens are then used as ordinary utilities: `bg-primary`, `text-secondary`, `font-bangla`.
Arbitrary values are everywhere: `text-[13px]`, `rounded-[32px]`,
`shadow-[0_30px_80px_-20px_rgba(100,60,180,0.15)]`.

### src/App.jsx (React Router v7 — import from `react-router`, NOT `react-router-dom`)
```jsx
import React from 'react'
import { createBrowserRouter, createRoutesFromElements, Route, RouterProvider } from 'react-router'
import LayoutOne from './layout/LayoutOne'
import AdminLayout from './layout/AdminLayout'
import ProtectedRoute from './components/common/ProtectedRoute'
import Home from './pages/Home'
import SignIn from './pages/SignIn'
import AdminProducts from './pages/admin/AdminProducts'

const App = () => {
  const myRoute = createBrowserRouter(
    createRoutesFromElements(
      <>
        {/* ====== Public ====== */}
        <Route path="/" element={<LayoutOne />}>
          <Route index element={<Home />} />
        </Route>
        <Route path="/signin" element={<SignIn />} />

        {/* ====== Admin ====== */}
        <Route
          path="/admin"
          element={
            <ProtectedRoute allowedRoles={['admin']}>
              <AdminLayout />
            </ProtectedRoute>
          }
        >
          <Route path="products" element={<AdminProducts />} />
        </Route>
      </>
    )
  )

  return <RouterProvider router={myRoute} />
}

export default App
```
Newer code (`Air_Bnb`) uses the declarative form instead — both are acceptable:
```jsx
<BrowserRouter>
  <Routes>
    <Route path="/" element={<LayoutOne />}>
      <Route index element={<Home />} />
    </Route>
  </Routes>
</BrowserRouter>
```

### src/main.jsx
```jsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { Provider } from 'react-redux'
import { store } from './store/store'
import App from './App'
import './index.css'

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <Provider store={store}>
      <App />
    </Provider>
  </StrictMode>
)
```

### src/layout/LayoutOne.jsx
```jsx
import React from 'react'
import { Outlet } from 'react-router'
import Navbar from '../components/Navbar'
import Footer from '../components/Footer'

const LayoutOne = () => {
  return (
    <div className="min-h-screen flex flex-col">
      <Navbar />
      <main className="flex-1">
        <Outlet />
      </main>
      <Footer />
    </div>
  )
}

export default LayoutOne
```

### src/pages/SignIn.jsx (component pattern + fetch + guard-clause validation)
```jsx
import React, { useState } from 'react'
import { useNavigate } from 'react-router'
import toast from 'react-hot-toast'
import { authServices } from '../api'
import { setCookie } from '../components/common/Services'

const SignIn = () => {
  const [formData, setFormData] = useState({ email: '', password: '' })
  const [errors, setErrors] = useState('')
  const [loading, setLoading] = useState(false)
  const navigate = useNavigate()

  // ====== Submit
  const handleSignIn = async (e) => {
    e.preventDefault()
    if (!formData.email) return setErrors('Email is required.')
    if (!formData.password) return setErrors('Password is required.')

    try {
      setLoading(true)
      const res = await authServices.signIn(formData)
      if (res?.success) {
        setCookie('token', res.data.token)
        toast.success(res.message, { duration: 3000, position: 'top-center' })
        setTimeout(() => navigate('/'), 1500)
      }
    } catch (err) {
      console.error(err)
      setErrors(err?.response?.data?.message || 'Something went wrong')
    } finally {
      setLoading(false)
    }
  }

  return (
    <form onSubmit={handleSignIn} className="max-w-sm mx-auto mt-10 flex flex-col gap-3">
      {errors && (
        <p className="mb-5 bg-amber-300 rounded-md py-2 text-center text-red-500 font-medium">{errors}</p>
      )}
      <input
        className="border border-gray-300 rounded-lg px-3 py-2 outline-none focus:border-primary"
        placeholder="Email"
        value={formData.email}
        onChange={(e) => { setFormData((p) => ({ ...p, email: e.target.value })); setErrors('') }}
      />
      <input
        type="password"
        className="border border-gray-300 rounded-lg px-3 py-2 outline-none focus:border-primary"
        placeholder="Password"
        value={formData.password}
        onChange={(e) => { setFormData((p) => ({ ...p, password: e.target.value })); setErrors('') }}
      />
      <button
        disabled={loading}
        className="bg-primary text-white rounded-lg py-2 font-medium active:scale-95 cursor-pointer disabled:opacity-60"
      >
        {loading ? 'Signing in...' : 'Sign In'}
      </button>
    </form>
  )
}

export default SignIn
```

### Loading / error / empty — three ternary branches, same visual language
```jsx
{isLoading ? (
  <p className="text-center py-20 font-black uppercase tracking-wider">Loading…</p>
) : error ? (
  <p className="text-center py-20 font-black uppercase tracking-wider text-red-500">Failed to load</p>
) : products.length === 0 ? (
  <div className="border border-dashed rounded-2xl py-20 text-center">No products found</div>
) : (
  <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
    {products.map((p) => <ProductCard key={p._id} product={p} />)}
  </div>
)}
```

### Libraries he reaches for
`react-icons` (`/fa`, `/fi`, `/lu`, `/hi`) or `lucide-react` · `framer-motion` for entrance
and hover animation · `react-slick` for carousels · `chart.js` + `react-chartjs-2` for admin
charts · `aos` for scroll reveals · **no** component library (no shadcn, no DaisyUI, no MUI).
Toasts: `react-hot-toast` (`position: 'top-center'`) or `react-toastify`
(`<ToastContainer position="top-right" autoClose={3000} theme="dark" />`).

Bilingual UI is normal: Bengali strings with `font-bangla`, `৳{price}` currency. This covers
**rendered user-facing copy only** — comments, variable names and commit messages stay English
(see the Language rule in `SKILL.md`).

---

## Next.js (App Router)

### Folder tree
```
app/
├── (auth)/
│   ├── layout.jsx
│   ├── signin/page.jsx
│   └── verify-email/page.jsx
├── (e-commerce)/
│   ├── layout.js
│   ├── page.js
│   ├── shop/  cart/  checkout/  product/
├── (admin)/
│   ├── layout.jsx
│   ├── services/api.js        # RTK Query, provided via <ApiProvider>
│   └── admin/products/page.jsx
├── components/
│   ├── admin/  auth/  ecommerce/  shared/
├── globals.css
├── layout.js
└── page.js
lib/
├── apiClient.js
└── utils.js
proxy.js            # ← the middleware file, named proxy.js
jsconfig.json       # { "paths": { "@/*": ["./*"] } }
next.config.mjs
```
Convention: composition-only route files are `.js` (`layout.js`, `page.js`); interactive ones
are `.jsx`. Components are always `.jsx`.

### lib/apiClient.js (fetch wrapper — used by the public/e-commerce side)
```js
const BASE_URL = process.env.NEXT_PUBLIC_API_BASE_URL || 'http://localhost:8000'

const request = async (endpoint, { method = 'GET', body, ...options } = {}) => {
  const res = await fetch(`${BASE_URL}${endpoint}`, {
    method,
    credentials: 'include',
    headers: { 'Content-Type': 'application/json' },
    ...(body && { body: JSON.stringify(body) }),
    ...options,
  })
  if (!res.ok) throw new Error(`Request failed: ${res.status}`)
  return await res.json()
}

const apiClient = {
  get: (endpoint, options) => request(endpoint, { ...options }),
  post: (endpoint, body, options) => request(endpoint, { method: 'POST', body, ...options }),
  put: (endpoint, body, options) => request(endpoint, { method: 'PUT', body, ...options }),
  delete: (endpoint, options) => request(endpoint, { method: 'DELETE', ...options }),
}

export default apiClient
```
The admin side uses RTK Query instead, scoped with `<ApiProvider api={adminApiService}>`
in `(admin)/layout.jsx` — there is no global store in his Next.js apps.

### app/(auth)/signin/page.jsx ("use client" for interactive)
```jsx
"use client";

import React, { useState } from 'react'
import { useRouter } from 'next/navigation'
import toast, { Toaster } from 'react-hot-toast'
import apiClient from '@/lib/apiClient'

const SignInPage = () => {
  const [formData, setFormData] = useState({ email: '', password: '' })
  const [errors, setErrors] = useState('')
  const router = useRouter()

  // ====== Submit
  const handleLogin = async (e) => {
    e.preventDefault()
    if (!formData.email) return setErrors('Email is required.')
    if (!formData.password) return setErrors('Password is required.')

    try {
      const res = await apiClient.post('/auth/signin', formData)
      toast.success(res.message, { duration: 4000, position: 'top-center' })
      setTimeout(() => router.push('/'), 1500)
    } catch (error) {
      console.log(error)
      setErrors('Login failed')
    }
  }

  return (
    <>
      <Toaster />
      <form onSubmit={handleLogin} className="max-w-sm mx-auto mt-10 flex flex-col gap-3">
        {errors && <p className="text-red-500 text-center">{errors}</p>}
        <input
          className="border rounded px-3 py-2"
          placeholder="Email"
          value={formData.email}
          onChange={(e) => setFormData((p) => ({ ...p, email: e.target.value }))}
        />
        <button className="bg-black text-white rounded py-2 cursor-pointer">Sign In</button>
      </form>
    </>
  )
}

export default SignInPage
```

### app/(e-commerce)/shop/page.jsx (server component fetch)
Purely presentational components omit `"use client"` and stay Server Components.
```jsx
import React from 'react'
import apiClient from '@/lib/apiClient'
import SingleProduct from '@/app/components/shared/SingleProduct'

const ShopPage = async () => {
  const res = await apiClient.get('/product/all')
  const products = res?.data || []

  return (
    <div className="grid grid-cols-2 md:grid-cols-4 gap-4 p-4">
      {products.map((p) => (
        <SingleProduct key={p._id} name={p.name} src={p.images[0]?.url} price={p.price} />
      ))}
    </div>
  )
}

export default ShopPage
```

### proxy.js — the middleware, verified with `jose` (Edge-compatible)
He names the file `proxy.js` and exports `proxy`, not `middleware`.
```js
import { NextResponse } from 'next/server'
import { jwtVerify } from 'jose'

const secret = new TextEncoder().encode(process.env.JWT_SEC)

export async function proxy(request) {
  const { pathname } = request.nextUrl

  if (pathname.startsWith('/admin')) {
    const token = request.cookies.get('R_FS-TOKEN')?.value
    if (!token) return NextResponse.redirect(new URL('/signin', request.url))

    try {
      const { payload } = await jwtVerify(token, secret)
      const allowedRoles = ['admin', 'super-admin']
      if (!allowedRoles.includes(payload.role)) return NextResponse.redirect(new URL('/', request.url))
      return NextResponse.next()
    } catch (error) {
      request.cookies.delete('X_AS-TOKEN')
      request.cookies.delete('R_FS-TOKEN')
      return NextResponse.redirect(new URL('/signin', request.url))
    }
  }

  return NextResponse.next()
}

export const config = { matcher: ['/admin/:path*'] }
```

### next.config.mjs — whitelist Cloudinary for next/image
```js
const nextConfig = {
  images: {
    remotePatterns: [{ protocol: 'https', hostname: 'res.cloudinary.com' }],
  },
}

export default nextConfig
```

### Markup habits
- Section wrappers use Bootstrap-style class names inside Tailwind:
  `<div className="container">` → `<div className="row flex justify-center">`
- Props destructured in the signature; capitalized prop keys appear (`{ Name, path }`, `{ Name, onClick }`)
- `.map((item, i) => ... key={i})` — index keys for static arrays, `key={p._id}` for API data
- Buttons get `cursor-pointer active:scale-95`
