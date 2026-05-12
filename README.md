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
