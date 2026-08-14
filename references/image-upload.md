# Image & file upload reference (midu-style)

**Storage is always Cloudinary. Never imgbb, never S3, never Firebase Storage.**
The client never talks to Cloudinary directly — it sends raw files as `multipart/form-data`
to his own Express backend, which uploads them and stores the resulting URL in MongoDB.

There are **two multer variants** in his repos. Prefer **Variant A** — it persists
`public_id`, which makes deletes correct. Use Variant B only when editing a repo that
already uses it.

---

## Variant A — multer diskStorage → Cloudinary → store `{ public_id, url }`
(from `Kazir_Haat_server` — the cleaner, recommended pattern)

### middleware/upload.js
```js
const multer = require('multer')
const path = require('path')

const storage = multer.diskStorage({
    destination: function (req, file, cb) {
        cb(null, path.join(__dirname, '../uploads'))
    },
    filename: function (req, file, cb) {
        const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1E9)
        cb(null, uniqueSuffix + path.extname(file.originalname))
    }
})

const fileFilter = (req, file, cb) => {
    const allowedTypes = ['image/jpeg', 'image/jpg', 'image/png', 'image/webp']
    if (allowedTypes.includes(file.mimetype)) cb(null, true)
    else cb(new Error('Only .jpeg, .jpg, .png and .webp files are allowed'), false)
}

const upload = multer({ storage, fileFilter, limits: { fileSize: 5 * 1024 * 1024 } })

module.exports = upload
```
A near-identical `middleware/videoUpload.js` uses `file.mimetype.startsWith('video/')` and 50MB.

### utils/cloudinaryConfig.js
```js
const cloudinary = require('cloudinary').v2

cloudinary.config({
    cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
    api_key: process.env.CLOUDINARY_API_KEY,
    api_secret: process.env.CLOUDINARY_API_SECRET
})

module.exports = cloudinary
```

### utils/cloudinaryService.js
```js
const cloudinary = require('./cloudinaryConfig')

// ====== Upload image
const uploadImage = async (filePath, folder = 'kazir-haat') => {
    try {
        const result = await cloudinary.uploader.upload(filePath, { folder, resource_type: 'image' })
        return { public_id: result.public_id, url: result.secure_url }
    } catch (error) {
        console.log('Cloudinary upload error:', error)
        throw new Error('Image upload failed')
    }
}

// ====== Delete image
const deleteImage = async (publicId) => {
    try {
        await cloudinary.uploader.destroy(publicId)
    } catch (error) {
        console.log('Cloudinary delete error:', error)
    }
}

module.exports = { uploadImage, deleteImage }
```

### What goes in the schema
```js
// productSchema.js — array of images
images: [ { public_id: { type: String }, url: { type: String } } ],

// categorySchema.js — single image
image: { public_id: { type: String }, url: { type: String } },
```

### Controller usage — loop `req.files`, delete old before replacing
```js
// ====== Add Product
const addProduct = async (req, res) => {
    try {
        let images = []
        if (req.files && req.files.length > 0) {
            for (let file of req.files) {
                const result = await uploadImage(file.path, 'kazir-haat/products')
                images.push(result)
            }
        }

        const size = typeof req.body.size === 'string' ? JSON.parse(req.body.size) : req.body.size

        const product = await productSchema.create({ ...req.body, size, images })
        return res.status(201).send({ success: true, message: 'Product added successfully.', data: product })
    } catch (error) {
        console.log(error)
        return res.status(500).send({ success: false, message: 'Internal server error' })
    }
}

// ====== On update: destroy the old images first
for (let img of product.images) {
    if (img.public_id) await deleteImage(img.public_id)
}
```
Cloudinary folder names are namespaced: `'kazir-haat/products'`, `'kazir-haat/categories'`, `'kazir-haat/videos'`.

### Route wiring — multer sits between roleCheck and the controller
```js
route.post('/add', authMiddleware, roleCheck('admin'), upload.array('images', 5), addProduct)
route.post('/create', authMiddleware, roleCheck('admin'), upload.single('image'), createCategory)
```

---

## Variant B — multer memory → base64 data URL → Cloudinary → store `secure_url` string
(from `E-commece-FullStack/server`)

Multer is instantiated bare, so files arrive as buffers with no temp file on disk:

```js
const cloudinary = require('cloudinary').v2

const uploadToCloudinary = async (file, folder) => {
  if (!file) return null
  const imageBase64 = file.buffer.toString('base64')
  const imageDataUrl = `data:${file.mimetype};base64,${imageBase64}`
  const result = await cloudinary.uploader.upload(imageDataUrl, { folder })
  return result                                   // caller uses result.secure_url
}

// NOTE: he names it deleteToCloudinary, and re-derives the public_id out of the stored URL
const deleteToCloudinary = async (file, folder) => {
  try {
    const publicId = file.split('/').pop().split('.').shift()
    await cloudinary.uploader.destroy(`${folder}/${publicId}`)
  } catch (error) { console.log(error) }
}

module.exports = { uploadToCloudinary, deleteToCloudinary }
```

Schema stores a **plain string**: `thumbnail: { type: String, required: true }`, `images: [{ type: String }]`, `avatar: { type: String }`.

