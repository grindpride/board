<template>
  <div class="audio-recorder">
    <label >
      Выбери микрофон:
      <select id="microphone-select" v-model="selectedMicrophone">
        <option v-for="microphone in microphones" :key="microphone.deviceId" :value="microphone.deviceId">
          {{ microphone.label }}
        </option>
      </select>
    </label>
    <label>
      Включить дисторшон
      <input type="checkbox" v-model="isDistortion">
    </label>

    <button @click="toggleRecording">
      {{ isRecording ? 'Stop stream' : 'Start stream' }}
    </button>
    <canvas ref="visualizer"></canvas>
  </div>
</template>

<script lang="ts" setup>
import {onUnmounted, ref, watch} from 'vue';

const isRecording = ref(false);
const isDistortion = ref(false);
const selectedMicrophone = ref('');
const microphones = ref<MediaDeviceInfo[]>([]);
const visualizer = ref<HTMLCanvasElement>(null)
let audioContext: AudioContext;
let audioStream: MediaStream | undefined;

navigator.mediaDevices.enumerateDevices().then((devices) => {
  const audioDevices = devices.filter((device) => device.kind === 'audioinput');
  microphones.value = audioDevices;
  selectedMicrophone.value = audioDevices.length > 0 ? audioDevices[0].deviceId : '';
});

function makeDistortionCurve(amount: number) {
  const k = amount;
  const n_samples = 44100;
  const curve = new Float32Array(n_samples);
  const deg = Math.PI / 180;
  let x: number;

  for (let i = 0; i < n_samples; ++i) {
    x = (i * 2) / n_samples - 1;
    curve[i] = ((3 + k) * x * 20 * deg) / (Math.PI + k * Math.abs(x));
  }

  return curve;
}


async function getMediaStream() {
  try {
    const constraints = {audio: {deviceId: selectedMicrophone.value}};
    return await navigator.mediaDevices.getUserMedia(constraints);
  } catch (err) {
    console.error('Error getting audio stream:', err);
  }
}

async function startRecording() {
  audioStream = await getMediaStream();

  if (audioStream) {
    audioContext = new AudioContext();
    const analyserNode = new AnalyserNode(audioContext, { fftSize: 256 })
    const mediaStreamSource = audioContext.createMediaStreamSource(audioStream).connect(analyserNode);


    if(isDistortion.value){
      const distortion = audioContext.createWaveShaper();
      distortion.curve = makeDistortionCurve(400);
      mediaStreamSource.connect(distortion);
      distortion.connect(audioContext.destination);
    }else {
      mediaStreamSource.connect(audioContext.destination);
    }

    isRecording.value = true;
  }
}

function stopRecording() {
  if (audioStream) {
    audioStream.getTracks().forEach((track) => track.stop());
    audioContext.close();
    isRecording.value = false;
  }
}

function toggleRecording() {
  if (isRecording.value) {
    stopRecording();
  } else {
    startRecording();
  }
}

async function restart(){
  if (isRecording.value) {
    stopRecording();
    await new Promise((resolve) => setTimeout(resolve, 500));
    toggleRecording()
  }
}

watch([selectedMicrophone, isDistortion], restart);

onUnmounted(() => {
  stopRecording();
});
</script>

<style>
body{
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
}
.audio-recorder {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 20px;
}
</style>