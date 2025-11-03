<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref, watch } from 'vue'
import {
  Chart,
  LineController,
  LineElement,
  PointElement,
  LinearScale,
  Title,
  CategoryScale,
} from 'chart.js'

// Register necessary Chart.js components
Chart.register(LineController, LineElement, PointElement, LinearScale, Title, CategoryScale)

const loop_ms = 50
const display_seconds = 10
const timeout_seconds = 15
const margin = 0.15
const countdown_seconds = 3

const listSize = Math.floor((display_seconds * 1000) / loop_ms)

// --- Fixed Time Vector ---
const dt = loop_ms / 1000
const timeVector: number[] = Array.from({ length: listSize }, (_, i) => -display_seconds + i * dt)
timeVector[timeVector.length - 1] = 0

// --- Data Lists (Y-values) ---
let channel1: number[] = Array(listSize).fill(NaN)
let channel2: number[] = Array(listSize).fill(NaN)

// Chart reference
let chart: Chart<'line'> | undefined
const canvasRef = ref<HTMLCanvasElement | null>(null)

function updateChart() {
  if (chart) {
    chart.data.datasets[0].data = channel1
    chart.data.datasets[1].data = channel2
    chart.update('none')
  }
}

// --- Add a new Y-value and keep list fixed size ---
// --- Add new values for both channels ---
function addValues(value1: number, value2: number) {
  // Channel 1
  channel1.push(value1)
  if (channel1.length > listSize) channel1.shift()

  // Channel 2
  channel2.push(value2)
  if (channel2.length > listSize) channel2.shift()

  updateChart()
}

const loop_active = ref(false)
let intervalId: number | null = null

// Watch for changes in the reactive value
watch(loop_active, (newVal) => {
  if (newVal) {
    // Start the loop
    intervalId = window.setInterval(() => {
      gameLoop()
    }, loop_ms)
  } else {
    // Stop the loop
    if (intervalId !== null) {
      clearInterval(intervalId)
      intervalId = null
      if (!aborted) finished.value = true
      aborted = false
    }
  }
})

// --- Initialize Chart.js ---
onMounted(() => {
  if (!canvasRef.value) return

  chart = new Chart(canvasRef.value, {
    type: 'line',
    data: {
      labels: timeVector,
      datasets: [
        {
          label: 'Channel 1',
          data: channel1,
          borderColor: 'rgb(75, 192, 192)',
          borderWidth: 1.5,
          pointRadius: 0,
          fill: false,
          tension: 0.1,
        },
        {
          label: 'Channel 2',
          data: channel2,
          borderColor: 'rgb(255, 99, 132)',
          borderWidth: 1.5,
          pointRadius: 0,
          fill: false,
          tension: 0.1,
        },
      ],
    },
    options: {
      animation: false,
      responsive: true,
      scales: {
        x: {
          title: { display: false, text: 'Time' },
          ticks: { display: false },
        },
        y: {
          title: { display: false, text: 'Value' },
          ticks: { display: false },
          min: -1.5,
          max: 1.5,
        },
      },
      plugins: {
        legend: { display: true, position: 'top' },
      },
    },
  })
})

onBeforeUnmount(() => {
  // Clean up when component unmounts
  if (intervalId !== null) {
    clearInterval(intervalId)
  }
  chart?.destroy()
})

const userInput = ref(0)
const game_mode = ref('humanil')
const target_function = ref('1')
const score = ref(0)
const countdown = ref(-1)
const finished = ref(false)
let aborted = false

function reset() {
  score.value = 0
  userInput.value = 0
  if (loop_active.value) aborted = true
  loop_active.value = false
  finished.value = false

  channel1 = Array(listSize).fill(NaN)
  channel2 = Array(listSize).fill(NaN)

  t = 0
  y1 = y2 = 0
  integral = lastError = 0

  updateChart()
}

// --- Constants ---
const MODE: 'PT1' | 'PT2' = 'PT2' // switch here
const K = 1.0
const T = 1.0 // time constant (for PT1)
const D = 0.5 // damping ratio (for PT2)
const wn = 2.0 // natural frequency (for PT2)

// ---- Controller parameters (PID) ----
const Kp = 3.28
const Ki = 3.38
const Kd = 1.48

// --- State ---
let t = 0

let y1 = 0 // output
let y2 = 0 // derivative (for PT2)

let integral = 0
let lastError = 0

// ---- Target function ----
function targetFunction(t: number): number {
  let freq = 0.4
  // triangle
  if (target_function.value === '1') 
    return 1 - 4 * Math.abs(((((t + 0.25 / freq) * freq) / 2) % 1) - 0.5)
  // sawtooth
  else if (target_function.value === '2') return 2 * (((t + 0.25 / freq) * freq) % 1) - 1
  // double sine
  else if (target_function.value === '3')
    return Math.sin(0.3 * t) + 0.3 * Math.sin(2 * t)
  // simple sine
  else return Math.sin(0.3 * t)
}

// ---- Controller ----
function controller(target: number, actual: number): number {
  const error = target - actual
  integral += error * dt
  const derivative = (error - lastError) / dt
  lastError = error

  // PID output
  const u = Kp * error + Ki * integral + Kd * derivative
  return u
}

function updateSystem(u: number) {
  if (MODE === 'PT1') {
    // PT1 system
    y1 += (dt / T) * (K * u - y1)
  } else {
    // PT2 system
    y1 += dt * y2
    y2 += dt * (wn * wn * (K * u - y1) - 2 * D * wn * y2)
  }

  return y1
}