```js
const upload = multer()   // memory storage
route.post('/upload', authMiddleware, roleCheckMiddleware('admin', 'editor'),
  upload.fields([{ name: 'thumbnail', maxCount: 1 }, { name: 'images', maxCount: 4 }]), createProduct)
route.put('/profile', authMiddleware, upload.single('avatar'), updateUserProfile)
```
```js
const thumbnail = req.files?.thumbnail?.[0]
const images = req.files?.images

const thumbnailUrl = await uploadToCloudinary(thumbnail, 'thumbnail')
const imagesUrl = []
if (images) {
  for (const img of images) {
    const imgUrl = await uploadToCloudinary(img, 'product')
    imagesUrl.push(imgUrl.secure_url)
  }
}
// stored: { thumbnail: thumbnailUrl.secure_url, images: imagesUrl }
```
Update flow = upload new, then delete old:
```js
const imgRes = await uploadToCloudinary(avatar, 'avatar')
deleteToCloudinary(user.avatar, 'avatar')
user.avatar = imgRes.secure_url
```

---

## Frontend — always a hand-built `FormData`, never an upload helper

Rules he follows without exception:
- Build `new FormData()` **inline in the submit handler**. There is no shared upload utility.
- Append scalars one at a time.
- **Arrays are `JSON.stringify`'d** — the backend re-parses defensively.
- **Multiple files reuse the same key** in a loop.
- File inputs are uncontrolled; the raw `File`/`FileList` goes straight into `useState`.

### Field-name conventions the backend expects
| Purpose | Field name |
|---|---|
| single cover image | `thumbnail` or `image` |
| image gallery (repeated key) | `images` |
| user avatar | `avatar` (or `profileImg` in `Air_Bnb`) |

### With axios services (`Kazir_Haat_Client`)
```jsx
const [images, setImages] = useState([]);

<input type="file" multiple accept="image/*" onChange={(e) => setImages(e.target.files)} />

const handleSubmit = async (e) => {
  e.preventDefault();
  const formData = new FormData();
  formData.append('name', name);
  formData.append('price', price);
  formData.append('discountPrice', discountPrice || 0);
  formData.append('category', category);
  formData.append('featured', featured);

  if (size) {
    const sizeArr = size.split(',').map((s) => s.trim());
    formData.append('size', JSON.stringify(sizeArr));
  }
  if (images.length > 0) {
    for (let i = 0; i < images.length; i++) formData.append('images', images[i]);
  }

  const res = editId
    ? await productServices.updateProduct(editId, formData)
    : await productServices.addProduct(formData);
  if (res?.success) setMessage(res.message);
};
```
The axios service sets the header explicitly:
```js
addProduct: async (formData) => {
    const res = await api.post('/product/add', formData, {
        headers: { 'Content-Type': 'multipart/form-data' }
    })
    return res.data
},
```

### With RTK Query (`Air_Bnb`, `E-commece-FullStack`)
Pass the `FormData` straight through as `body` — **do not set Content-Type**, RTK Query
detects `FormData` and sets the multipart boundary itself.
```js
createProperty: builder.mutation({
  query: (formData) => ({ url: '/property/create', method: 'POST', body: formData }),
  invalidatesTags: [{ type: 'Property', id: 'LIST' }],
}),
```
```jsx
const [createProperty, { isLoading }] = useCreatePropertyMutation();

const handleSubmit = async (data) => {
  if (!data.thumbnailFile) { toast.error('Thumbnail image is required!'); return; }

  const fd = new FormData();
  fd.append('title', data.title);
  fd.append('pricePerNight', data.pricePerNight);
  fd.append('amenities', JSON.stringify(data.amenities));
  fd.append('thumbnail', data.thumbnailFile);
  if (data.imagesFiles?.length > 0) {
    for (const img of data.imagesFiles) fd.append('images', img);
  }

  try {
    await createProperty(fd).unwrap();
    toast.success('Listing created successfully!');
    setTimeout(() => navigate('/admin/properties'), 1500);
  } catch (err) {
    toast.error(err?.data?.message || 'Failed to create listing');
  }
};
```

### Rendering uploaded images
- Variant A (`{public_id, url}`): `<img src={product.images[0]?.url} />`
- Variant B (string): `<img src={property.thumbnail || 'https://picsum.photos/400/300'} />`
- Avatar fallback: `src={user.profileImg || 'https://picsum.photos/200'}`
- Preview before upload: `URL.createObjectURL(file)`
- Next.js: whitelist `res.cloudinary.com` in `next.config.mjs` `images.remotePatterns`.

### Styling the file input (his actual classes)
```jsx
<input type="file" accept="image/*"
  onChange={(e) => setThumbnailFile(e.target.files[0])}
  className="w-full text-sm file:mr-4 file:py-2 file:px-4 file:rounded-full file:border-0
             file:bg-black file:text-white file:uppercase file:font-black cursor-pointer" />
```

## Env vars
```
CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=
```
(`E-commece-FullStack` uses the shorter `CLOUDINARY_NAME` / `CLOUDINARY_API_SEC` — match the repo.)

## Things he does NOT do
- No client-side upload widget, no signed uploads, no image transformations.
- No drag-and-drop, no compression.
- Product/category controllers do **not** clean up the local `uploads/` temp dir
  (only the video controller calls `fs.unlinkSync(req.file.path)`).
