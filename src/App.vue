<template>
  <div class="container">
    <div class="fretboard-wrapper">
      <!-- Fret numbers column -->
      <div class="fret-numbers">
        <!-- Open string row -->
        <div class="fret-number"></div>
        <!-- Frets 1–12 -->
        <div v-for="fret in fretCount" :key="fret" class="fret-number">
          <span v-if="[5, 9, 12].includes(fret)">{{ fret }}</span>
        </div>
      </div>

      <!-- Fretboard -->
      <div class="fretboard">
        <!-- Open string row -->
        <div class="fret-row open-row">
          <div
            v-for="openNote in strings"
            :key="`open-${openNote}`"
            class="fret-cell open-cell"
            @click="toggleAt(openNote, 0)"
            @keydown.enter.prevent="toggleAt(openNote, 0)"
            @keydown.space.prevent="toggleAt(openNote, 0)"
            role="button"
            tabindex="0"
            :aria-label="`Toggle ${noteAt(openNote, 0)}`"
          >
            <template v-for="n in activeNotesAt(openNote, 0)" :key="n">
              <span
                class="dot open-dot has-tooltip"
                :style="{ backgroundColor: noteColors[n], borderColor: noteColors[n] }"
                :data-tooltip="n"
                :aria-label="n"
              ></span>
            </template>
          </div>
        </div>

        <!-- Frets 1–12 -->
        <div v-for="fret in fretCount" :key="fret" class="fret-row">
          <div
            v-for="openNote in strings"
            :key="`${openNote}-${fret}`"
            class="fret-cell"
            @click="toggleAt(openNote, fret)"
            @keydown.enter.prevent="toggleAt(openNote, fret)"
            @keydown.space.prevent="toggleAt(openNote, fret)"
            role="button"
            tabindex="0"
            :aria-label="`Toggle ${noteAt(openNote, fret)}`"
          >
            <template v-for="n in activeNotesAt(openNote, fret)" :key="n">
              <span
                class="dot has-tooltip"
                :style="{ backgroundColor: noteColors[n], borderColor: noteColors[n] }"
                :data-tooltip="n"
                :aria-label="n"
              ></span>
            </template>
          </div>
        </div>
      </div>
    </div>

    <!-- Interaction Panel -->
    <div class="controls">
      <label
        v-for="n in notes.slice().reverse()"
        :key="n"
        class="note-toggle"
        :style="{ color: noteColors[n] }"
      >
        <input type="checkbox" v-model="visible[n]" />
        {{ n }}
      </label>
      <div class="shift-controls">
        <button type="button" @click="shiftAll(1)" title="Shift up 1 semitone">▲ Up</button>
        <button type="button" @click="shiftAll(-1)" title="Shift down 1 semitone">▼ Down</button>
      </div>
      <button type="button" @click="clearAll" class="clear-button">Clear</button>
    </div>

    <div class="right-container">
      <!-- Piano view (5 octaves) -->
      <PianoKeyboard
        :notes="notes"
        :note-colors="noteColors"
        :visible="visible"
        :octaves="octaves"
        :is-black="isBlack"
        :note-short="noteShort"
        :toggle-note="toggleNote"
      />
      <!-- Circle of Fifths image from public folder -->
      <div class="circle-image">
        <img src="/fifths.jpg" alt="Circle of Fifths" />
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { reactive, onMounted, watch } from 'vue'
import PianoKeyboard from './components/PianoKeyboard.vue'

// Tunings + chromatic scale
const strings = ['E', 'A', 'D', 'G', 'B', 'E']
const notes = ['C', 'C#/Db', 'D', 'D#/Eb', 'E', 'F', 'F#/Gb', 'G', 'G#/Ab', 'A', 'A#/Bb', 'B']
const fretCount = 13

// Rainbow-ish note colors (change as you like)
const noteColors: Record<string, string> = {
  C: '#ff0000', // red
  'C#/Db': '#ff7f00', // orange
  D: '#ffff00', // yellow
  'D#/Eb': '#bfff00', // lime
  E: '#00ff00', // green
  F: '#00ff7f', // spring
  'F#/Gb': '#00ffff', // cyan
  G: '#007fff', // azure
  'G#/Ab': '#0000ff', // blue
  A: '#4b00ff', // indigo
  'A#/Bb': '#8b00ff', // violet
  B: '#ff00ff', // magenta
}

// Toggle states — all OFF initially
const visible = reactive<Record<string, boolean>>(Object.fromEntries(notes.map((n) => [n, false])))

