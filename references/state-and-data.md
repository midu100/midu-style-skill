# State & data layer reference (midu-style)

Two data layers exist across his repos. **Newer projects use RTK Query; older ones use
axios service objects.** Never both in a new project (`Air_Bnb` has both only because the
axios layer is leftover legacy).

- **No axios in RTK Query projects. No TanStack Query, no SWR, no Zustand, no Context for data.**
- Shared UI state: `createSlice` when needed, otherwise lift with `useState` + props.
- `Kazir_Haat_Client` uses **no store at all** — just `useState`/`useEffect` and
  `window.dispatchEvent(new Event('cartUpdated'))` for cross-component signalling.

---

## Option 1 — RTK Query (preferred for new work)

### src/store/apiSlice.js — one root API, re-auth on 401
```js
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";
import { getCookie } from "../components/common/Services";

const baseQuery = fetchBaseQuery({
  baseUrl: import.meta.env.VITE_API_BASE_URL || "http://localhost:8000",
  credentials: "include",
  prepareHeaders: (headers) => {
    const token = getCookie("X_AS-TOKEN");
    if (token) headers.set("Authorization", `${token}`);   // no "Bearer " prefix
    return headers;
  },
});

const baseQueryWithReauth = async (args, api, extraOptions) => {
  let result = await baseQuery(args, api, extraOptions);
  if (result.error && result.error.status === 401) {
    const refreshResult = await baseQuery({ url: "/auth/refreshtoken", method: "POST" }, api, extraOptions);
    if (refreshResult.data) result = await baseQuery(args, api, extraOptions);
  }
  return result;
};

export const apiSlice = createApi({
  reducerPath: "api",
  baseQuery: baseQueryWithReauth,
  tagTypes: ["Property", "Category", "Booking", "Review", "User"],
  endpoints: () => ({}),
});
```

### src/store/api/propertyApi.js — one file per domain via `injectEndpoints`
```js
import { apiSlice } from "../apiSlice";

export const propertyApi = apiSlice.injectEndpoints({
  endpoints: (builder) => ({
    getProperties: builder.query({
      query: (params) => ({ url: "/property/all", params }),
      providesTags: (result) =>
        result
          ? [...result.properties.map(({ _id }) => ({ type: "Property", id: _id })),
             { type: "Property", id: "LIST" }]
          : [{ type: "Property", id: "LIST" }],
    }),
    createProperty: builder.mutation({
      query: (formData) => ({ url: "/property/create", method: "POST", body: formData }),
      invalidatesTags: [{ type: "Property", id: "LIST" }],
    }),
    updateProperty: builder.mutation({
      query: ({ id, formData }) => ({ url: `/property/update/${id}`, method: "PUT", body: formData }),
      invalidatesTags: (result, error, { id }) => [{ type: "Property", id }, { type: "Property", id: "LIST" }],
    }),
  }),
});

export const { useGetPropertiesQuery, useCreatePropertyMutation, useUpdatePropertyMutation } = propertyApi;
```
Tag convention: lists cache under `{ type, id: "LIST" }`; single items under `{ type, id }`.

### src/store/store.js
```js
import { configureStore } from "@reduxjs/toolkit";
import { apiSlice } from "./apiSlice";
import authReducer from "./slices/authSlice";
import cartReducer from "./slices/cartSlice";

export const store = configureStore({
  reducer: { [apiSlice.reducerPath]: apiSlice.reducer, auth: authReducer, cart: cartReducer },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({ serializableCheck: false }).concat(apiSlice.middleware),
});
```
In Next.js he scopes it instead: `<ApiProvider api={adminApiService}>` inside `(admin)/layout.jsx`,
with no global store at all.

### Consuming
```jsx
const { data, isLoading, error, refetch } = useGetPropertiesQuery();
const properties = data?.properties || [];

const [createProperty, { isLoading: creating }] = useCreatePropertyMutation();
try {
  const res = await createProperty(fd).unwrap();
  toast.success(res.message || 'Created', { duration: 3000, position: 'top-center' });
} catch (err) {
  toast.error(err?.data?.message || 'Something went wrong', { position: 'top-center' });
}
```
Always `.unwrap()` on mutations. Always `data?.key || []` on queries.

### createSlice — selectors co-located and exported from the slice file
```js
import { createSlice } from '@reduxjs/toolkit';

const authSlice = createSlice({
  name: 'auth',
  initialState: { user: null, token: null, isAuthenticated: false },
  reducers: {
    setCredentials: (state, action) => {
      state.user = action.payload.user;
      state.token = action.payload.token;
      state.isAuthenticated = true;
    },
    logout: (state) => {
      state.user = null;
      state.token = null;
      state.isAuthenticated = false;
    },
  },
});

export const { setCredentials, logout } = authSlice.actions;
export const selectCurrentUser = (state) => state.auth.user;
export const selectIsAuthenticated = (state) => state.auth.isAuthenticated;
export default authSlice.reducer;
```
`cartSlice` persists to `localStorage` behind `try/catch` helpers (`loadCartFromStorage`/
`saveCartToStorage`, key `'airbnb_cart'`).

