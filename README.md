portfolio-manager/
├── app.js
├── package.json
├── .env
├── .gitignore
├── config/
│   └── db.js
├── controllers/
│   └── portfolioController.js
├── models/
│   └── portfolioModel.js
├── routes/
│   └── portfolioRoutes.js
├── utils/
│   └── response.js
├── public/
│   ├── index.html
│   ├── style.css
│   └── script.js

// app.js
const express = require('express');
const app = express();
const path = require('path');
require('dotenv').config();
const portfolioRoutes = require('./routes/portfolioRoutes');

app.use(express.json());
app.use(express.static(path.join(__dirname, 'public')));
app.use('/api/portfolio', portfolioRoutes);

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));

// .env
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=yourpassword
DB_NAME=portfolio_db
PORT=3000

// config/db.js
const mysql = require('mysql2/promise');
require('dotenv').config();

const pool = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  waitForConnections: true,
  connectionLimit: 10,
  queueLimit: 0,
});

module.exports = pool;

// models/portfolioModel.js
const db = require('../config/db');

async function getAllItems({ type, sort, search }) {
  let query = `SELECT * FROM portfolio WHERE 1=1`;
  const params = [];

  if (type) {
    query += ' AND type = ?';
    params.push(type);
  }
  if (search) {
    query += ' AND name LIKE ?';
    params.push(`%${search}%`);
  }
  if (sort) {
    query += ` ORDER BY ${sort}`;
  }

  const [rows] = await db.query(query, params);
  return rows;
}

async function addItem(item) {
  const { name, type, value, category } = item;
  const [result] = await db.query(
    'INSERT INTO portfolio (name, type, value, category) VALUES (?, ?, ?, ?)',
    [name, type, value, category]
  );
  return result.insertId;
}

async function removeItem(id) {
  await db.query('DELETE FROM portfolio WHERE id = ?', [id]);
}

async function getHistoricalPerformance() {
  const [rows] = await db.query(`
    SELECT DATE(created_at) as date, SUM(value) as total_value
    FROM portfolio
    GROUP BY DATE(created_at)
    ORDER BY date
  `);
  return rows;
}

module.exports = { getAllItems, addItem, removeItem, getHistoricalPerformance };

// controllers/portfolioController.js
const model = require('../models/portfolioModel');
const response = require('../utils/response');

exports.getPortfolio = async (req, res) => {
  try {
    const data = await model.getAllItems(req.query);
    response.success(res, data);
  } catch (err) {
    response.error(res, err.message);
  }
};

exports.addItem = async (req, res) => {
  try {
    const id = await model.addItem(req.body);
    response.success(res, { id });
  } catch (err) {
    response.error(res, err.message);
  }
};

exports.removeItem = async (req, res) => {
  try {
    await model.removeItem(req.params.id);
    response.success(res, { message: 'Item removed' });
  } catch (err) {
    response.error(res, err.message);
  }
};

exports.getPerformance = async (req, res) => {
  try {
    const data = await model.getHistoricalPerformance();
    response.success(res, data);
  } catch (err) {
    response.error(res, err.message);
  }
};

// routes/portfolioRoutes.js
const express = require('express');
const router = express.Router();
const controller = require('../controllers/portfolioController');

router.get('/', controller.getPortfolio);
router.post('/', controller.addItem);
router.delete('/:id', controller.removeItem);
router.get('/performance', controller.getPerformance);

module.exports = router;

// utils/response.js
exports.success = (res, data) => {
  res.status(200).json({ status: 'success', data });
};

exports.error = (res, message) => {
  res.status(500).json({ status: 'error', message });
};

// public/index.html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Portfolio Performance</title>
  <link rel="stylesheet" href="style.css" />
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body>
  <h1>Net Worth Performance</h1>
  <canvas id="netWorthChart"></canvas>
  <script src="script.js"></script>
</body>
</html>

// public/style.css
body {
  font-family: sans-serif;
  padding: 2rem;
  background-color: #f4f4f4;
}

canvas {
  background: white;
  border: 1px solid #ccc;
  padding: 1rem;
  max-width: 100%;
}

// public/script.js
fetch('/api/portfolio/performance')
  .then(res => res.json())
  .then(res => {
    const labels = res.data.map(row => row.date);
    const data = res.data.map(row => row.total_value);

    const ctx = document.getElementById('netWorthChart').getContext('2d');
    new Chart(ctx, {
      type: 'line',
      data: {
        labels,
        datasets: [{
          label: 'Net Worth (USD)',
          data,
          fill: true,
          borderColor: 'rgb(75, 192, 192)',
          tension: 0.2,
        }]
      },
      options: {
        responsive: true,
        plugins: {
          legend: { position: 'top' },
          title: { display: true, text: '30-Day Portfolio Value' }
        }
      }
    });
  });




  //mysql workbench
  CREATE DATABASE IF NOT EXISTS portfolio_db;
USE portfolio_db;

CREATE TABLE portfolio (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    type ENUM('stock', 'bond', 'cash') NOT NULL,
    value DECIMAL(15,2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

