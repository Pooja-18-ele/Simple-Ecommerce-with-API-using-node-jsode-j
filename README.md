# Simple-Ecommerce-with-API-using-node-jsode-j
const express = require('express');
const mongoose = require('mongoose');
const dotenv = require('dotenv');
const authRoutes = require('./routes/authRoutes');
const productRoutes = require('./routes/productRoutes');
const cartRoutes = require('./routes/cartRoutes');
const orderRoutes = require('./routes/orderRoutes');

dotenv.config();
const app = express();
app.use(express.json());

mongoose.connect(process.env.MONGO_URI, () => {
  console.log('MongoDB Connected');
});

app.use('/api/auth', authRoutes);
app.use('/api/products', productRoutes);
app.use('/api/cart', cartRoutes);
app.use('/api/orders', orderRoutes);

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(Server running on port ${PORT}));






models/User.js

const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: String,
  email: String,
  password: String,
  role: { type: String, enum: ['customer', 'admin'], default: 'customer' }
});

module.exports = mongoose.model('User', userSchema);

models/Product.js

const mongoose = require('mongoose');

const productSchema = new mongoose.Schema({
  name: String,
  category: String,
  price: Number,
  description: String
});

module.exports = mongoose.model('Product', productSchema);

models/Cart.js

const mongoose = require('mongoose');

const cartSchema = new mongoose.Schema({
  userId: String,
  items: [
  {
     productId: String,
      quantity: Number
    }
  ]
});

module.exports = mongoose.model('Cart', cartSchema);

models/Order.js

const mongoose = require('mongoose');

const orderSchema = new mongoose.Schema({
  userId: String,
  items: [
    {
      productId: String,
      quantity: Number
    }
  ],
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Order', orderSchema);




middlewares/authMiddleware.js

const jwt = require('jsonwebtoken');

module.exports = (req, res, next) => {
  const token = req.header('Authorization');
  if (!token) return res.status(401).json({ msg: 'No token, auth denied' });

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch {
    res.status(400).json({ msg: 'Token is not valid' });
  }
};

middlewares/roleMiddleware.js

module.exports = (role) => (req, res, next) => {
  if (req.user.role !== role) {
    return res.status(403).json({ msg: 'Access denied' });
  }
  next();
};

controllers/authController.js

const User = require('../models/User');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

exports.register = async (req, res) => {
  const { username, email, password, role } = req.body;
  const hash = await bcrypt.hash(password, 10);
  const user = new User({ username, email, password: hash, role });
  await user.save();
  res.json({ msg: 'Registered successfully' });
};

exports.login = async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });
  if (!user) return res.status(400).json({ msg: 'User not found' });

  const isMatch = await bcrypt.compare(password, user.password);
  if (!isMatch) return res.status(400).json({ msg: 'Invalid credentials' });

  const token = jwt.sign({ id: user._id, role: user.role }, process.env.JWT_SECRET);
  res.json({ token });
};

routes/authRoutes.js

const router = require('express').Router();
const { register, login } = require('../controllers/authController');

router.post('/register', register);
router.post('/login', login);

module.exports = router;


controllers/productController.js

const Product = require('../models/Product');

exports.getProducts = async (req, res) => {
  const { page = 1, limit = 5, search = '' } = req.query;
  const query = { name: { $regex: search, $options: 'i' } };
  const products = await Product.find(query)
    .skip((page - 1) * limit)
    .limit(parseInt(limit));
  res.json(products);
};

exports.createProduct = async (req, res) => {
  const product = new Product(req.body);
  await product.save();
  res.json(product);
};

exports.updateProduct = async (req, res) => {
  const product = await Product.findByIdAndUpdate(req.params.id, req.body, { new: true });
  res.json(product);
};

exports.deleteProduct = async (req, res) => {
  await Product.findByIdAndDelete(req.params.id);
  res.json({ msg: 'Product deleted' });
};

routes/productRoutes.js

const router = require('express').Router();
const auth = require('../middlewares/authMiddleware');
const role = require('../middlewares/roleMiddleware');
const ctrl = require('../controllers/productController');

router.get('/', ctrl.getProducts);
router.post('/', auth, role('admin'), ctrl.createProduct);
router.put('/:id', auth, role('admin'), ctrl.updateProduct);
router.delete('/:id', auth, role('admin'), ctrl.deleteProduct);

module.exports = router;




controllers/cartController.js

const Cart = require('../models/Cart');

exports.getCart = async (req, res) => {
  const cart = await Cart.findOne({ userId: req.user.id });
  res.json(cart || { items: [] });
};

exports.addToCart = async (req, res) => {
  let cart = await Cart.findOne({ userId: req.user.id });
  if (!cart) cart = new Cart({ userId: req.user.id, items: [] });

  const itemIndex = cart.items.findIndex(i => i.productId === req.body.productId);
  if (itemIndex > -1) {
    cart.items[itemIndex].quantity += req.body.quantity;
  } else {
    cart.items.push(req.body);
  }
  await cart.save();
  res.json(cart);
};

exports.removeFromCart = async (req, res) => {
  const cart = await Cart.findOne({ userId: req.user.id });
  cart.items = cart.items.filter(i => i.productId !== req.params.productId);
  await cart.save();
  res.json(cart);
};

routes/cartRoutes.js

const router = require('express').Router();
const auth = require('../middlewares/authMiddleware');
const ctrl = require('../controllers/cartController');

router.get('/', auth, ctrl.getCart);
router.post('/', auth, ctrl.addToCart);
router.delete('/:productId', auth, ctrl.removeFromCart);

module.exports = router;




controllers/orderController.js

const Order = require('../models/Order');
const Cart = require('../models/Cart');

exports.createOrder = async (req, res) => {
  const cart = await Cart.findOne({ userId: req.user.id });
  if (!cart || cart.items.length === 0)
    return res.status(400).json({ msg: 'Cart is empty' });

  const order = new Order({ userId: req.user.id, items: cart.items });
  await order.save();
  cart.items = [];
  await cart.save();
  res.json(order);
};

exports.getOrders = async (req, res) => {
  const orders = await Order.find({ userId: req.user.id });
  res.json(orders);
};

routes/orderRoutes.js

const router = require('express').Router();
const auth = require('../middlewares/authMiddleware');
const ctrl = require('../controllers/orderController');

router.post('/', auth, ctrl.createOrder);
router.get('/', auth, ctrl.getOrders);

module.exports = router;



PORT=5000
MONGO_URI=mongodb://localhost:27017/ecommerce
JWT_SECRET=your_jwt_secret



