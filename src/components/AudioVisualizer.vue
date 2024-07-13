<template>
  <div class="visualizer">
    <header :style="{ borderColor: strokeStyle }">
      <input type="file" @change="handleFileUpload" />

      <section class="controls">
        <button class="play" @click="togglePlayback" :disabled="!audioBuffer">
          {{ isPlaying ? 'Pause' : 'Play' }}
        </button>
        <button @click="clearCanvas">Clear Visualization</button>
      </section>
      <section class="controls">
        <label for="lineWidth">Line 1 Width:</label>
        <input type="number" v-model="lineWidth" min="1" max="30" />
        <label for="strokeStyle">Line 1 Color:</label>
        <input type="color" v-model="strokeStyle" />
      </section>
      <section class="controls">
        <label for="line2Width">Line 2 Width:</label>
        <input type="number" v-model="line2Width" min="1" max="90" />
        <label for="stroke2Style">Line 2 Color:</label>
        <input type="color" v-model="stroke2Style" />
      </section>
    </header>

    <section class="canvas-stage">
      <canvas ref="canvas" class="top-wave"></canvas>
      <canvas ref="canvas2" class="bottom-wave"></canvas>
    </section>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue';
import chroma from "chroma-js";

export default {
  setup() {
    const audioContext = ref(null);
    const canvas = ref(null);
    const canvas2 = ref(null);
    const audioBuffer = ref(null);
    const audioSource = ref(null);
    const analyser = ref(null);
    const dataArray = ref(null);
    const isPlaying = ref(false);
    const visualizationType = ref('waveform');
    const lineWidth = ref(6);
    const strokeStyle = ref('#000000');
    const line2Width = ref(1);
    const stroke2Style = ref('#ffffff');
    let startTime = 0;
    let pausedTime = 0;

    const handleFileUpload = (event) => {
      audioContext.value = new (window.AudioContext || window.webkitAudioContext)();
      const file = event.target.files[0];
      if (file && (file.type === 'audio/mpeg' || file.type === 'audio/mp3')) {
        const reader = new FileReader();
        reader.onload = (e) => {
          const arrayBuffer = e.target.result;
          audioContext.value.decodeAudioData(arrayBuffer, (buffer) => {
            audioBuffer.value = buffer;
          });
        };
        reader.readAsArrayBuffer(file);
      }
    };

    const playAudio = (resume = false) => {
      if (!audioBuffer.value) return;
      if (audioSource.value) audioSource.value.stop();

      audioSource.value = audioContext.value.createBufferSource();
      audioSource.value.buffer = audioBuffer.value;

      analyser.value = audioContext.value.createAnalyser();
      analyser.value.fftSize = 2048;
      const bufferLength = analyser.value.frequencyBinCount;
      dataArray.value = new Uint8Array(bufferLength);

      audioSource.value.connect(analyser.value);
      analyser.value.connect(audioContext.value.destination);

      if (resume) {
        startTime = audioContext.value.currentTime - pausedTime;
        audioSource.value.start(0, pausedTime);
      } else {
        startTime = audioContext.value.currentTime;
        audioSource.value.start();
      }
      isPlaying.value = true;
      drawVisualizer();
    };

    const pauseAudio = () => {
      if (!audioSource.value) return;
      audioSource.value.stop();
      pausedTime = audioContext.value.currentTime - startTime;
      isPlaying.value = false;
    };

    const togglePlayback = () => {
      if (isPlaying.value) {
        pauseAudio();
      } else {
        playAudio(pausedTime > 0);
      }
    };

    const clearCanvas = () => {
      const canvasCtx = canvas.value.getContext('2d');
      const canvas2Ctx = canvas2.value.getContext('2d');
      canvasCtx.clearRect(0, 0, canvas.value.width, canvas.value.height);
      canvas2Ctx.clearRect(0, 0, canvas2.value.width, canvas2.value.height);
    };

    const drawVisualizer = () => {
      if (!isPlaying.value) return;
      const canvasCtx = canvas.value.getContext('2d');
      const canvasCtx2 = canvas2.value.getContext('2d');
      const WIDTH = canvas.value.width;
      const HEIGHT = canvas.value.height;

      requestAnimationFrame(drawVisualizer);

      analyser.value.getByteTimeDomainData(dataArray.value);

      canvasCtx.clearRect(0, 0, WIDTH, HEIGHT);

      switch (visualizationType.value) {
        case 'waveform':
          drawWaveform(canvasCtx, WIDTH, HEIGHT);
          drawWaveform2(canvasCtx2, WIDTH, HEIGHT);
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

    const drawWaveform2 = (canvasCtx, WIDTH, HEIGHT) => {
      canvasCtx.lineWidth = line2Width.value;
      canvasCtx.strokeStyle = stroke2Style.value;
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

      canvasCtx.lineTo(canvas2.value.width, canvas2.value.height / 2);
      canvasCtx.stroke();
    };

    onMounted(() => {
      canvas.value.width = window.innerWidth;
      canvas.value.height = window.innerHeight;
      strokeStyle.value = chroma.random();
      canvas2.value.width = window.innerWidth;
      canvas2.value.height = window.innerHeight;
      stroke2Style.value = chroma.random();
    });

    return {
      canvas,
      canvas2,
      handleFileUpload,
      togglePlayback,
      isPlaying,
      audioBuffer,
      visualizationType,
      lineWidth,
      strokeStyle,
      line2Width,
      stroke2Style,
      clearCanvas,
    };
  },
};
</script>

<style scoped>
.visualizer {
  text-align: center;
  margin-top: 0;
}

label {
  display: block;
}

.canvas-stage {
  position: relative;
  margin-top: -100px;
}

canvas {
  display: block;
  border: none;
  position: absolute;
  top: 0;
  width: 110vw;
  margin-left: -10px;
}

canvas.top-wave {
  z-index: 10;
}

canvas.bottom-wave {
  z-index: 1;
}

header {
  padding: 20px;
  background-color: rgba(0,0,0,0.7);
  max-width: 500px;
  margin: 1rem 0 0 1rem;
  border-radius: 0 0 20px 20px;
  border-top: 10px solid;
  position: relative;
  z-index: 100;
}

.controls {
  display: flex;
  justify-content: center;
  padding: 10px;
}

input {
  margin: 0 20px;
}

button {
  text-transform: uppercase;
  padding: 10px 30px;
  border-radius: 10px;
  margin-right: 20px;
  transition: all .3s;
}

button.play {
  width: 200px;
}

button:hover {
  transform: scale(1.1);
}
</style>
