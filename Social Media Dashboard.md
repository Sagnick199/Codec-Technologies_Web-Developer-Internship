
# Social Media Dashboard

## Tech Stack
- **Frontend**: React.js
- **Backend**: Node.js with Express.js
- **Database**: PostgreSQL
- **APIs**: Twitter API, Instagram API (or third-party social media APIs)
- **Authentication**: JWT (JSON Web Token) + Passport.js or OAuth for social media logins
- **Data Visualization**: Chart.js or Recharts
- **Scheduling**: Node.js with cron jobs or third-party libraries like **node-cron**

---

## Features

1. **User Authentication and Profiles**:
   - Users can register, log in, and manage their profiles.
   - Integration with social media logins (OAuth) or email/password authentication.

2. **Track Social Media Metrics**:
   - Fetch data from Twitter, Instagram, or any other social media platform via their REST APIs.
   - Track metrics such as followers count, engagement rate, number of posts, etc.

3. **Data Visualization**:
   - Display graphs and charts of metrics using tools like Chart.js or Recharts.

4. **Schedule and Post Content**:
   - Users can schedule posts to Twitter, Instagram, or other platforms.
   - Manage scheduled content from the dashboard.

---

## Project Structure

```plaintext
social-media-dashboard/
├── backend/
│   ├── controllers/
│   │   └── socialMediaController.js
│   ├── models/
│   │   └── User.js
│   ├── routes/
│   │   └── authRoutes.js
│   │   └── socialMediaRoutes.js
│   ├── utils/
│   │   └── api.js
│   ├── config/
│   │   └── db.js
│   ├── server.js
│   └── .env
├── frontend/
│   ├── components/
│   │   ├── Navbar.js
│   │   ├── Dashboard.js
│   │   └── Graph.js
│   ├── pages/
│   │   ├── index.js
│   │   ├── login.js
│   │   └── profile.js
│   ├── services/
│   │   └── apiService.js
│   └── .env.local
├── .gitignore
└── package.json
```

---

## Step-by-Step Implementation

### 1. **Backend Setup (Node.js + PostgreSQL)**

#### A. Install Dependencies

1. **Install Backend Dependencies**:

```bash
mkdir social-media-dashboard-backend
cd social-media-dashboard-backend
npm init -y
npm install express sequelize pg pg-hstore bcryptjs jsonwebtoken passport passport-google-oauth20 axios dotenv cron
```

2. **Setup Sequelize for PostgreSQL**:
   - Set up **Sequelize** ORM for PostgreSQL to interact with the database.
   - Configure your database connection in `config/db.js`.

```js
// config/db.js
const { Sequelize } = require('sequelize');

const sequelize = new Sequelize(process.env.DATABASE_URL, {
  dialect: 'postgres',
  logging: false,
});

module.exports = sequelize;
```

#### B. Create Models

1. **User Model (models/User.js)**:

```js
// models/User.js
const { DataTypes } = require('sequelize');
const sequelize = require('../config/db');

const User = sequelize.define('User', {
  username: { type: DataTypes.STRING, allowNull: false, unique: true },
  email: { type: DataTypes.STRING, allowNull: false, unique: true },
  password: { type: DataTypes.STRING, allowNull: true },
  googleId: { type: DataTypes.STRING, allowNull: true },
});

module.exports = User;
```

#### C. Authentication Routes

1. **User Registration and Login (routes/authRoutes.js)**:

```js
// routes/authRoutes.js
const express = require('express');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const router = express.Router();

// Register User
router.post('/register', async (req, res) => {
  const { username, email, password } = req.body;
  const hashedPassword = await bcrypt.hash(password, 10);

  try {
    const user = await User.create({ username, email, password: hashedPassword });
    res.status(201).json({ message: 'User registered successfully', user });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// Login User
router.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ where: { email } });

  if (!user) {
    return res.status(404).json({ message: 'User not found' });
  }

  const isMatch = await bcrypt.compare(password, user.password);
  if (!isMatch) {
    return res.status(400).json({ message: 'Invalid credentials' });
  }

  const token = jwt.sign({ userId: user.id }, process.env.JWT_SECRET, { expiresIn: '1h' });
  res.json({ token });
});

module.exports = router;
```

#### D. Fetch Social Media Data (Twitter API)

1. **Get Twitter Metrics (controllers/socialMediaController.js)**:

