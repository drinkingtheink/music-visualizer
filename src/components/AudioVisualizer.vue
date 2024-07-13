<template>
  <div class="visualizer">
    <input type="file" @change="handleFileUpload" />
    <button @click="togglePlayback" :disabled="!audioBuffer">
      {{ isPlaying ? 'Pause' : 'Play' }}
    </button>
    <!-- <select v-model="visualizationType">
      <option value="waveform">Waveform</option>
      <option value="bars">Bars</option>
      <option value="circles">Circles</option>
    </select> -->
    <section class="controls">
      <label for="lineWidth">Line Width:</label>
      <input type="number" v-model="lineWidth" min="1" max="10" />
      <label for="strokeStyle">Line Color:</label>
      <input type="color" v-model="strokeStyle" />
    </section>

    <canvas ref="canvas"></canvas>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue';

export default {
  setup() {
    const audioContext = new (window.AudioContext || window.webkitAudioContext)();
    const canvas = ref(null);
    const audioBuffer = ref(null);
    const audioSource = ref(null);
    const analyser = ref(null);
    const dataArray = ref(null);
    const isPlaying = ref(false);
    const visualizationType = ref('waveform');
    const lineWidth = ref(9);
    const strokeStyle = ref('#000000');
    let startTime = 0;
    let pausedTime = 0;

    const handleFileUpload = (event) => {
      const file = event.target.files[0];
      if (file && (file.type === 'audio/mpeg' || file.type === 'audio/mp3')) {
        const reader = new FileReader();
        reader.onload = (e) => {
          const arrayBuffer = e.target.result;
          audioContext.decodeAudioData(arrayBuffer, (buffer) => {
            audioBuffer.value = buffer;
          });
        };
        reader.readAsArrayBuffer(file);
      }
    };

    const playAudio = (resume = false) => {
      if (!audioBuffer.value) return;
      if (audioSource.value) audioSource.value.stop();

      audioSource.value = audioContext.createBufferSource();
      audioSource.value.buffer = audioBuffer.value;

      analyser.value = audioContext.createAnalyser();
      analyser.value.fftSize = 2048;
      const bufferLength = analyser.value.frequencyBinCount;
      dataArray.value = new Uint8Array(bufferLength);

      audioSource.value.connect(analyser.value);
      analyser.value.connect(audioContext.destination);

      if (resume) {
        startTime = audioContext.currentTime - pausedTime;
        audioSource.value.start(0, pausedTime);
      } else {
        startTime = audioContext.currentTime;
        audioSource.value.start();
      }
      isPlaying.value = true;
      drawVisualizer();
    };

    const pauseAudio = () => {
      if (!audioSource.value) return;
      audioSource.value.stop();
      pausedTime = audioContext.currentTime - startTime;
      isPlaying.value = false;
    };

    const togglePlayback = () => {
      if (isPlaying.value) {
        pauseAudio();
      } else {
        playAudio(pausedTime > 0);
      }
    };

    const drawVisualizer = () => {
      if (!isPlaying.value) return;
      const canvasCtx = canvas.value.getContext('2d');
      const WIDTH = canvas.value.width;
      const HEIGHT = canvas.value.height;

      requestAnimationFrame(drawVisualizer);

      analyser.value.getByteTimeDomainData(dataArray.value);

      canvasCtx.clearRect(0, 0, WIDTH, HEIGHT);

      switch (visualizationType.value) {
        case 'waveform':
          drawWaveform(canvasCtx, WIDTH, HEIGHT);
          break;
        case 'bars':
          drawBars(canvasCtx, WIDTH, HEIGHT);
          break;
        case 'circles':
          drawCircles(canvasCtx, WIDTH, HEIGHT);
          break;
      }
    };

    const drawWaveform = (canvasCtx, WIDTH, HEIGHT) => {
      canvasCtx.lineWidth = lineWidth.value;
      canvasCtx.strokeStyle = strokeStyle.value;
      canvasCtx.beginPath();

      const sliceWidth = (WIDTH * 1.0) / dataArray.value.length;
      let x = 0;

      for (let i = 0; i < dataArray.value.length; i++) {
        const v = dataArray.value[i] / 128.0;
        const y = (v * HEIGHT) / 2;

        if (i === 0) {
          canvasCtx.moveTo(x, y);
        } else {
          canvasCtx.lineTo(x, y);
        }

        x += sliceWidth;
      }

      canvasCtx.lineTo(canvas.value.width, canvas.value.height / 2);
      canvasCtx.stroke();
    };

    const drawBars = (canvasCtx, WIDTH, HEIGHT) => {
      const barWidth = (WIDTH / dataArray.value.length) * 2.5;
      let barHeight;
      let x = 0;

      for (let i = 0; i < dataArray.value.length; i++) {
        barHeight = dataArray.value[i] / 2;

        canvasCtx.fillStyle = strokeStyle.value;
        canvasCtx.fillRect(x, HEIGHT - barHeight / 2, barWidth, barHeight);

        x += barWidth + 1;
      }
    };

    const drawCircles = (canvasCtx, WIDTH, HEIGHT) => {
      const radius = (HEIGHT / 2) * (dataArray.value[0] / 128.0);

      canvasCtx.beginPath();
      canvasCtx.arc(WIDTH / 2, HEIGHT / 2, radius, 0, 2 * Math.PI, false);
      canvasCtx.fillStyle = strokeStyle.value;
      canvasCtx.fill();
    };

    onMounted(() => {
      canvas.value.width = window.innerWidth;
      canvas.value.height = window.innerHeight;
    });

    return {
      canvas,
      handleFileUpload,
      togglePlayback,
      isPlaying,
      audioBuffer,
      visualizationType,
      lineWidth,
      strokeStyle,
    };
  },
};
</script>

<style scoped>
.visualizer {
  text-align: center;
  margin-top: 50px;
}

label {
  display: block;
}

canvas {
  display: block;
  margin: 0 auto;
  border: none;
}

.controls {
  display: flex;
  justify-content: center;
  padding-top: 20px;
}

input {
  margin-right: 20px;
}
</style>
