# Backend reference — Express + Mongoose (midu-style)

CommonJS, Express 5, Mongoose 9, `bcrypt` (not bcryptjs). **Flat root, no `src/`.**
Start script is `"start": "node --watch index.js"` — no nodemon, no test runner.

For file/image uploads see `image-upload.md`. For socket.io see `realtime-socket.md`.

## Folder tree
```
backend/
├── controllers/
│   ├── authController.js
│   └── productController.js
├── models/
│   ├── userSchema.js
│   └── productSchema.js
├── routes/
│   ├── index.js
│   ├── auth.js
│   └── product.js
├── middleware/
│   ├── authMiddleware.js
│   ├── roleCheckMiddleware.js
│   └── upload.js
├── services/           # emailServices.js, helpers.js, regexValidation.js, template.js
├── utils/              # cloudinaryConfig.js, cloudinaryService.js, responseHandler.js
├── dbConfig/
│   └── index.js
├── uploads/
├── .env
├── index.js
└── package.json
```
Route files are bare nouns (`auth.js`, `product.js`), controllers are `xController.js`,
models are `xSchema.js`.

## index.js (entry)
```js
require('dotenv').config()
const express = require('express')
const cors = require('cors')
const cookieParser = require('cookie-parser')
const dbConfig = require('./dbConfig')
const route = require('./routes')

const app = express()
const port = process.env.PORT || 8000

// ====== Middleware
app.use(cors({ origin: process.env.CLIENT_URL, credentials: true }))
app.use(express.json())
app.use(express.urlencoded({ extended: true }))
app.use(cookieParser())

// ====== Database
dbConfig()

// ====== Routes
app.use(route)
app.get('/', (req, res) => res.status(200).send({ success: true, message: 'API is running 🚀' }))

// ====== Server
app.listen(port, () => {
  console.log(`server is running on port ${port}`)
})
```

## dbConfig/index.js (.then only for the connect line)
```js
const mongoose = require('mongoose')

const dbConfig = () => {
    mongoose
        .connect(process.env.DB_STRING)
        .then(() => console.log('DB Connected!'))
        .catch((err) => console.log('DB connection error:', err.message))
}

module.exports = dbConfig
```

## models/userSchema.js
The variable is named `userSchema` and **holds the compiled model** — controllers do
`const userSchema = require('../models/userSchema')` then `userSchema.findOne(...)`.
Model name is registered lowercase-singular.
```js
const mongoose = require('mongoose')
const bcrypt = require('bcrypt')

const userSchema = new mongoose.Schema(
    {
        avatar: { public_id: { type: String }, url: { type: String } },
        fullName: { type: String },
        email: { type: String, required: true, unique: true },
        password: { type: String, required: true },
        phone: { type: String },
        role: { type: String, default: 'user', enum: ['admin', 'user'] },
        isVerified: { type: Boolean, default: false },
        otp: { type: Number, default: null },
        otpExpire: { type: Date },
    },
    { timestamps: true }
)

// ====== Hash password before save
userSchema.pre('save', async function () {
    const user = this
    if (!user.isModified('password')) return
    try {
        user.password = await bcrypt.hash(user.password, 10)
    } catch (err) {
        console.log(err)
    }
})

// ====== Compare password
userSchema.methods.comparePassword = async function (candidatePassword) {
    return bcrypt.compare(candidatePassword, this.password)
}

module.exports = mongoose.model('user', userSchema)
```
Refs: `{ type: mongoose.Types.ObjectId, ref: 'user' }`. `{ timestamps: true }` on most models.

## services/helpers.js
```js
const jwt = require('jsonwebtoken')

// ====== Generate JWT Token
const generateToken = (id) =>
    jwt.sign({ id }, process.env.JWT_SECRET, { expiresIn: process.env.JWT_EXPIRE || '7d' })

// ====== 4-digit OTP
const generateOtp = () => Math.floor(1000 + Math.random() * 9000)

module.exports = { generateToken, generateOtp }
```

## controllers/authController.js
`{ success, message, data }` sent with `.send()` (not `.json()`). Guard clauses return early.
Catch logs and returns a 500. Token goes in an httpOnly cookie **and** in `data.token`.
```js
const userSchema = require('../models/userSchema')
const { generateToken } = require('../services/helpers')

// ====== Sign Up
const signUp = async (req, res) => {
    try {
        const { fullName, email, password } = req.body
        if (!email) return res.status(400).send({ success: false, message: 'Email is required' })
        if (!password) return res.status(400).send({ success: false, message: 'Password is required' })

        const isExist = await userSchema.findOne({ email })
        if (isExist) return res.status(400).send({ success: false, message: 'User already exists' })

        const user = await userSchema.create({ fullName, email, password })

        return res.status(201).send({
            success: true,
            message: 'Registration successful. Please verify your email.',
            data: { _id: user._id, fullName: user.fullName, email: user.email },
        })
    } catch (error) {
        console.log(error)
        return res.status(500).send({ success: false, message: 'Internal server error' })
    }
}

// ====== Sign In
const signIn = async (req, res) => {
    try {
        const { email, password } = req.body
        if (!email || !password)
            return res.status(400).send({ success: false, message: 'Email and password are required' })

        const user = await userSchema.findOne({ email })
        if (!user || !(await user.comparePassword(password)))
            return res.status(401).send({ success: false, message: 'Invalid credentials' })

        const token = generateToken(user._id)
        const isSecure = process.env.NODE_ENV === 'production'

        res.cookie('token', token, {
            httpOnly: true,
            secure: isSecure,
            sameSite: isSecure ? 'none' : 'lax',
            maxAge: 7 * 24 * 60 * 60 * 1000,
        })

        return res.status(200).send({
            success: true,
            message: 'Login successful',
            data: { _id: user._id, fullName: user.fullName, email: user.email, role: user.role, token },
        })
    } catch (error) {
        console.log(error)
        return res.status(500).send({ success: false, message: 'Internal server error' })
    }
}

module.exports = { signUp, signIn }
```