```js
// controllers/socialMediaController.js
const axios = require('axios');

const getTwitterData = async (req, res) => {
  const twitterApiUrl = 'https://api.twitter.com/2/users/by/username/yourUsername';  // Example API endpoint
  const token = req.header('Authorization').split(' ')[1];  // Bearer Token

  try {
    const response = await axios.get(twitterApiUrl, {
      headers: { Authorization: `Bearer ${token}` },
    });
    res.json(response.data);
  } catch (error) {
    res.status(500).json({ error: 'Error fetching Twitter data' });
  }
};

module.exports = { getTwitterData };
```

2. **Set Up Routes (routes/socialMediaRoutes.js)**:

```js
// routes/socialMediaRoutes.js
const express = require('express');
const { getTwitterData } = require('../controllers/socialMediaController');

const router = express.Router();

router.get('/twitter', getTwitterData);

module.exports = router;
```

#### E. Scheduling Posts (Using cron)

You can use `node-cron` to schedule posts to Twitter, Instagram, or other platforms.

```js
// utils/scheduler.js
const cron = require('node-cron');
const axios = require('axios');

// Example: Post content to Twitter every day at 9 AM
cron.schedule('0 9 * * *', async () => {
  try {
    const response = await axios.post('https://api.twitter.com/2/tweets', {
      status: 'Scheduled tweet content here',
    });
    console.log('Scheduled post sent successfully');
  } catch (err) {
    console.log('Error scheduling post:', err);
  }
});
```

---

### 2. **Frontend Setup (React.js)**

#### A. Install Frontend Dependencies

```bash
npx create-react-app social-media-dashboard-frontend
cd social-media-dashboard-frontend
npm install axios react-router-dom chart.js react-chartjs-2
```

#### B. Create Dashboard Component

1. **Dashboard with Graph (components/Dashboard.js)**:

```js
// components/Dashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { Line } from 'react-chartjs-2';
import Chart from 'chart.js/auto';

const Dashboard = () => {
  const [twitterData, setTwitterData] = useState(null);

  useEffect(() => {
    const fetchTwitterData = async () => {
      const response = await axios.get('/api/socialmedia/twitter');
      setTwitterData(response.data);
    };
    fetchTwitterData();
  }, []);

  const data = {
    labels: twitterData?.labels || [],
    datasets: [
      {
        label: 'Twitter Followers',
        data: twitterData?.followers || [],
        borderColor: 'rgba(75, 192, 192, 1)',
        tension: 0.1,
      },
    ],
  };

  return (
    <div>
      <h1>Social Media Dashboard</h1>
      {twitterData ? (
        <Line data={data} />
      ) : (
        <p>Loading Twitter data...</p>
      )}
    </div>
  );
};

export default Dashboard;
```

#### C. User Authentication and Profile Pages

1. **Login Page (pages/login.js)**:

```js
// pages/login.js
import React, { useState } from 'react';
import axios from 'axios';

const Login = ({ setAuthToken }) => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  const handleLogin = async () => {
    try {
      const response = await axios.post('/api/auth/login', { email, password });
      setAuthToken(response.data.token);
    } catch (err) {
      console.log('Login failed', err);
    }
  };

  return (
    <div>
      <h2>Login</h2>
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
      />
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Password"
      />
      <button onClick={handleLogin}>Login</button>
    </div>
  );
};

export default Login;
```

#### D. API Service

1. **API Service for Fetching Data (services/apiService.js)**:

```js
// services/apiService.js
import axios from 'axios';

const fetchTwitterData = async () => {
  const token = localStorage.getItem('token');
  const response = await axios.get('/api/socialmedia/twitter', {
    headers: { Authorization: `Bearer ${token}` },
  });
  return response.data;
};

export default { fetchTwitterData };
```

---

### 3. **Running the Application**

1. **Backend**:

```bash
cd social-media-dashboard-backend
node server.js
```

2. **Frontend**:

```bash
cd social-media-dashboard-frontend
npm start
```

---

## Conclusion

This **Social Media Dashboard** provides:

- **User Authentication**: Register, log in, and manage profiles.
- **Social Media Metrics**: Fetch and display data from Twitter or Instagram.
- **Data Visualization**: Display metrics with charts using Chart.js.
- **Content Scheduling**: Use cron jobs to schedule posts.

You can now download the full project in a `.md` format.

[Download Social Media Dashboard Project Details](sandbox:/mnt/data/social_media_dashboard_project.md)
