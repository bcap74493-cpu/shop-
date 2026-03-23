# 🌿 FreshLeaf Backend API

> Full Node.js + Express + MongoDB REST API for the FreshLeaf Green Grocer frontend.

---

## 📁 Project Structure

```
freshleaf-backend/
├── server.js                  # Entry point (Express + Socket.io)
├── package.json
├── .env.example               # Copy to .env and fill in values
│
├── config/
│   ├── db.js                  # MongoDB connection
│   ├── cloudinary.js          # Multer + Cloudinary / disk storage
│   └── seed.js                # Seeds default owner account
│
├── models/
│   ├── User.js                # Customer + Owner (role field)
│   ├── Product.js
│   ├── Order.js               # Full pipeline status
│   └── index.js               # Offer, AdMedia, Rating, Notification, Settings
│
├── controllers/
│   ├── authController.js
│   ├── productController.js
│   ├── orderController.js
│   ├── offerController.js
│   ├── adMediaController.js
│   ├── ratingController.js
│   ├── notificationController.js
│   └── settingsController.js
│
├── routes/
│   ├── auth.js
│   ├── products.js
│   ├── orders.js
│   ├── offers.js
│   ├── adMedia.js
│   ├── ratings.js
│   ├── notifications.js
│   ├── settings.js
│   └── upload.js
│
├── middleware/
│   ├── auth.js                # JWT protect + ownerOnly
│   └── validate.js            # express-validator errors
│
└── uploads/                   # Local file storage (dev)
```

---

## ⚙️ Setup Instructions

### 1. Prerequisites
- Node.js 18+
- MongoDB (local or Atlas)
- (Optional) Cloudinary account for production file storage

### 2. Install dependencies
```bash
cd freshleaf-backend
npm install
```

### 3. Configure environment variables
```bash
cp .env.example .env
# Edit .env with your values:
# MONGO_URI, JWT_SECRET, OWNER_EMAIL, OWNER_PASSWORD, etc.
```

### 4. Seed the database (creates owner account + default settings)
```bash
node config/seed.js
```

### 5. Start the server
```bash
# Development (with auto-reload)
npm run dev

# Production
npm start
```

The API will be running at: **http://localhost:5000**

---

## 🔐 Authentication

All protected routes require:
```
Authorization: Bearer <JWT_TOKEN>
```

| Route | Access |
|-------|--------|
| `POST /api/auth/register` | Public |
| `POST /api/auth/login` | Public |
| `GET /api/auth/me` | Any logged-in user |
| `PUT /api/auth/me` | Any logged-in user |
| `POST /api/auth/change-password` | Any logged-in user |

---

## 📋 All API Endpoints

### 🔑 Auth
```
POST   /api/auth/register          Register a new customer
POST   /api/auth/login             Login (customer or owner)
GET    /api/auth/me                Get current user profile
PUT    /api/auth/me                Update profile / last address
POST   /api/auth/change-password   Change password
```

### 🥬 Products
```
GET    /api/products               List all active products (public)
GET    /api/products/:id           Get one product (public)
POST   /api/products               Create product (owner only)
PUT    /api/products/:id           Update product (owner only)
DELETE /api/products/:id           Soft-delete product (owner only)
```

### 🧾 Orders
```
POST   /api/orders                 Place order (customer)
GET    /api/orders                 All orders (owner only)
GET    /api/orders/my              Customer's own orders
GET    /api/orders/:id             Single order
PUT    /api/orders/:id/status      Update pipeline status (owner only)
PUT    /api/orders/:id/cancel      Cancel order (customer: confirmed only)
```

### 🔥 Offers
```
GET    /api/offers                 All active offers (public)
POST   /api/offers                 Create offer (owner only)
PUT    /api/offers/:id             Update offer (owner only)
DELETE /api/offers/:id             Delete offer (owner only)
```

