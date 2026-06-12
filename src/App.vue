<script setup>
import { computed, ref } from 'vue'

const symbolPool = [
  { id: 'seven', label: 'Diamond', image: '💎', tone: 'ruby', payout: 12 },
  { id: 'bar', label: 'Trophy', image: '🏆', tone: 'gold', payout: 10 },
  { id: 'cherry', label: 'Cherry', image: '🍒', tone: 'pink' },
  { id: 'bell', label: 'Bell', image: '🔔', tone: 'lemon' },
  { id: 'star', label: 'Star', image: '⭐', tone: 'cyan' },
  { id: 'crown', label: 'Crown', image: '👑', tone: 'violet' },
  { id: 'gem', label: 'Gem', image: '💠', tone: 'blue' },
  { id: 'clover', label: 'Clover', image: '☘', tone: 'green' },
  { id: 'fire', label: 'Fire', image: '🔥', tone: 'ruby' },
  { id: 'coin', label: 'Coin', image: '🪙', tone: 'gold' },
]

const machines = [
  {
    id: 'nova',
    title: 'Neon Nova',
    jackpot: '$18,440',
    credit: '$82.50',
    accent: 'pink',
    reels: ['cherry', 'bar', 'seven', 'star', 'crown', 'gem'],
  },
  {
    id: 'metro',
    title: 'Metro Gold',
    jackpot: '$42,875',
    credit: '$106.20',
    accent: 'gold',
    reels: ['seven', 'seven', 'bar', 'bell', 'star', 'clover'],
  },
  {
    id: 'pulse',
    title: 'Pulse London',
    jackpot: '$297,529',
    credit: '$308.66',
    accent: 'cyan',
    featured: true,
    reels: ['seven', 'bar', 'star', 'cherry', 'crown', 'seven'],
  },
  {
    id: 'volt',
    title: 'Volt Vegas',
    jackpot: '$92,730',
    credit: '$32.39',
    accent: 'green',
    reels: ['bell', 'crown', 'seven', 'bar', 'gem', 'star'],
  },
  {
    id: 'rush',
    title: 'Ruby Rush',
    jackpot: '$65,210',
    credit: '$74.10',
    accent: 'red',
    reels: ['crown', 'star', 'bell', 'gem', 'seven', 'bar'],
  },
]

const activeTiles = ref(Array.from({ length: 5 }, () => pickSymbol()))
const reelStrips = ref(Array.from({ length: 5 }, () => makeReelStrip(pickSymbol())))
const stoppedReels = ref([])
const coins = ref([])
const status = ref('Insert Credit')
const credits = ref(308.66)
const win = ref(0)
const banked = ref(0)
const betLevels = [2.5, 5, 10, 25]
const betIndex = ref(0)
const isSpinning = ref(false)
const leverPulled = ref(false)
const jackpotMode = ref(false)
const isDraggingLever = ref(false)
const leverProgress = ref(0)
const leverStartY = ref(0)
const leverRebounding = ref(false)

const formattedCredits = computed(() => `$${credits.value.toFixed(2)}`)
const formattedWin = computed(() => `$${win.value.toFixed(2)}`)
const formattedBanked = computed(() => `$${banked.value.toFixed(2)}`)
const currentBet = computed(() => betLevels[betIndex.value])
const formattedBet = computed(() => `$${currentBet.value.toFixed(2)}`)
const leverStyle = computed(() => ({
  '--lever-progress': leverProgress.value,
  '--lever-angle': `${leverProgress.value * 180}deg`,
}))

function pickSymbol() {
  return symbolPool[Math.floor(Math.random() * symbolPool.length)]
}

function symbolById(id) {
  return symbolPool.find((symbol) => symbol.id === id) ?? symbolPool[0]
}

function makeRandomTiles(length = 5) {
  return Array.from({ length }, () => pickSymbol())
}

function makeReelStrip(centerSymbol = pickSymbol()) {
  return [
    pickSymbol(),
    pickSymbol(),
    centerSymbol,
    pickSymbol(),
    pickSymbol(),
    pickSymbol(),
    pickSymbol(),
    pickSymbol(),
  ]
}

function makeWinTiles() {
  const winner = symbolPool[Math.floor(Math.random() * 4)]
  return Array.from({ length: 5 }, (_, index) => (index < 3 ? winner : pickSymbol()))
}

function scoreTiles(tiles) {
  const counts = tiles.reduce((acc, symbol) => {
    acc[symbol.id] = (acc[symbol.id] ?? 0) + 1
    return acc
  }, {})
  const match = Object.entries(counts)
    .filter(([, count]) => count >= 3)
    .sort(([, a], [, b]) => b - a)[0]

  if (!match) return null

  return {
    symbol: symbolById(match[0]),
    count: match[1],
  }
}