## Pagination / search / filter (list endpoints)
```js
// ====== Get Products
const getProducts = async (req, res) => {
    try {
        const { page = 1, limit = 12, search, sort } = req.query
        const skip = (Number(page) - 1) * Number(limit)

        const query = {}
        if (search) query.name = { $regex: search, $options: 'i' }

        const sortMap = { price_asc: { price: 1 }, price_desc: { price: -1 }, newest: { createdAt: -1 } }

        const total = await productSchema.countDocuments(query)
        const products = await productSchema
            .find(query)
            .sort(sortMap[sort] || { createdAt: -1 })
            .skip(skip)
            .limit(Number(limit))

        return res.status(200).send({
            success: true,
            message: 'Products fetched.',
            data: products,
            pagination: {
                total,
                page: Number(page),
                limit: Number(limit),
                totalPages: Math.ceil(total / Number(limit)),
            },
        })
    } catch (error) {
        console.log(error)
        return res.status(500).send({ success: false, message: 'Internal server error' })
    }
}
```

## middleware/authMiddleware.js (cookie first, Bearer fallback)
```js
const jwt = require('jsonwebtoken')
const userSchema = require('../models/userSchema')

const authMiddleware = async (req, res, next) => {
    try {
        let token = req.cookies.token
        if (!token && req.headers.authorization && req.headers.authorization.startsWith('Bearer ')) {
            token = req.headers.authorization.split(' ')[1]
        }
        if (!token) return res.status(401).send({ success: false, message: 'Unauthorized. Please login.' })

        const decoded = jwt.verify(token, process.env.JWT_SECRET)
        const user = await userSchema.findById(decoded.id).select('-password')
        if (!user) return res.status(401).send({ success: false, message: 'User not found.' })
        if (!user.isVerified)
            return res.status(403).send({ success: false, message: 'Please verify your email first.' })

        req.user = user
        next()
    } catch (error) {
        console.log(error)
        return res.status(401).send({ success: false, message: 'Invalid or expired token.' })
    }
}

module.exports = { authMiddleware }
```

## middleware/roleCheckMiddleware.js (curried variadic factory)
```js
const roleCheck = (...roles) => (req, res, next) => {
    if (!req.user) return res.status(401).send({ success: false, message: 'Unauthorized. Please login.' })
    if (!roles.includes(req.user.role))
        return res.status(403).send({ success: false, message: 'Access denied. You do not have permission.' })
    next()
}

module.exports = roleCheck
```
Used as `roleCheck('admin')` / `roleCheck('admin', 'editor')`.

## routes/product.js (the router variable is `route`, not `router`)
```js
const express = require('express')
const { getProducts, addProduct } = require('../controllers/productController')
const { authMiddleware } = require('../middleware/authMiddleware')
const roleCheck = require('../middleware/roleCheckMiddleware')
const upload = require('../middleware/upload')

const route = express.Router()

// ====== Public Routes
route.get('/all', getProducts)

// ====== Admin Routes
route.post('/add', authMiddleware, roleCheck('admin'), upload.array('images', 5), addProduct)

module.exports = route
```

## routes/index.js (central mount)
```js
const express = require('express')
const authRoute = require('./auth')
const productRoute = require('./product')
const { authMiddleware } = require('../middleware/authMiddleware')

const route = express.Router()

route.use('/auth', authRoute)
route.use('/product', productRoute)
route.use('/cart', authMiddleware, cartRoute)   // middleware can be applied at the mount

module.exports = route
```

## .env
```
PORT=8000
NODE_ENV=development
CLIENT_URL=http://localhost:5173
DB_STRING=mongodb://127.0.0.1:27017/yourdb
JWT_SECRET=your_secret_here
JWT_EXPIRE=7d
CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=
SMTP_USER=
```

## Notes on his real code
- `utils/responseHandler.js` (`successResponse`/`errorResponse`) exists in several repos but is
  **almost never used** — controllers inline the response object. Do the same.
- Email verification uses a 4-digit OTP with a 5-minute expiry stored on the user doc,
  cleared with `user.otp = undefined`. Same mechanism is reused for password reset.
- Emails go through `nodemailer` (gmail service) in `services/emailServices.js`, with HTML
  string templates in `services/template.js`.
- Payments: `sslcommerz-lts` (Bangladesh) or `stripe`, `paymentMethod: { enum: ['cod', 'sslcommerz'] }`.
- Some repos hardcode the port (`8000`, `1000`) instead of reading `process.env.PORT`. Prefer the env var.
