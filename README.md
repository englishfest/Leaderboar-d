# Leaderboar-d
game-leaderboard/
├── backend/
│   ├── server.js
│   ├── db.js
│   ├── routes/
│   │   └── leaderboard.js
│   ├── package.json
│   └── .env
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── Leaderboard.jsx
│   │   │   ├── PlayerProfile.jsx
│   │   │   └── Filters.jsx
│   │   ├── App.jsx
│   │   └── index.css
│   ├── package.json
│   └── vite.config.js
└── README.md
{
  "name": "game-leaderboard-backend",
  "version": "1.0.0",
  "description": "Game Leaderboard API",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.11.0",
    "dotenv": "^16.0.3",
    "cors": "^2.8.5",
    "body-parser": "^1.20.2"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
const { Pool } = require('pg');
require('dotenv').config();

const pool = new Pool({
  user: process.env.DB_USER || 'postgres',
  password: process.env.DB_PASSWORD || 'password',
  host: process.env.DB_HOST || 'localhost',
  port: process.env.DB_PORT || 5432,
  database: process.env.DB_NAME || 'game_leaderboard'
});

// Tabloları oluştur
const initDB = async () => {
  try {
    await pool.query(`
      CREATE TABLE IF NOT EXISTS players (
        id SERIAL PRIMARY KEY,
        username VARCHAR(255) UNIQUE NOT NULL,
        avatar_url VARCHAR(255),
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      );

      CREATE TABLE IF NOT EXISTS scores (
        id SERIAL PRIMARY KEY,
        player_id INT NOT NULL REFERENCES players(id) ON DELETE CASCADE,
        score INT NOT NULL,
        game_mode VARCHAR(50),
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      );

      CREATE INDEX IF NOT EXISTS idx_scores_player_id ON scores(player_id);
      CREATE INDEX IF NOT EXISTS idx_scores_created_at ON scores(created_at);
    `);
    console.log('✅ Veritabanı hazır!');
  } catch (err) {
    console.error('❌ DB Hatası:', err);
  }
};

initDB();

module.exports = pool;
const express = require('express');
const cors = require('cors');
const bodyParser = require('body-parser');
require('dotenv').config();
const leaderboardRoutes = require('./routes/leaderboard');

const app = express();
const PORT = process.env.PORT || 5000;

// Middleware
app.use(cors());
app.use(bodyParser.json());

// Routes
app.use('/api/leaderboard', leaderboardRoutes);

// Health check
app.get('/api/health', (req, res) => {
  res.json({ status: 'OK' });
});

app.listen(PORT, () => {
  console.log(`🎮 Server ${PORT} portunda çalışıyor...`);
});
const express = require('express');
const pool = require('../db');
const router = express.Router();

// Oyuncu oluştur veya var olanı getir
router.post('/player', async (req, res) => {
  try {
    const { username, avatar_url } = req.body;
    const result = await pool.query(
      'INSERT INTO players (username, avatar_url) VALUES ($1, $2) ON CONFLICT (username) DO UPDATE SET avatar_url = $2 RETURNING *',
      [username, avatar_url || 'https://via.placeholder.com/50']
    );
    res.json(result.rows[0]);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Puan ekle
router.post('/score', async (req, res) => {
  try {
    const { player_id, score, game_mode } = req.body;
    const result = await pool.query(
      'INSERT INTO scores (player_id, score, game_mode) VALUES ($1, $2, $3) RETURNING *',
      [player_id, score, game_mode || 'classic']
    );
    res.json(result.rows[0]);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Tüm zamanlar en yüksek puanlar (Top 100)
router.get('/all-time', async (req, res) => {
  try {
    const result = await pool.query(`
      SELECT 
        p.id, p.username, p.avatar_url,
        MAX(s.score) as best_score,
        COUNT(s.id) as total_games,
        AVG(s.score)::INT as average_score
      FROM players p
      LEFT JOIN scores s ON p.id = s.player_id
      GROUP BY p.id, p.username, p.avatar_url
      ORDER BY best_score DESC
      LIMIT 100
    `);
    res.json(result.rows);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Haftalık en yüksek puanlar
router.get('/weekly', async (req, res) => {
  try {
    const result = await pool.query(`
      SELECT 
        p.id, p.username, p.avatar_url,
        MAX(s.score) as best_score,
        COUNT(s.id) as total_games,
        AVG(s.score)::INT as average_score
      FROM players p
      LEFT JOIN scores s ON p.id = s.player_id
      WHERE s.created_at >= NOW() - INTERVAL '7 days'
      GROUP BY p.id, p.username, p.avatar_url
      ORDER BY best_score DESC
      LIMIT 100
    `);
    res.json(result.rows);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Günlük en yüksek puanlar
router.get('/daily', async (req, res) => {
  try {
    const result = await pool.query(`
      SELECT 
        p.id, p.username, p.avatar_url,
        MAX(s.score) as best_score,
        COUNT(s.id) as total_games,
        AVG(s.score)::INT as average_score
      FROM players p
      LEFT JOIN scores s ON p.id = s.player_id
      WHERE s.created_at >= NOW() - INTERVAL '1 day'
      GROUP BY p.id, p.username, p.avatar_url
      ORDER BY best_score DESC
      LIMIT 100
    `);
    res.json(result.rows);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Oyuncu profili
router.get('/player/:id', async (req, res) => {
  try {
    const { id } = req.params;
    const result = await pool.query(`
      SELECT 
        p.id, p.username, p.avatar_url, p.created_at,
        COUNT(s.id) as total_games,
        MAX(s.score) as best_score,
        AVG(s.score)::INT as average_score,
        MIN(s.score) as worst_score
      FROM players p
      LEFT JOIN scores s ON p.id = s.player_id
      WHERE p.id = $1
      GROUP BY p.id, p.username, p.avatar_url, p.created_at
    `, [id]);
    res.json(result.rows[0]);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Oyuncu puanları (detaylı)
router.get('/player/:id/scores', async (req, res) => {
  try {
    const { id } = req.params;
    const result = await pool.query(`
      SELECT * FROM scores WHERE player_id = $1 ORDER BY created_at DESC
    `, [id]);
    res.json(result.rows);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

module.exports = router;
DB_USER=postgres
DB_PASSWORD=password
DB_HOST=localhost
DB_PORT=5432
DB_NAME=game_leaderboard
PORT=5000
{
  "name": "game-leaderboard-frontend",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "axios": "^1.6.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.0.0",
    "vite": "^4.4.0"
  }
}
import { useState, useEffect } from 'react';
import Leaderboard from './components/Leaderboard';
import PlayerProfile from './components/PlayerProfile';
import Filters from './components/Filters';
import axios from 'axios';
import './App.css';

const API_URL = 'http://localhost:5000/api/leaderboard';

function App() {
  const [timeFilter, setTimeFilter] = useState('all-time');
  const [leaderboardData, setLeaderboardData] = useState([]);
  const [selectedPlayer, setSelectedPlayer] = useState(null);
  const [loading, setLoading] = useState(false);
  const [newScore, setNewScore] = useState({ username: '', score: '' });

  // Leaderboard verilerini çek
  const fetchLeaderboard = async (filter) => {
    setLoading(true);
    try {
      const endpoint = filter === 'all-time' ? '/all-time' : `/${filter}`;
      const response = await axios.get(`${API_URL}${endpoint}`);
      setLeaderboardData(response.data);
    } catch (err) {
      console.error('Leaderboard yüklenirken hata:', err);
    }
    setLoading(false);
  };

  // Puan ekle
  const addScore = async (e) => {
    e.preventDefault();
    if (!newScore.username || !newScore.score) return;

    try {
      // Oyuncuyu oluştur/getir
      const playerRes = await axios.post(`${API_URL}/player`, {
        username: newScore.username,
        avatar_url: `https://api.dicebear.com/7.x/avataaars/svg?seed=${newScore.username}`
      });

      // Puanı ekle
      await axios.post(`${API_URL}/score`, {
        player_id: playerRes.data.id,
        score: parseInt(newScore.score),
        game_mode: 'classic'
      });

      // Leaderboard'u yenile
      fetchLeaderboard(timeFilter);
      setNewScore({ username: '', score: '' });
    } catch (err) {
      console.error('Puan eklenirken hata:', err);
    }
  };

  useEffect(() => {
    fetchLeaderboard(timeFilter);
  }, [timeFilter]);

  return (
    <div className="app">
      <header className="header">
        <h1>🎮 Oyun Leaderboard'u</h1>
        <p>En yüksek puanları ve oyuncu sıralamalarını görüntüle</p>
      </header>

      <div className="container">
        {/* Puan Ekleme Formu */}
        <div className="add-score-section">
          <h2>Puan Ekle</h2>
          <form onSubmit={addScore}>
            <input
              type="text"
              placeholder="Oyuncu adı"
              value={newScore.username}
              onChange={(e) => setNewScore({ ...newScore, username: e.target.value })}
            />
            <input
              type="number"
              placeholder="Puan"
              value={newScore.score}
              onChange={(e) => setNewScore({ ...newScore, score: e.target.value })}
            />
            <button type="submit">Ekle</button>
          </form>
        </div>

        {/* Filtreler */}
        <Filters timeFilter={timeFilter} setTimeFilter={setTimeFilter} />

        {/* Leaderboard */}
        {selectedPlayer ? (
          <PlayerProfile 
            playerId={selectedPlayer} 
            onBack={() => setSelectedPlayer(null)}
          />
        ) : (
          <Leaderboard 
            data={leaderboardData} 
            loading={loading}
            onPlayerClick={setSelectedPlayer}
          />
        )}
      </div>
    </div>
  );
}

export default App;
import './Leaderboard.css';

function Leaderboard({ data, loading, onPlayerClick }) {
  if (loading) return <div className="loading">Yükleniyor...</div>;

  return (
    <div className="leaderboard">
      <table>
        <thead>
          <tr>
            <th>#</th>
            <th>Avatar</th>
            <th>Oyuncu Adı</th>
            <th>En Yüksek Puan</th>
            <th>Toplam Oyun</th>
            <th>Ort. Puan</th>
          </tr>
        </thead>
        <tbody>
          {data.map((player, index) => (
            <tr key={player.id} onClick={() => onPlayerClick(player.id)} className="clickable">
              <td className="rank">{index + 1}</td>
              <td>
                <img 
                  src={player.avatar_url} 
                  alt={player.username} 
                  className="avatar"
                />
              </td>
              <td className="username">{player.username}</td>
              <td className="score">{player.best_score || 0}</td>
              <td>{player.total_games || 0}</td>
              <td>{player.average_score || 0}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}

export default Leaderboard;
import './Filters.css';

function Filters({ timeFilter, setTimeFilter }) {
  return (
    <div className="filters">
      <button 
        className={timeFilter === 'daily' ? 'active' : ''} 
        onClick={() => setTimeFilter('daily')}
      >
        📅 Günlük
      </button>
      <button 
        className={timeFilter === 'weekly' ? 'active' : ''} 
        onClick={() => setTimeFilter('weekly')}
      >
        📊 Haftalık
      </button>
      <button 
        className={timeFilter === 'all-time' ? 'active' : ''} 
        onClick={() => setTimeFilter('all-time')}
      >
        🏆 Tüm Zamanlar
      </button>
    </div>
  );
}

export default Filters;