function burstCoins() {
  const isCompact = window.matchMedia('(max-width: 640px)').matches
  const coinCount = isCompact ? 56 : 110

  coins.value = Array.from({ length: coinCount }, (_, index) => ({
    id: `${Date.now()}-${index}`,
    x: `${Math.random() * 100}%`,
    start: `${8 + Math.random() * 78}%`,
    drift: `${(Math.random() - 0.5) * (isCompact ? 360 : 640)}px`,
    lift: `${150 + Math.random() * (isCompact ? 280 : 420)}px`,
    delay: `${Math.random() * 0.75}s`,
    size: `${12 + Math.random() * (isCompact ? 16 : 22)}px`,
  }))

  window.setTimeout(() => {
    coins.value = []
  }, 4300)
}

function spin() {
  if (isSpinning.value) return

  isSpinning.value = true
  leverPulled.value = true
  leverProgress.value = 1
  leverRebounding.value = true
  jackpotMode.value = false
  status.value = 'Spinning'
  win.value = 0
  stoppedReels.value = []
  reelStrips.value = Array.from({ length: 5 }, () => makeReelStrip())
  credits.value = Math.max(0, credits.value - currentBet.value)

  window.setTimeout(() => {
    leverPulled.value = false
    leverProgress.value = 0
    leverRebounding.value = false
  }, 620)

  const finalTiles = Math.random() > 0.42 ? makeWinTiles() : makeRandomTiles()
  Array.from({ length: 5 }, (_, reelIndex) => {
    window.setTimeout(() => {
      activeTiles.value = activeTiles.value.map((symbol, index) =>
        index === reelIndex ? finalTiles[reelIndex] : symbol,
      )
      reelStrips.value = reelStrips.value.map((strip, index) =>
        index === reelIndex ? makeReelStrip(finalTiles[reelIndex]) : strip,
      )
      stoppedReels.value = [...stoppedReels.value, reelIndex]
      status.value = `Reel ${reelIndex + 1} Locked`

      if (reelIndex === 4) {
        const score = scoreTiles(activeTiles.value)
        win.value = score ? currentBet.value * score.count * (score.symbol.payout ?? 4) : 0
        credits.value += win.value
        status.value = score ? `${score.count} ${score.symbol.label} Match` : 'Try Again'
        jackpotMode.value = Boolean(score)
        if (score) {
          burstCoins()
        }
        isSpinning.value = false
      }
    }, 620 + reelIndex * 360)
  })
}

function increaseBet() {
  if (isSpinning.value) return
  betIndex.value = (betIndex.value + 1) % betLevels.length
  status.value = `Bet ${formattedBet.value}`
}

function maxBet() {
  if (isSpinning.value) return
  betIndex.value = betLevels.length - 1
  status.value = `Max Bet ${formattedBet.value}`
}

function takeWin() {
  if (isSpinning.value || win.value <= 0) return
  banked.value += win.value
  win.value = 0
  jackpotMode.value = false
  status.value = `Collected ${formattedBanked.value}`
}

function startLeverDrag(event) {
  if (isSpinning.value) return

  isDraggingLever.value = true
  leverPulled.value = false
  leverStartY.value = event.clientY
  event.currentTarget.setPointerCapture?.(event.pointerId)
  event.preventDefault()
}

function startLeverTouch(event) {
  const touch = event.touches[0]
  if (!touch || isSpinning.value) return

  isDraggingLever.value = true
  leverPulled.value = false
  leverStartY.value = touch.clientY
  event.preventDefault()
}

function dragLever(event) {
  if (!isDraggingLever.value || isSpinning.value) return

  const distance = Math.max(0, event.clientY - leverStartY.value)
  leverProgress.value = Math.min(1, distance / 74)
  event.preventDefault()
}

function dragLeverTouch(event) {
  const touch = event.touches[0]
  if (!touch || !isDraggingLever.value || isSpinning.value) return

  const distance = Math.max(0, touch.clientY - leverStartY.value)
  leverProgress.value = Math.min(1, distance / 74)
  event.preventDefault()
}

function releaseLever(event) {
  if (!isDraggingLever.value) return

  isDraggingLever.value = false
  event.currentTarget?.releasePointerCapture?.(event.pointerId)

  if (leverProgress.value >= 0.82) {
    spin()
    return
  }

  leverRebounding.value = true
  leverProgress.value = 0
  window.setTimeout(() => {
    leverRebounding.value = false
  }, 320)
}

function releaseLeverTouch(event) {
  releaseLever(event)
}
</script>