// Helper: note at string + fret
const noteAt = (open: string, fret: number): string => {
  const idx = (notes.indexOf(open) + fret) % notes.length
  return notes[idx]!
}

// Return *all* visible notes at this cell (could be > 1 if enharmonic someday)
const activeNotesAt = (openNote: string, fret: number): string[] => {
  const pitch = noteAt(openNote, fret)
  return pitch && visible[pitch] ? [pitch] : []
}

// --- Piano view additions ---

// Start octave (C2..B6 covers 5 octaves: 2,3,4,5,6)
const baseOctave = 2
const octaves = Array.from({ length: 5 }, (_, i) => baseOctave + i)

// black key indices in the notes array
const blackIndices = new Set([1, 3, 6, 8, 10])
const isBlack = (idx: number) => blackIndices.has(idx)

// Short label for keys (prefer the sharp side of enharmonic pair)
const noteShort = (n: string) => (n.includes('/') ? n.split('/')[0] : n)

// Allow clicking a piano key to toggle the same note class across octaves
const toggleNote = (noteName: string) => {
  visible[noteName] = !visible[noteName]
}

// Toggle the pitch at a given string+fret
const toggleAt = (openNote: string, fret: number) => {
  const pitch = noteAt(openNote, fret)
  if (!pitch) return
  visible[pitch] = !visible[pitch]
}

// Shift all currently visible notes by delta semitones (positive or negative)
const shiftAll = (delta: number) => {
  // Collect currently active note names
  const active = Object.keys(visible).filter((n) => visible[n])
  if (active.length === 0) return

  // Build a new visibility map (start all false)
  const nextState: Record<string, boolean> = Object.fromEntries(notes.map((n) => [n, false]))

  // Map each active note to its new pitch class
  active.forEach((name) => {
    const idx = notes.indexOf(name)
    if (idx === -1) return
    const newIdx = (idx + delta + notes.length) % notes.length
    const newName = notes[newIdx]!
    nextState[newName] = true
  })

  // Apply the new visibility into the reactive object
  Object.keys(nextState).forEach((k) => {
    visible[k] = !!nextState[k]
  })
}

// Clear all toggles
const clearAll = () => {
  Object.keys(visible).forEach((k) => {
    visible[k] = false
  })
}

// Load saved state from localStorage on mount
onMounted(() => {
  try {
    const raw = localStorage.getItem('fretboard-visible')
    if (!raw) return
    const saved = JSON.parse(raw)
    Object.keys(visible).forEach((k) => {
      visible[k] = !!saved[k]
    })
  } catch {
    // ignore
  }
})

// Watch visible changes and save to localStorage
watch(
  () => Object.keys(visible).filter((k) => visible[k]),
  () => {
    const map = Object.fromEntries(Object.keys(visible).map((k) => [k, visible[k]]))
    localStorage.setItem('fretboard-visible', JSON.stringify(map))
  },
  { deep: true },
)
</script>

<style scoped>
.container {
  display: flex;
}
.fretboard-wrapper {
  display: flex;
}

/* Fret numbers column */
.fret-numbers {
  display: flex;
  flex-direction: column;
  margin-right: 6px;
  gap: 1px;
}

.fret-number {
  width: 1.5rem;
  height: 3rem; /* same as fret-cell */
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.8rem;
  color: #555;
  font-family: sans-serif;
}

/* Open string row height matches cells */
.fret-number:first-child {
  height: 3rem;
}

/* Fretboard (same as before) */
.fretboard {
  display: flex;
  flex-direction: column;
  /* border: 2px solid #222; */
}

.fret-row {
  display: flex;
  flex-direction: row;

  .fret-cell {
    width: 2.5rem;
    height: 3rem;
    /* border: 1px solid #aaa; */
    border-left: 1px solid;
    border-bottom: 1px solid;
    display: flex;
    align-items: center;
    justify-content: center;
    position: relative;
  }

  .fret-cell:last-child {
    border-bottom: none;
  }
}

/* Dots */
.dot {
  width: 12px;
  height: 12px;
  left: -6px;
  bottom: 10px;
  border-radius: 50%;
  /* border: 2px solid; */
  position: absolute;
}

/* Slightly shorter open string cells so you see the "nut" visually */
.open-row .fret-cell {
  border-bottom: 2px solid #000;
}

.open-row {
  &:not(:last-child) {
    background: #eee;
  }
}