---

## Option 2 — axios instance + service objects (`Kazir_Haat_Client`)

`src/api/index.js` is the **entire** data layer. One `axios.create`, one request interceptor,
then a named exported object per domain. Every method is `async` and returns `res.data`.

```js
import axios from 'axios'
import { getCookie } from '../components/common/Services'

const expressBaseUrl = 'https://kazir-haat-server.onrender.com'

const api = axios.create({
    baseURL: expressBaseUrl,
    withCredentials: true, // critical for cookies to work
    headers: { 'Content-Type': 'application/json' }
})

// ====== Attach token
api.interceptors.request.use(
    (config) => {
        const token = getCookie('token')
        if (token) config.headers.Authorization = `Bearer ${token}`
        return config
    },
    (error) => Promise.reject(error)
)

// ====== Auth services
export const authServices = {
    signUp: async (payload) => { const res = await api.post('/auth/signup', payload); return res.data },
    signIn: async (payload) => { const res = await api.post('/auth/signin', payload); return res.data },
    getProfile: async () => { const res = await api.get('/auth/profile'); return res.data },
}

// ====== Product services
export const productServices = {
    getProducts: async (params) => { const res = await api.get('/product/all', { params }); return res.data },
    addProduct: async (formData) => {
        const res = await api.post('/product/add', formData, {
            headers: { 'Content-Type': 'multipart/form-data' }
        })
        return res.data
    },
}

export default api
```
Consumed with `useState` + `useEffect` + `try/catch/finally` in every page (no query cache).
Parallel loads use `Promise.all`.

---

## Cookie auth helpers — `src/components/common/Services.js`
Hand-rolled, no library. This file is always named `Services.js` (PascalCase, `.js`).
```js
export const getCookie = (name) => {
  const value = `; ${document.cookie}`;
  const parts = value.split(`; ${name}=`);
  if (parts.length === 2) return parts.pop().split(';').shift();
  return null;
};

export const setCookie = (name, value, days = 7) => {
  const date = new Date();
  date.setTime(date.getTime() + days * 24 * 60 * 60 * 1000);
  document.cookie = `${name}=${value || ""}; expires=${date.toUTCString()}; path=/; SameSite=Lax; Secure`;
};

export const deleteCookie = (name) => {
  document.cookie = `${name}=; Path=/; Expires=Thu, 01 Jan 1970 00:00:01 GMT;`;
};
```
Cookie names actually used: `token` (Kazir_Haat), `X_AS-TOKEN` + `R_FS-TOKEN` (Air_Bnb,
E-commerce), `acc_tkn` + `ref_tkn` (ChatWebApplication). Logout clears the cookie manually
and dispatches `logout()`.

---

## ProtectedRoute — role gate, redirect with `state.from`
```jsx
import { Navigate, useLocation, Outlet } from 'react-router';
import { useSelector } from 'react-redux';
import { selectCurrentUser, selectIsAuthenticated } from '../store/slices/authSlice';

const ProtectedRoute = ({ allowedRoles, children }) => {
  const location = useLocation();
  const user = useSelector(selectCurrentUser);
  const isAuthenticated = useSelector(selectIsAuthenticated);

  if (!isAuthenticated) return <Navigate to="/login" state={{ from: location }} replace />;
  if (allowedRoles && !allowedRoles.includes(user?.role)) return <Navigate to="/" replace />;

  return children ? children : <Outlet />;
};

export default ProtectedRoute;
```
Used as `<ProtectedRoute allowedRoles={["admin", "host"]}><AdminLayout /></ProtectedRoute>`.

In the store-less variant (`Kazir_Haat_Client`) the same component calls
`authServices.getProfile()` on mount, shows a spinner while `loading`, then redirects.

A sibling `PersistAuth` wrapper restores Redux state after a refresh: if the cookie exists but
`isAuthenticated` is false, it fires `useGetProfileQuery(undefined, { skip: isAuthenticated || !hasCookie })`
and dispatches `setCredentials({ user: data.userData, token: null })`.

---

## Forms — no library, ever
```jsx
const [formData, setFormData] = useState({ email: '', password: '' });
const [errors, setErrors] = useState('');   // a single STRING, not an object

const handleChange = (e) => {
  const { name, value } = e.target;
  setFormData((prev) => ({ ...prev, [name]: value }));
};

const handleLogin = async (e) => {
  e.preventDefault();
  if (!formData.email) return setErrors('Email is required.');
  if (!formData.password) return setErrors('Password is required.');
  // ...submit
};
```
Errors render in one banner:
```jsx
{errors && <p className="mb-5 bg-amber-300 rounded-md py-2 text-center text-red-500 font-medium">{errors}</p>}
```
Where a toast library is present, validation short-circuits into it instead:
`if (!fullName) return toast.error("Full Name is required");`

After a success toast he navigates on a delay so the toast is visible:
`setTimeout(() => navigate('/admin/properties'), 1500);`
