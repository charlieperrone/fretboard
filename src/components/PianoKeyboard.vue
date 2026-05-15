<template>
  <div class="piano-wrapper">
    <div class="piano" role="application" aria-label="Piano keyboard">
      <div class="keys">
        <div v-for="oct in octaves" :key="oct" class="octave">
          <div
            v-for="(noteName, idx) in notes"
            :key="`${noteName}-${oct}`"
            :class="['key', isBlack(idx) ? 'black' : 'white', { active: visible[noteName] }]"
            @click="toggleNote(noteName)"
            :title="`${noteShort(noteName)}${oct}`"
          >
            <span
              v-if="visible[noteName]"
              class="key-dot has-tooltip"
              :style="{ backgroundColor: noteColors[noteName] }"
              :data-tooltip="`${noteShort(noteName)}${oct}`"
              aria-hidden="true"
            ></span>

            <span class="key-label" v-if="!isBlack(idx)"
              >{{ noteShort(noteName) }}<sub>{{ oct }}</sub></span
            >
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
defineProps<{
  notes: string[]
  noteColors: Record<string, string>
  visible: Record<string, boolean>
  octaves: number[]
  isBlack: (idx: number) => boolean
  noteShort: (n: string) => string
  toggleNote: (n: string) => void
}>()
</script>

<style scoped>
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

.octave {
  display: flex;
  position: relative;
}

.key {
  box-sizing: border-box;
  cursor: pointer;
  font-family: sans-serif;
  position: relative; /* ensure dot can be absolutely positioned */
}

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

.key.active.white {
  box-shadow: inset 0 -30px 0 rgba(0, 0, 0, 0.08);
  outline: 3px solid rgba(0, 0, 0, 0.08);
}

.key.active.black {
  box-shadow:
    0 2px 6px rgba(0, 0, 0, 0.4),
    inset 0 -10px 0 rgba(255, 255, 255, 0.04);
  border: 2px solid rgba(255, 255, 255, 0.06);
}

.key-label {
  font-size: 10px;
  color: #222;
  margin-bottom: 3px;
  display: none;
}

/* dot that indicates the note is enabled */
.key-dot {
  position: absolute;
  border-radius: 50%;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.35);
  pointer-events: none;
}

.key.white .key-dot {
  width: 12px;
  height: 12px;
  bottom: 8px;
  left: 50%;
  transform: translateX(-50%);
  border: 2px solid rgba(255, 255, 255, 0.9);
}

.key.black .key-dot {
  width: 10px;
  height: 10px;
  bottom: 2rem;
  left: 50%;
  transform: translateX(-50%);
  border: 1px solid rgba(255, 255, 255, 0.6);
}
</style>
