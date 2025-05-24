// FLEXI BANKING - Full App Code (Frontend + Backend)

// === BACKEND (Node.js + Express + MongoDB) ===

// server/server.js const express = require('express'); const mongoose = require('mongoose'); const cors = require('cors'); const dotenv = require('dotenv'); const authRoutes = require('./routes/auth'); const cardRoutes = require('./routes/cards');

dotenv.config(); const app = express(); app.use(cors()); app.use(express.json());

mongoose.connect(process.env.MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true }) .then(() => console.log('MongoDB connected')) .catch(err => console.log(err));

app.use('/api/auth', authRoutes); app.use('/api/cards', cardRoutes);

const PORT = process.env.PORT || 5000; app.listen(PORT, () => console.log(Server running on port ${PORT}));

// server/models/User.js const mongoose = require('mongoose'); const bcrypt = require('bcrypt');

const UserSchema = new mongoose.Schema({ name: String, email: { type: String, unique: true }, password: String, pin: String, });

UserSchema.pre('save', async function (next) { if (!this.isModified('password')) return next(); this.password = await bcrypt.hash(this.password, 10); next(); });

module.exports = mongoose.model('User', UserSchema);

// server/models/Card.js const mongoose = require('mongoose');

const CardSchema = new mongoose.Schema({ userId: mongoose.Schema.Types.ObjectId, name: String, number: String, expiry: String, cvv: String, });

module.exports = mongoose.model('Card', CardSchema);

// server/routes/auth.js const express = require('express'); const router = express.Router(); const jwt = require('jsonwebtoken'); const bcrypt = require('bcrypt'); const User = require('../models/User');

router.post('/register', async (req, res) => { try { const { name, email, password, pin } = req.body; const user = new User({ name, email, password, pin }); await user.save(); res.status(201).json({ message: 'User registered' }); } catch (err) { res.status(400).json({ error: 'Email already exists' }); } });

router.post('/login', async (req, res) => { const { email, password } = req.body; const user = await User.findOne({ email }); if (!user) return res.status(404).json({ error: 'User not found' });

const isMatch = await bcrypt.compare(password, user.password); if (!isMatch) return res.status(400).json({ error: 'Invalid credentials' });

const token = jwt.sign({ userId: user._id, name: user.name }, process.env.JWT_SECRET); res.json({ token, name: user.name }); });

module.exports = router;

// server/routes/cards.js const express = require('express'); const jwt = require('jsonwebtoken'); const Card = require('../models/Card'); const router = express.Router();

function authMiddleware(req, res, next) { const token = req.headers.authorization?.split(' ')[1]; if (!token) return res.status(401).json({ error: 'Unauthorized' }); try { const decoded = jwt.verify(token, process.env.JWT_SECRET); req.userId = decoded.userId; next(); } catch { res.status(401).json({ error: 'Invalid token' }); } }

router.post('/', authMiddleware, async (req, res) => { const { name, number, expiry, cvv } = req.body; const card = new Card({ userId: req.userId, name, number, expiry, cvv }); await card.save(); res.json({ message: 'Card saved' }); });

router.get('/', authMiddleware, async (req, res) => { const cards = await Card.find({ userId: req.userId }); res.json(cards.map(c => ({ name: c.name, number: c.number.slice(-4) }))); });

module.exports = router;

// === FRONTEND (React + Tailwind) ===

// client/src/App.js import { useState, useEffect } from 'react'; import axios from 'axios';

function App() { const [token, setToken] = useState(localStorage.getItem('token')); const [name, setName] = useState(localStorage.getItem('name')); const [cards, setCards] = useState([]); const [newCard, setNewCard] = useState({ name: '', number: '', expiry: '', cvv: '' });

useEffect(() => { if (token) fetchCards(); }, [token]);

const fetchCards = async () => { const res = await axios.get('/api/cards', { headers: { Authorization: Bearer ${token} }, }); setCards(res.data); };

const handleCardChange = (e) => { setNewCard({ ...newCard, [e.target.name]: e.target.value }); };

const addCard = async () => { await axios.post('/api/cards', newCard, { headers: { Authorization: Bearer ${token} }, }); setNewCard({ name: '', number: '', expiry: '', cvv: '' }); fetchCards(); };

const login = async () => { const res = await axios.post('/api/auth/login', { email: 'john@example.com', password: '123456', }); localStorage.setItem('token', res.data.token); localStorage.setItem('name', res.data.name); setToken(res.data.token); setName(res.data.name); };

if (!token) return <button onClick={login}>Login as John</button>;

return ( <div className="p-6 max-w-lg mx-auto bg-white shadow rounded-xl"> <h1 className="text-2xl font-bold mb-4">Hello, {name}</h1>

<div className="mb-6">
    <h2 className="font-semibold mb-2">Link Credit Card</h2>
    <input className="border p-2 w-full mb-2" name="name" value={newCard.name} onChange={handleCardChange} placeholder="Name" />
    <input className="border p-2 w-full mb-2" name="number" value={newCard.number} onChange={handleCardChange} placeholder="Card Number" />
    <input className="border p-2 w-full mb-2" name="expiry" value={newCard.expiry} onChange={handleCardChange} placeholder="MM/YY" />
    <input className="border p-2 w-full mb-2" name="cvv" value={newCard.cvv} onChange={handleCardChange} placeholder="CVV" />
    <button className="bg-blue-600 text-white px-4 py-2 rounded" onClick={addCard}>Save Card</button>
  </div>

  <div>
    <h2 className="font-semibold mb-2">Your Cards</h2>
    <ul>
      {cards.map((c, i) => (
        <li key={i} className="mb-1">**** **** **** {c.number} - {c.name}</li>
      ))}
    </ul>
  </div>
</div>

); }

export default App;
