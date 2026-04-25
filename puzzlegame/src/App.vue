


// =========================
// CLIENT (Vue 3 - Vite)
// =========================

// main.js
import { createApp } from 'vue'
import App from './App.vue'

createApp(App).mount('#app')


// App.vue
<template>
  <div class="app">
    <h1>Puzzle Game</h1>

    <div>
      <label>Difficulty:</label>
      <select v-model="difficulty" @change="startGame">
        <option value="3">Easy</option>
        <option value="4">Medium</option>
        <option value="5">Hard</option>
      </select>
    </div>

    <div class="stats">
      <p>Time: {{ time }}s</p>
      <p>Moves: {{ moves }}</p>
    </div>

    <div class="grid" :style="gridStyle">
      <div
        v-for="(tile, index) in tiles"
        :key="index"
        class="tile"
        :class="{ empty: tile === 0 }"
        @click="moveTile(index)"
      >
        {{ tile !== 0 ? tile : '' }}
      </div>
    </div>

    <button @click="startGame">Restart</button>

    <div v-if="won">
      <h2>You Win!</h2>
      <input v-model="playerName" placeholder="Your name" />
      <button @click="submitScore">Submit Score</button>
    </div>

    <h2>Leaderboard</h2>
    <ul>
      <li v-for="score in leaderboard" :key="score._id">
        {{ score.playerName }} - {{ score.time }}s / {{ score.moves }} moves
      </li>
    </ul>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'

const size = ref(3)
const tiles = ref([])
const time = ref(0)
const moves = ref(0)
const timer = ref(null)
const won = ref(false)
const playerName = ref('')
const leaderboard = ref([])
const difficulty = ref('3')

const API = 'http://localhost:5000/api/scores'

const gridStyle = ref({})

function startGame() {
  size.value = parseInt(difficulty.value)
  const total = size.value * size.value
  tiles.value = shuffle([...Array(total).keys()])

  moves.value = 0
  time.value = 0
  won.value = false

  clearInterval(timer.value)
  timer.value = setInterval(() => time.value++, 1000)

  gridStyle.value = {
    gridTemplateColumns: `repeat(${size.value}, 80px)`
  }

  fetchLeaderboard()
}

function shuffle(arr) {
  return arr.sort(() => Math.random() - 0.5)
}

function moveTile(index) {
  if (won.value) return

  const emptyIndex = tiles.value.indexOf(0)
  const validMoves = getValidMoves(emptyIndex)

  if (validMoves.includes(index)) {
    [tiles.value[index], tiles.value[emptyIndex]] = [
      tiles.value[emptyIndex],
      tiles.value[index]
    ]

    moves.value++

    if (isSolved()) {
      won.value = true
      clearInterval(timer.value)
    }
  }
}

function getValidMoves(emptyIndex) {
  const moves = []
  const row = Math.floor(emptyIndex / size.value)
  const col = emptyIndex % size.value

  if (row > 0) moves.push(emptyIndex - size.value)
  if (row < size.value - 1) moves.push(emptyIndex + size.value)
  if (col > 0) moves.push(emptyIndex - 1)
  if (col < size.value - 1) moves.push(emptyIndex + 1)

  return moves
}

function isSolved() {
  return tiles.value.every((val, i) => val === i)
}

async function submitScore() {
  await axios.post(API, {
    playerName: playerName.value,
    time: time.value,
    moves: moves.value,
    difficulty: difficulty.value
  })

  fetchLeaderboard()
}

async function fetchLeaderboard() {
  const res = await axios.get(`${API}?difficulty=${difficulty.value}`)
  leaderboard.value = res.data
}

onMounted(startGame)
</script>

<style>
.app {
  text-align: center;
  font-family: Arial;
  background: #0a0a0a;
  color: #ffffff;
  min-height: 100vh;
  padding: 20px;
}

h1 {
  color: #ff2c2c;
}

.grid {
  display: grid;
  gap: 8px;
  justify-content: center;
  margin: 20px auto;
}

.tile {
  width: 80px;
  height: 80px;
  background: #1a1a1a;
  border: 2px solid #ff2c2c;
  color: #ff2c2c;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 24px;
  cursor: pointer;
  border-radius: 10px;
  transition: all 0.2s ease;
}

.tile:hover {
  background: #ff2c2c;
  color: #000;
}

.tile.empty {
  background: #000;
  border: 2px dashed #333;
  cursor: default;
}

button {
  background: #ff2c2c;
  color: white;
  border: none;
  padding: 10px 20px;
  margin-top: 10px;
  border-radius: 6px;
  cursor: pointer;
}

button:hover {
  background: #cc0000;
}

select, input {
  background: #111;
  color: white;
  border: 1px solid #ff2c2c;
  padding: 5px;
  border-radius: 5px;
}

.stats {
  margin-top: 10px;
  color: #ffb3b3;
}
</style>


// =========================
// SERVER (Node + Express)
// =========================

// index.js
require('dotenv').config()
const express = require('express')
const mongoose = require('mongoose')
const cors = require('cors')

const app = express()
app.use(cors())
app.use(express.json())

mongoose.connect(process.env.MONGO_URI)
  .then(() => console.log('MongoDB connected'))
  .catch(err => console.log(err))

const Score = mongoose.model('Score', {
  playerName: String,
  time: Number,
  moves: Number,
  difficulty: String,
  date: { type: Date, default: Date.now }
})

app.post('/api/scores', async (req, res) => {
  const score = new Score(req.body)
  await score.save()
  res.json(score)
})

app.get('/api/scores', async (req, res) => {
  const { difficulty } = req.query

  const scores = await Score.find({ difficulty })
    .sort({ time: 1, moves: 1 })
    .limit(10)

  res.json(scores)
})

app.listen(5000, () => console.log('Server running on port 5000'))


// =========================
// .env (inside /server)
// =========================

// MONGO_URI=your_mongodb_connection_string_here