### 📺 Ad Media
```
GET    /api/ad-media               All active media (public)
POST   /api/ad-media               Add media slot (owner only)
PUT    /api/ad-media/reorder       Reorder slots (owner only)
PUT    /api/ad-media/:id           Update caption (owner only)
DELETE /api/ad-media/:id           Remove slot (owner only)
```

### ⭐ Ratings
```
GET    /api/ratings                All ratings with averages (owner only)
GET    /api/ratings/public         Public rating summary
POST   /api/ratings                Submit rating (customer, delivered orders only)
```

### 🔔 Notifications
```
GET    /api/notifications/owner    Owner notifications (owner only)
GET    /api/notifications/customer Customer's notifications
PUT    /api/notifications/read-all Mark all read
PUT    /api/notifications/:id/read Mark one read
```

### ⚙️ Settings
```
GET    /api/settings               Get shop settings (public)
PUT    /api/settings               Update settings (owner only)
```

### 📸 File Upload
```
POST   /api/upload/image           Upload product image (owner only)
POST   /api/upload/media           Upload image or video for ads (owner only)
```

---

## 💡 Frontend Integration — Replace localStorage with fetch

### Register a customer
```js
const res = await fetch('/api/auth/register', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name, email, password, phone })
});
const { token, user } = await res.json();
localStorage.setItem('fl_token', token); // store JWT
```

### Login
```js
const res = await fetch('/api/auth/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ email, password })
});
const { token, user } = await res.json();
localStorage.setItem('fl_token', token);
```

### Helper — authenticated fetch
```js
const authFetch = (url, options = {}) => {
  const token = localStorage.getItem('fl_token');
  return fetch(url, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...options.headers,
    },
  });
};
```

### Load products
```js
const res = await authFetch('/api/products');
const { products } = await res.json();
renderProducts(products);
```

### Place an order
```js
const res = await authFetch('/api/orders', {
  method: 'POST',
  body: JSON.stringify({ items, address, total, payment: 'cod' })
});
const { order } = await res.json();
```

### Upload product image
```js
const formData = new FormData();
formData.append('image', fileInput.files[0]);

const res = await fetch('/api/upload/image', {
  method: 'POST',
  headers: { Authorization: `Bearer ${token}` },
  body: formData  // Don't set Content-Type — let browser set multipart boundary
});
const { url } = await res.json();
// Use url as product.image
```

### Real-time order tracking (Socket.io)
```js
import { io } from 'socket.io-client';

const socket = io('http://localhost:5000');

// Join the order's room after placing
socket.emit('track_order', orderId);

// Listen for status updates
socket.on('order_status_update', ({ orderId, status }) => {
  updateTrackingUI(status);  // refresh your track steps
});
```

---

## 🔒 Security Notes

- Passwords are hashed with **bcrypt** (12 salt rounds)
- JWT tokens expire in 7 days (configurable via `JWT_EXPIRES_IN`)
- Owner routes are protected with `ownerOnly` middleware (role check)
- File uploads are filtered to allowed types only (jpg/png/webp/mp4/mov)
- Input validation via `express-validator` on all write endpoints
- Use **HTTPS** in production (reverse proxy with Nginx/Caddy)
- Set `NODE_ENV=production` and use strong `JWT_SECRET` (32+ random chars)

---

## 🚀 Production Deployment (Render / Railway / VPS)

1. Set all `.env` values in your hosting dashboard
2. Set `STORAGE_MODE=cloudinary` and fill Cloudinary credentials
3. Set `MONGO_URI` to your Atlas connection string
4. Set `CLIENT_URL` to your frontend's domain
5. Run `node config/seed.js` once to create the owner account
6. Start with `node server.js`

---

## 📦 Tech Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js 18+ |
| Framework | Express.js |
| Database | MongoDB + Mongoose |
| Auth | JWT + bcryptjs |
| File Upload | Multer + Cloudinary (or disk) |
| Real-time | Socket.io |
| Validation | express-validator |
| Dev | nodemon |