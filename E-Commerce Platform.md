
# E-Commerce Platform

## Tech Stack
- **Frontend**: React.js / Next.js
- **Backend**: Node.js (Express)
- **Database**: MongoDB (or MySQL)
- **Payment Integration**: Stripe API
- **Authentication**: JWT (JSON Web Token)

---

## Features

- **Product Search & Filters**: Users can search products and filter by categories or price range.
- **Shopping Cart & Checkout**: Users can add products to the cart, view, and proceed to checkout.
- **Admin Panel**: Admins can manage products, view orders, and update product details.
- **User Authentication**: Users can register, login, and access their profile.

---

## Project Structure

```plaintext
ecommerce-backend/
├── models/
│   ├── Product.js
│   ├── User.js
│   └── Order.js
├── routes/
│   ├── productRoutes.js
│   ├── authRoutes.js
│   └── adminRoutes.js
├── controllers/
│   └── productController.js
├── middleware/
│   └── authMiddleware.js
├── .env
├── server.js
├── package.json
ecommerce-frontend/
├── components/
│   ├── ProductCard.js
│   └── Cart.js
├── pages/
│   ├── index.js
│   ├── success.js
│   └── cancel.js
├── redux/
│   └── cartReducer.js
├── utils/
│   └── api.js
└── .env.local
```

---

## Backend Setup (Node.js + MongoDB)

### Install Dependencies

```bash
mkdir ecommerce-backend
cd ecommerce-backend
npm init -y
npm install express mongoose bcryptjs jsonwebtoken cors stripe dotenv
```

### Basic Express Server (`server.js`)

```js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();
app.use(cors());
app.use(express.json()); // For parsing application/json

// Connect to MongoDB
mongoose.connect(process.env.MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(() => console.log('Connected to MongoDB'))
  .catch(err => console.log(err));

// Routes
const productRoutes = require('./routes/productRoutes');
app.use('/api/products', productRoutes);

// Server listening on port
const PORT = process.env.PORT || 5000;
app.listen(PORT, () => {
  console.log(`Server is running on port ${PORT}`);
});
```

### Product Model (`models/Product.js`)

```js
const mongoose = require('mongoose');
const Schema = mongoose.Schema;

const productSchema = new Schema({
  name: { type: String, required: true },
  description: { type: String, required: true },
  price: { type: Number, required: true },
  category: { type: String },
  stock: { type: Number, required: true },
  imageUrl: { type: String },
});

module.exports = mongoose.model('Product', productSchema);
```

### User Model (`models/User.js`)

```js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  isAdmin: { type: Boolean, default: false },
});

// Password hashing
userSchema.pre('save', async function (next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

userSchema.methods.comparePassword = async function (password) {
  return await bcrypt.compare(password, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Authentication Routes (`routes/authRoutes.js`)

```js
const express = require('express');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const router = express.Router();

// Register
router.post('/register', async (req, res) => {
  const { name, email, password } = req.body;

  try {
    const user = await User.findOne({ email });
    if (user) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const hashedPassword = await bcrypt.hash(password, 10);
    const newUser = new User({ name, email, password: hashedPassword });

    await newUser.save();
    res.status(201).json({ message: 'User created successfully' });
  } catch (err) {
    res.status(500).json({ message: 'Error registering user', error: err });
  }
});

// Login
router.post('/login', async (req, res) => {
  const { email, password } = req.body;

  try {
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }

    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }

    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, { expiresIn: '1h' });
    res.json({ token });
  } catch (err) {
    res.status(500).json({ message: 'Error logging in', error: err });
  }
});

module.exports = router;
```

---

## Frontend Setup (React.js / Next.js)

### Install Dependencies

```bash
npx create-next-app ecommerce-frontend
cd ecommerce-frontend
npm install axios react-redux @stripe/stripe-js @stripe/react-stripe-js
```

### Displaying Products (`pages/index.js`)

```js
import { useEffect, useState } from 'react';
import axios from 'axios';

const HomePage = () => {
  const [products, setProducts] = useState([]);

  useEffect(() => {
    const fetchProducts = async () => {
      const { data } = await axios.get('http://localhost:5000/api/products');
      setProducts(data);
    };
    fetchProducts();
  }, []);

  return (
    <div>
      <h1>Products</h1>
      <div>
        {products.map(product => (
          <div key={product._id}>
            <h3>{product.name}</h3>
            <p>{product.description}</p>
            <p>${product.price}</p>
            <button>Add to Cart</button>
          </div>
        ))}
      </div>
    </div>
  );
};

export default HomePage;
```

### Checkout Button (`components/CheckoutButton.js`)

```js
import { useStripe, useElements } from '@stripe/react-stripe-js';
import axios from 'axios';

const CheckoutButton = ({ cartItems }) => {
  const stripe = useStripe();
  const elements = useElements();

  const handleCheckout = async () => {
    const response = await axios.post('http://localhost:5000/api/create-checkout-session', {
      cartItems
    });

    const { id } = response.data;
    const { error } = await stripe.redirectToCheckout({ sessionId: id });

    if (error) console.log("Error during checkout", error);
  };

  return (
    <button onClick={handleCheckout} disabled={!stripe}>Checkout</button>
  );
};

export default CheckoutButton;
```

### Stripe Checkout Integration

```js
import { loadStripe } from '@stripe/stripe-js';
import { Elements } from '@stripe/react-stripe-js';

const stripePromise = loadStripe(process.env.STRIPE_PUBLISHABLE_KEY);

const App = () => (
  <Elements stripe={stripePromise}>
    <CheckoutButton cartItems={cartItems} />
  </Elements>
);
```

---

## Payment Integration (Stripe)

1. **Backend**: Create a checkout session using Stripe API.
2. **Frontend**: Use Stripe's React library to process payments.
3. **Success & Cancel Pages**: Display messages after payment completion.

---

## Admin Panel

### Admin Routes

Protect the routes using JWT and ensure only admins can access the product management features.

```js
const verifyAdmin = (req, res, next) => {
  const token = req.header('Authorization')?.split(' ')[1];

  if (!token) return res.status(401).json({ message: 'Access denied' });

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    if (!decoded.isAdmin) {
      return res.status(403).json({ message: 'Access forbidden' });
    }
    next();
  } catch (err) {
    return res.status(400).json({ message: 'Invalid token' });
  }
};
```

---

## Conclusion

This e-commerce project uses a **Node.js** backend with **MongoDB** and integrates **Stripe** for payments. The frontend, built with **React** or **Next.js**, handles product listings, a shopping cart, and the checkout process.

---

### Downloadable Project

You can now download the full project in a `.md` format using the following link:

[Download E-Commerce Project Details](sandbox:/mnt/data/ecommerce_project_details.md)