<template>
  <main class="casino-floor">
    <div class="ambient ambient--magenta"></div>
    <div class="ambient ambient--cyan"></div>
    <div class="ambient ambient--gold"></div>

    <section class="slot-wall" :class="{ 'slot-wall--jackpot': jackpotMode }" aria-label="Neon slot machine wall">
      <div class="spotlight spotlight--left"></div>
      <div class="spotlight spotlight--right"></div>
      <div class="audience-light audience-light--left"></div>
      <div class="audience-light audience-light--right"></div>

      <div class="wall-copy">
        <p class="eyebrow">Live Terminal Casino</p>
        <h1>Neon Slot Arcade</h1>
      </div>

      <div class="machines" aria-label="Row of casino slot machines">
        <article
          v-for="(machine, index) in machines"
          :key="machine.id"
          class="cabinet"
          :class="[
            `cabinet--${machine.accent}`,
            {
              'cabinet--featured': machine.featured,
              'cabinet--spinning': machine.featured && isSpinning,
              'cabinet--jackpot': machine.featured && jackpotMode,
            },
          ]"
          :style="{ '--offset': index - 2, '--depth': Math.abs(index - 2) }"
        >
          <div class="tower">
            <div class="sign">
              <span>{{ machine.title }}</span>
            </div>
            <div class="beacon"></div>
          </div>

          <div class="cabinet-body">
            <div class="side-light side-light--left"></div>
            <div class="side-light side-light--right"></div>

            <div class="top-strip">
              <span>Grand Jackpot</span>
              <strong>{{ machine.jackpot }}</strong>
            </div>

            <div class="display">
              <template v-if="machine.featured">
                <div class="score-row">
                  <span>Credit {{ formattedCredits }}</span>
                  <span>Win {{ formattedWin }}</span>
                  <span>Bet {{ formattedBet }}</span>
                </div>
                <div class="hero-reels hero-reels--line" :class="{ 'hero-reels--spinning': isSpinning }">
                  <div
                    v-for="(strip, reelIndex) in reelStrips"
                    :key="`reel-${reelIndex}`"
                    class="film-reel"
                    :class="{
                      'film-reel--rolling': isSpinning && !stoppedReels.includes(reelIndex),
                      'film-reel--locked': stoppedReels.includes(reelIndex),
                    }"
                  >
                    <div class="film-strip">
                      <figure
                        v-for="(symbol, stripIndex) in strip"
                        :key="`${symbol.id}-${reelIndex}-${stripIndex}`"
                        class="symbol-card"
                        :class="`symbol-card--${symbol.tone}`"
                      >
                        <span class="symbol-card__image">{{ symbol.image }}</span>
                        <figcaption>{{ symbol.label }}</figcaption>
                      </figure>
                    </div>
                  </div>
                </div>
                <div class="ticker">{{ status }} • Banked {{ formattedBanked }} • Match 3+ In Five Reels</div>
              </template>
              <template v-else>
                <div class="mini-game-title">{{ machine.title }}</div>
                <div class="mini-reels">
                  <span
                    v-for="(symbolId, reelIndex) in machine.reels"
                    :key="`${machine.id}-${reelIndex}`"
                  >
                    {{ symbolById(symbolId).image }}
                  </span>
                </div>
                <div class="mini-prize">Credit {{ machine.credit }}</div>
              </template>
            </div>

            <button
              v-if="machine.featured"
              class="lever"
              :class="{
                'lever--pulled': leverPulled,
                'lever--dragging': isDraggingLever,
                'lever--rebounding': leverRebounding,
              }"
              :style="leverStyle"
              type="button"
              :disabled="isSpinning"
              aria-label="Drag slot machine lever"
              @pointerdown="startLeverDrag"
              @pointermove="dragLever"
              @pointerup="releaseLever"
              @pointercancel="releaseLever"
              @touchstart="startLeverTouch"
              @touchmove="dragLeverTouch"
              @touchend="releaseLeverTouch"
              @touchcancel="releaseLeverTouch"
            >
              <span class="lever__mount"></span>
              <span class="lever__track"></span>
              <span class="lever__arm"></span>
              <span class="lever__knob"></span>
            </button>

            <div class="control-screen">
              <span>More Games</span>
              <strong>Bonus Active</strong>
            </div>

            <div v-if="machine.featured" class="coin-burst" aria-hidden="true">
              <span
                v-for="coin in coins"
                :key="coin.id"
                class="coin"
                :style="{
                  '--coin-x': coin.x,
                  '--coin-start': coin.start,
                  '--coin-drift': coin.drift,
                  '--coin-lift': coin.lift,
                  '--coin-delay': coin.delay,
                  '--coin-size': coin.size,
                }"
              ></span>
            </div>

            <div class="button-deck" :class="{ 'button-deck--featured': machine.featured }">
              <button
                v-if="machine.featured"
                class="spin-button"
                type="button"
                :disabled="isSpinning"
              >
                {{ isSpinning ? 'Rolling' : 'Drag Lever' }}
              </button>
              <span v-if="machine.featured" class="panel-label">Current Bet {{ formattedBet }}</span>
              <span v-else class="button-light button-light--red"></span>
              <button class="deck-button deck-button--green" type="button" @click="machine.featured && increaseBet()">
                <span>Base Bet</span>
              </button>
              <button class="deck-button deck-button--gold" type="button" @click="machine.featured && maxBet()">
                <span>Max Bet</span>
              </button>
              <button class="deck-button deck-button--blue" type="button" @click="machine.featured && takeWin()">
                <span>Collect</span>
              </button>
            </div>
          </div>
        </article>
      </div>
    </section>
  </main>
</template>