let countdownTimer: number | null = null
function startStop() {
  if (loop_active.value) {
    loop_active.value = false
    return
  }

  // skip countdown for closedL-auto
  if(game_mode.value === 'closedL-auto')
  {
    loop_active.value = true
    return
  }

  // If countdown is already running, stop it and reset state
  if (countdownTimer !== null) {
    clearInterval(countdownTimer)
    countdownTimer = null
    countdown.value = -1
    loop_active.value = false
    return
  }

  countdown.value = countdown_seconds
  loop_active.value = false

  countdownTimer = window.setInterval(() => {
    countdown.value -= 1

    if (countdown.value <= 0) {
      clearInterval(countdownTimer!)
      countdownTimer = null
      countdown.value = -1
      loop_active.value = true
    }
  }, 1000)
}

function gameLoop() {
  let input = 0
  let target = targetFunction(t)
  t += dt

  // stop the game
  if (t >= timeout_seconds) {
    loop_active.value = false
  }

  if (game_mode.value === 'humanil' || game_mode.value === 'openL') {
    input = userInput.value / 100

    if (game_mode.value === 'openL') {
      target = input.valueOf()
    }
  } else if (game_mode.value === 'closedL' || game_mode.value === 'closedL-auto') {
    if (game_mode.value === 'closedL') {
      target = userInput.value / 100
    }

    input = controller(target, y1)
  }

  // system response
  let output = updateSystem(input)

  // scoring
  if (game_mode.value === 'humanil' || game_mode.value === 'closedL-auto') {
    let error = target - output
    if (Math.abs(error) < margin) score.value += 1
  }

  addValues(target, output)
}
</script>

<template>
  Understand the basics of control loops
  <h1>Score: {{ score }}</h1>
  <article>
    <div>
      <canvas ref="canvasRef"></canvas>
    </div>
  </article>
  <br />
    
  <div role="group">
    <button @click="startStop" style="width: 120px" :disabled="finished">
      <span v-if="countdown > 0">{{ countdown }}</span>
      <span v-else-if="!loop_active">START</span>
      <span v-else>STOP</span>
    </button>
    <button @click="reset" class="secondary" :class="{ outline: !finished }">RESET</button>
  </div>

  <input
    type="range"
    id="userInput"
    min="-150"
    max="150"
    list="middle"
    v-model="userInput"
    :disabled="game_mode === 'closedL-auto'"
  />
  <datalist id="middle">
    <option value="0"></option>
  </datalist>

  <table><thead><tr>
    <th>
      <label for="game-mode">Mode</label>
      <select id="game-mode" v-model="game_mode" :disabled="loop_active">
        <option value="openL">Open Loop</option>
        <option value="humanil">HumanIL</option>
        <option value="closedL">PID</option>
        <option value="closedL-auto">PID (observe)</option>
      </select>
    </th>
    <th>
      <label for="target-function">Function</label>
      <select
        id="target-function"
        v-model="target_function"
        :disabled="loop_active || game_mode === 'openL' || game_mode === 'closedL'"
      >
        <option value="0">0</option>
        <option value="1">1</option>
        <option value="2">2</option>
        <option value="3">3</option>
      </select>
    </th>
  </tr></thead></table>

    

    

  <article id="help">
    <header><h1>🗒️ Instructions</h1></header>

    <p>🏁 Goal: <br />Bring the red line as close to the green line as possible.</p>
    <p>🥇 Scoring: <br />If the lines are within a defined margin, score is increased.</p>
    <p>START initiates a 3s countdown. Click again to abort the countdown.</p>
    <p>RESET clears the chart.</p>

    <h4>Game Modes</h4>
    <p>Choose a game mode from the Dropdown</p>

    <details>
      <summary role="button" class="contrast outline">Open Loop</summary>
      <p>Observe behaviour of process (no scoring)</p>
      <p><img src="/images/humanil-game-openL.svg" class="dark-mode-invert"></img></p>
      <p>🎚️ Slider: Process Input</p>
      <p>🟢: Process input</p>
      <p>🔴: Process output</p>
    </details>

    <details>
      <summary role="button" class="contrast outline">HumanIL</summary>
      <p>Human is the controller and must follow a given set point value.</p>
      <p><img src="/images/humanil-game-HumanIL.svg" class="dark-mode-invert"></img></p>
      <p>🎚️ Slider: Process Input</p>
      <p>🟢: Set point</p>
      <p>🔴: Process variable</p>
    </details>

    <details>
      <summary role="button" class="contrast outline">PID</summary>
      <p>Observe behaviour of controller (manual set point, no scoring)</p>
      <p><img src="/images/humanil-game-openL.svg" class="dark-mode-invert"></img></p>
      <p>🎚️ Slider: Set point value</p>
      <p>🟢: Set point</p>
      <p>🔴: Process variable</p>
    </details>
    
    <details>
      <summary role="button" class="contrast outline">PID (observe)</summary>
      <p>Observe behaviour of controller (function as set point)</p>
      <p><img src="/images/humanil-game-openL.svg" class="dark-mode-invert"></img></p>
      <p>🎚️ Slider: deactivated</p>
      <p>🟢: Set point</p>
      <p>🔴: Process variable</p>
    </details>
    
  </article>
</template>

<style scoped>
/* Apply only in dark mode */
@media (prefers-color-scheme: dark) {
  @supports (-webkit-overflow-scrolling: touch) {
    .dark-mode-invert {
      filter: invert(1) brightness(1.2);
    }
  }
}
</style>