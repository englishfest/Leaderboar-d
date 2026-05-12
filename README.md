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
import { useState, useEffect } from 'react';
import axios from 'axios';
import './PlayerProfile.css';

const API_URL = 'http://localhost:5000/api/leaderboard';

function PlayerProfile({ playerId, onBack }) {
  const [player, setPlayer] = useState(null);
  const [scores, setScores] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const fetchData = async () => {
      try {
        const [playerRes, scoresRes] = await Promise.all([
          axios.get(`${API_URL}/player/${playerId}`),
          axios.get(`${API_URL}/player/${playerId}/scores`)
        ]);
        setPlayer(playerRes.data);
        setScores(scoresRes.data);
      } catch (err) {
        console.error('Profil yüklenirken hata:', err);
      }
      setLoading(false);
    };

    fetchData();
  }, [playerId]);

  if (loading) return <div className="loading">Yükleniyor...</div>;
  if (!player) return <div className="error">Oyuncu bulunamadı</div>;

  return (
    <div className="player-profile">
      <button onClick={onBack} className="back-btn">← Geri</button>
      
      <div className="profile-header">
        <img src={player.avatar_url} alt={player.username} className="large-avatar" />
        <div className="profile-info">
          <h2>{player.username}</h2>
          <p>Katılım: {new Date(player.created_at).toLocaleDateString('tr-TR')}</p>
        </div>
      </div>

      <div className="stats-grid">
        <div className="stat">
          <label>En Yüksek Puan</label>
          <span>{player.best_score || 0}</span>
        </div>
        <div className="stat">
          <label>En Düşük Puan</label>
          <span>{player.worst_score || 0}</span>
        </div>
        <div className="stat">
          <label>Ort. Puan</label>
          <span>{player.average_score || 0}</span>
        </div>
        <div className="stat">
          <label>Toplam Oyun</label>
          <span>{player.total_games || 0}</span>
        </div>
      </div>

      <div className="scores-history">
        <h3>Puan Geçmişi</h3>
        {scores.length > 0 ? (
          <table>
            <thead>
              <tr>
                <th>Puan</th>
                <th>Mod</th>
                <th>Tarih</th>
              </tr>
            </thead>
            <tbody>
              {scores.map((score) => (
                <tr key={score.id}>
                  <td className="score">{score.score}</td>
                  <td>{score.game_mode}</td>
                  <td>{new Date(score.created_at).toLocaleString('tr-TR')}</td>
                </tr>
              ))}
            </tbody>
          </table>
        ) : (
          <p>Henüz puan kaydı yok</p>
        )}
      </div>
    </div>
  );
}

export default PlayerProfile;
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  min-height: 100vh;
  color: #333;
}

.app {
  min-height: 100vh;
  padding: 20px;
}

.header {
  text-align: center;
  color: white;
  margin-bottom: 40px;
  animation: fadeIn 0.6s ease-in;
}

.header h1 {
  font-size: 3rem;
  margin-bottom: 10px;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.3);
}

.header p {
  font-size: 1.1rem;
  opacity: 0.9;
}

.container {
  max-width: 1200px;
  margin: 0 auto;
}

.add-score-section {
  background: white;
  padding: 30px;
  border-radius: 15px;
  margin-bottom: 30px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
}

.add-score-section h2 {
  margin-bottom: 20px;
  color: #333;
}

.add-score-section form {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.add-score-section input {
  flex: 1;
  min-width: 150px;
  padding: 12px;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  font-size: 1rem;
  transition: all 0.3s;
}

.add-score-section input:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 10px rgba(102, 126, 234, 0.3);
}

.add-score-section button {
  padding: 12px 30px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1rem;
  font-weight: bold;
  transition: all 0.3s;
}

.add-score-section button:hover {
  transform: translateY(-2px);
  box-shadow: 0 5px 15px rgba(102, 126, 234, 0.4);
}

.loading {
  text-align: center;
  padding: 40px;
  color: white;
  font-size: 1.2rem;
}

.error {
  text-align: center;
  padding: 40px;
  color: white;
  font-size: 1.2rem;
  background: rgba(255, 0, 0, 0.3);
  border-radius: 10px;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(-20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@media (max-width: 600px) {
  .header h1 {
    font-size: 2rem;
  }

  .add-score-section form {
    flex-direction: column;
  }
}
.leaderboard {
  background: white;
  border-radius: 15px;
  overflow: hidden;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
  animation: slideUp 0.6s ease-out;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

th {
  padding: 20px;
  text-align: left;
  font-weight: 600;
  text-transform: uppercase;
  font-size: 0.9rem;
  letter-spacing: 0.5px;
}

td {
  padding: 15px 20px;
  border-bottom: 1px solid #f0f0f0;
}

tbody tr {
  transition: all 0.3s ease;
}

tbody tr:hover {
  background: #f8f9ff;
}

tbody tr.clickable {
  cursor: pointer;
}

tbody tr.clickable:hover {
  background: #f0f4ff;
}

.rank {
  font-weight: bold;
  color: #667eea;
  font-size: 1.1rem;
}

.avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  object-fit: cover;
  border: 2px solid #667eea;
}

.username {
  font-weight: 600;
  color: #333;
}

.score {
  color: #764ba2;
  font-weight: 600;
  font-size: 1.1rem;
}

@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(30px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@media (max-width: 768px) {
  table {
    font-size: 0.9rem;
  }

  th, td {
    padding: 10px;
  }

  .avatar {
    width: 35px;
    height: 35px;
  }
}
.filters {
  display: flex;
  justify-content: center;
  gap: 15px;
  margin-bottom: 30px;
  flex-wrap: wrap;
}

.filters button {
  padding: 12px 25px;
  background: white;
  color: #667eea;
  border: 2px solid #667eea;
  border-radius: 25px;
  cursor: pointer;
  font-size: 1rem;
  font-weight: 600;
  transition: all 0.3s ease;
}

.filters button:hover {
  background: #f0f4ff;
  transform: translateY(-2px);
  box-shadow: 0 5px 15px rgba(102, 126, 234, 0.3);
}

.filters button.active {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border-color: transparent;
  box-shadow: 0 5px 20px rgba(102, 126, 234, 0.4);
}

@media (max-width: 600px) {
  .filters {
    gap: 10px;
  }

  .filters button {
    padding: 10px 15px;
    font-size: 0.9rem;
  }
}
