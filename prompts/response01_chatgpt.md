Alright — I’ll give you a **clear, numbered, repeatable recipe** for building your Vue 3 + Vite + TypeScript app from scratch, starting with “Hello World” and adding features step-by-step.

We’ll begin **very minimal** and gradually grow toward your French phone-number → MIDI-tune visualizer.

---

## Step 1 – Create the base project

```bash
npm create vite@latest
```

* **Project name**: `phone-midi-viz`
* **Framework**: `Vue`
* **Variant**: `TypeScript`

Then:

```bash
cd phone-midi-viz
npm install
npm run dev
```

You should now see the default Vite + Vue starter page in your browser at `http://localhost:5173`.

---

## Step 2 – Clean up to a blank app

1. Open `src/App.vue` and replace everything with:

```vue
<template>
  <main>
    <h1>Phone Number to MIDI Visualizer</h1>
  </main>
</template>

<script setup lang="ts">
// No logic yet
</script>

<style scoped>
main {
  font-family: sans-serif;
  padding: 1rem;
}
</style>
```

2. Remove the unused `HelloWorld.vue` component and logo image from `src/assets`.

You should now see just your title.

---

## Step 3 – Add a text box and button

Edit `src/App.vue`:

```vue
<template>
  <main>
    <h1>Phone Number to MIDI Visualizer</h1>
    <input
      type="text"
      v-model="phoneNumber"
      placeholder="Enter French phone number e.g. 06 01 02 03 04"
    />
    <button @click="translateToMidi">Translate to MIDI</button>
  </main>
</template>

<script setup lang="ts">
import { ref } from 'vue';

const phoneNumber = ref('');

function translateToMidi() {
  console.log('Phone number entered:', phoneNumber.value);
}
</script>

<style scoped>
input {
  padding: 0.5rem;
  margin-right: 0.5rem;
  width: 300px;
}
</style>
```

Now typing in the input and pressing the button logs the value.

---

## Step 4 – Install Tone.js for audio playback

```bash
npm install tone
```

---

## Step 5 – Play back the phone number as MIDI notes

Edit `src/App.vue`:

```vue
<script setup lang="ts">
import { ref } from 'vue';
import * as Tone from 'tone';

const phoneNumber = ref('');

function translateToMidi() {
  // Extract digits after the first two (French mobile prefix)
  const matches = phoneNumber.value.match(/\d{2}/g);
  if (!matches || matches.length < 5) {
    alert('Please enter a valid French mobile number');
    return;
  }
  const midiNotes = matches.slice(1).map(num => parseInt(num, 10));

  console.log('MIDI numbers before transpose:', midiNotes);

  // Transpose any notes < 21 (lowest piano key) upward in octaves
  const transposed = midiNotes.map(note => {
    while (note < 21) note += 12;
    return note;
  });

  console.log('MIDI numbers after transpose:', transposed);

  playNotes(transposed);
}

async function playNotes(notes: number[]) {
  await Tone.start();
  const synth = new Tone.Synth().toDestination();
  let time = 0;
  notes.forEach(note => {
    synth.triggerAttackRelease(Tone.Frequency(note, 'midi'), '8n', time);
    time += 0.5;
  });
}
</script>
```

Now typing `06 01 02 03 04` and pressing the button will play four notes (with low ones transposed up).

---

## Step 6 – Add a basic piano roll canvas

1. Create a new file: `src/components/PianoRoll.vue`

```vue
<template>
  <canvas ref="canvas" width="800" height="200"></canvas>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue';

const canvas = ref<HTMLCanvasElement | null>(null);

onMounted(() => {
  const ctx = canvas.value!.getContext('2d')!;
  ctx.fillStyle = '#ddd';
  ctx.fillRect(0, 0, 800, 200);

  // Example: Draw 88 keys
  const keyWidth = 800 / 88;
  for (let i = 0; i < 88; i++) {
    ctx.fillStyle = i % 12 === 1 || i % 12 === 3 || i % 12 === 6 || i % 12 === 8 || i % 12 === 10
      ? '#333'
      : '#fff';
    ctx.fillRect(i * keyWidth, 0, keyWidth, 200);
    ctx.strokeStyle = '#999';
    ctx.strokeRect(i * keyWidth, 0, keyWidth, 200);
  }
});
</script>
```

2. In `src/App.vue`:

```vue
<template>
  <main>
    <h1>Phone Number to MIDI Visualizer</h1>
    <input ... />
    <button @click="translateToMidi">Translate to MIDI</button>
    <PianoRoll />
  </main>
</template>

<script setup lang="ts">
import PianoRoll from './components/PianoRoll.vue';
...
</script>
```

You should now see a simple piano layout.

---

## Step 7 – Show mouse-over note info

We’ll track mouse position over the piano roll and show note details in a status bar.

1. Update `PianoRoll.vue`:

```vue
<template>
  <div>
    <canvas
      ref="canvas"
      width="800"
      height="200"
      @mousemove="handleMouse"
      @click="playNoteClick"
    ></canvas>
    <div>{{ status }}</div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue';
import * as Tone from 'tone';

const canvas = ref<HTMLCanvasElement | null>(null);
const status = ref('');
const keyWidth = 800 / 88;

onMounted(() => {
  drawPiano();
});

function drawPiano() {
  const ctx = canvas.value!.getContext('2d')!;
  ctx.clearRect(0, 0, 800, 200);
  for (let i = 0; i < 88; i++) {
    const isBlack = [1, 3, 6, 8, 10].includes(i % 12);
    ctx.fillStyle = isBlack ? '#333' : '#fff';
    ctx.fillRect(i * keyWidth, 0, keyWidth, 200);
    ctx.strokeStyle = '#999';
    ctx.strokeRect(i * keyWidth, 0, keyWidth, 200);
  }
}

function handleMouse(e: MouseEvent) {
  const rect = canvas.value!.getBoundingClientRect();
  const x = e.clientX - rect.left;
  const noteIndex = Math.floor(x / keyWidth);
  const midiNumber = 21 + noteIndex; // A0 is MIDI 21
  const freq = Tone.Frequency(midiNumber, 'midi').toFrequency().toFixed(2);
  const noteName = Tone.Frequency(midiNumber, 'midi').toNote();
  status.value = `Note: ${noteName} (MIDI ${midiNumber}, ${freq} Hz)`;
}

async function playNoteClick(e: MouseEvent) {
  const rect = canvas.value!.getBoundingClientRect();
  const x = e.clientX - rect.left;
  const noteIndex = Math.floor(x / keyWidth);
  const midiNumber = 21 + noteIndex;
  await Tone.start();
  const synth = new Tone.Synth().toDestination();
  synth.triggerAttackRelease(Tone.Frequency(midiNumber, 'midi'), '8n');
}
</script>
```

Now moving over the piano roll shows the note info, clicking plays it.

---

## Step 8 – Integrate number playback with visual scroll (final core feature)

We’d:

* Store the translated notes in App.vue state
* Pass them to PianoRoll.vue
* Animate a vertical line moving across during playback

(We can implement this in the next steps after confirming the basic pieces work.)

---

If you like, I can continue this recipe to **Step 9–12** where we add:

* Passing notes to piano roll
* Drawing note rectangles
* Scrolling playback line
* Spacebar-triggered playback

Do you want me to continue building that out?
