<script setup lang="ts">
import { ref, onMounted } from 'vue'
import * as Tone from 'tone'

const phoneNumber = ref('06 01 02 03 04')
const midiNotes = ref<number[]>([])
const namesNotes = ref<string[]>([])

let synth: Tone.Synth

onMounted(async () => {
  synth = new Tone.Synth().toDestination()
})

const playPhoneNumber = async () => {
  // Start audio context if needed
  if (Tone.getContext().state !== 'running') {
    await Tone.start()
  }
  
  console.log('Playing phone number:', phoneNumber.value)
  // Extract the last 4 pairs of digits
  const parts = phoneNumber.value.replace(/\s/g, '').match(/.{2}/g) || []
  const lastFour = parts.slice(-4)
  midiNotes.value = lastFour.map(part => parseInt(part, 10))
  console.log('MIDI notes:', midiNotes.value)
  namesNotes.value = midiNotes.value.map(midiNum => Tone.Frequency(midiNum, 'midi').toNote())
  
  // Play the notes in sequence
  const now = Tone.now()
  midiNotes.value.forEach((note, index) => {
    const frequency = Tone.Frequency(note, 'midi').toFrequency()
    synth.triggerAttackRelease(frequency, '8n', now + index * 0.5)
  })
}
</script>

<template>
  <div id="app">
    <div class="header">
      <h1>French Phone Number MIDI Visualizer</h1>
      <div class="input-section">
        <input 
          v-model="phoneNumber" 
          type="text" 
          placeholder="06 01 02 03 04"
          class="phone-input"
        />
        <button @click="playPhoneNumber" class="play-button">
          Translate to MIDI
        </button>
      </div>
    </div>
    
    <div class="visualizer">
      <p>Piano roll will go here</p>
      <p>Phone: {{ phoneNumber }}</p>
      <p>Notes: {{ midiNotes }}</p>
      <p>Names: {{ namesNotes }}</p>
    </div>
  </div>
</template>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: center;
  color: #2c3e50;
  margin-top: 20px;
}

.header {
  margin-bottom: 40px;
}

.input-section {
  margin: 20px 0;
}

.phone-input {
  font-size: 24px;
  padding: 10px;
  width: 300px;
  margin-right: 10px;
  border: 2px solid #3498db;
  border-radius: 5px;
}

.play-button {
  font-size: 18px;
  padding: 12px 20px;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.play-button:hover {
  background-color: #2980b9;
}

.visualizer {
  margin-top: 20px;
}
</style>