.controls {
  display: flex;
  flex-direction: column;
}

.shift-controls {
  margin-top: 1rem;
  display: flex;
  gap: 6px;
  margin-bottom: 8px;
}
.shift-controls button {
  background: #f3f4f6;
  border: 1px solid #ccc;
  padding: 6px 8px;
  border-radius: 4px;
  cursor: pointer;
  font-family: sans-serif;
}
.shift-controls button:hover {
  background: #e9eef6;
}

/* Piano styles */
.piano-wrapper {
  margin-left: 1rem;
  display: flex;
  align-items: flex-start;
}

.piano {
  user-select: none;
}

.keys {
  display: flex;
  align-items: flex-end;
}

/* Each octave groups 12 keys in sequence; keys are narrow to fit */
.octave {
  display: flex;
  position: relative;
}

/* Base styles for keys */
.key {
  box-sizing: border-box;
  cursor: pointer;
  font-family: sans-serif;
}

/* White keys */
.key.white {
  width: 28px;
  height: 120px;
  background: #fff;
  border: 1px solid #ccc;
  display: flex;
  align-items: flex-end;
  justify-content: center;
  position: relative;
  z-index: 1;
}

/* Black keys: overlap white keys */
.key.black {
  width: 18px;
  height: 80px;
  background: #111;
  margin-left: -9px;
  margin-right: -9px;
  border-radius: 2px;
  z-index: 2;
  position: relative;
}

/* Active coloring uses noteColors where possible via inline style on dot in fretboard,
   here we'll rely on 'active' class and set color via CSS variables (applied inline) */
.key.active.white {
  box-shadow: inset 0 -30px 0 rgba(0, 0, 0, 0.08);
  outline: 3px solid rgba(0, 0, 0, 0.08);
}

/* show colored border for active black keys */
.key.active.black {
  box-shadow:
    0 2px 6px rgba(0, 0, 0, 0.4),
    inset 0 -10px 0 rgba(255, 255, 255, 0.04);
  border: 2px solid rgba(255, 255, 255, 0.06);
}

/* show custom colored indicator on keys using pseudo-element via inline style might be easier,
   but to keep things simple, we'll tint active keys using background color via style attribute when rendered.
*/

.key-label {
  font-size: 10px;
  color: #222;
  margin-bottom: 3px;
}

.key {
  box-sizing: border-box;
  cursor: pointer;
  font-family: sans-serif;
  position: relative; /* ensure dot can be absolutely positioned */
}

/* dot that indicates the note is enabled */
.key-dot {
  position: absolute;
  border-radius: 50%;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.35);
  pointer-events: none;
}

/* position/size for white keys */
.key.white .key-dot {
  width: 12px;
  height: 12px;
  top: 8px;
  left: 50%;
  transform: translateX(-50%);
  border: 2px solid rgba(255, 255, 255, 0.9);
}

/* position/size for black keys (smaller, lighter border) */
.key.black .key-dot {
  width: 10px;
  height: 10px;
  top: 6px;
  left: 50%;
  transform: translateX(-50%);
  border: 1px solid rgba(255, 255, 255, 0.6);
}

/* fast CSS tooltip for dots (appears immediately on hover) */
.has-tooltip[data-tooltip]:hover::after {
  content: attr(data-tooltip);
  position: absolute;
  white-space: nowrap;
  top: -28px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0, 0, 0, 0.85);
  color: #fff;
  padding: 4px 6px;
  border-radius: 4px;
  font-size: 12px;
  z-index: 999;
  pointer-events: none;
}

.has-tooltip[data-tooltip]:hover::before {
  content: '';
  position: absolute;
  top: -8px;
  left: 50%;
  transform: translateX(-50%);
  border-width: 6px 6px 0 6px;
  border-style: solid;
  border-color: rgba(0, 0, 0, 0.85) transparent transparent transparent;
  z-index: 999;
  pointer-events: none;
}

.circle-image {
  /* margin-left: 1rem; */
  display: flex;
  align-items: center;
}
.circle-image img {
  height: auto;
  width: 30rem;
  display: block;
}

.right-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
  height: 100vh;
}

.fret-cell.clickable,
.fret-cell[role='button'] {
  cursor: pointer;
}

.clear-button {
  margin-top: 8px;
  background: #fff;
  border: 1px solid #ccc;
  padding: 6px 8px;
  border-radius: 4px;
  cursor: pointer;
}
.clear-button:hover {
  background: #f6f6f6;
}
</style>
