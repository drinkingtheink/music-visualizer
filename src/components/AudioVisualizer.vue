<template>
  <div class="visualizer">
    <header :style="{ borderColor: strokeStyle }">
      <h1>Pixelphonic ⚡</h1>

      <section class="controls">
        <label>Play Samples:</label>
        <div v-for="sample in sampleFiles" :key="sample.name">
          <button @click="handleSampleSelect(sample.url)">{{ sample.name }}</button>
        </div>
      </section>

      <section class="controls">
        <label for="fileUpload">Upload a File:</label>
        <input id="fileUpload" type="file" @change="handleFileUpload" />
      </section>

      <section class="controls">
        <button class="play" @click="togglePlayback" :disabled="!audioBuffer">
          {{ isPlaying ? 'Pause' : 'Play' }}
        </button>
        <button @click="clearCanvas" :disabled="!audioBuffer">Clear Canvas</button>
      </section>
    </header>

    <header class="line-controls" :style="{ borderColor: strokeStyle }">
      <section class="controls">
        <label for="lineWidth">Foreground Line Width:</label>
        <input type="number" v-model="lineWidth" min="1" max="30" />
        <label for="strokeStyle">Foreground Line Color:</label>
        <input type="color" v-model="strokeStyle" />
        <button class="random" @click="randomizeColor(true)">Random</button>
      </section>
      <section class="controls">
        <label for="line2Width">Background Line Width:</label>
        <input type="number" v-model="line2Width" min="1" max="90" />
        <label for="stroke2Style">Background Line Color:</label>
        <input type="color" v-model="stroke2Style" />
        <button class="random" @click="randomizeColor(false)">Random</button>
      </section>
      <section class="controls links">
        <a href="https://github.com/drinkingtheink/music-visualizer">About This App</a>
        <a href="https://www.drinkingtheink.com/">About The Author</a>
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
import sample1 from '../assets/audio/pixelphonic-audio-sample-1.mp3';
import sample2 from '../assets/audio/pixelphonic-audio-sample-2.mp3';

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

    const sampleFiles = ref([
      { name: 'Sample 1', url: sample1 },
      { name: 'Sample 2', url: sample2 },
    ]);

    const handleSampleSelect = (url) => {
      if (!audioContext.value) {
        audioContext.value = new (window.AudioContext || window.webkitAudioContext)();
      }
      fetch(url)
        .then(response => response.arrayBuffer())
        .then(arrayBuffer => audioContext.value.decodeAudioData(arrayBuffer))
        .then(buffer => {
          audioBuffer.value = buffer;
          playAudio();
        });
    };

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

    const randomizeColor = (changeColor1) => {
      if (changeColor1) strokeStyle.value = chroma.random();
      else stroke2Style.value = chroma.random();
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
      randomizeColor,
      handleSampleSelect,
      sampleFiles,
    };
  },
};
</script>

<style  >
@import url('https://fonts.googleapis.com/css2?family=Playwrite+SK:wght@100..400&display=swap');

:root {
  --panelBg: rgba(0,0,0,0.7);
}

.visualizer {
  text-align: center;
  margin-top: 0;
}

h1 {
  font-family: "Playwrite SK", cursive;
  font-optical-sizing: auto;
  font-weight: 400;
  font-style: normal;
  padding: 0 0 0.5rem 0;
  margin: 0 0 0.5rem 0;
  border-bottom: 1px solid rgba(255,255,255,0.3);
}

label {
  display: block;
  margin-right: 20px;
}

.canvas-stage {
  position: relative;
  padding-top: 10rem;
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
  padding: 0 10px 10px 10px;
  background-color: var(--panelBg);
  max-width: 500px;
  margin: 0 0 0 1rem;
  border-radius: 0 0 20px 20px;
  border-top: 6px solid;
  position: fixed;
  top: 1rem;
  z-index: 100;
  transition: all 1s;
}

header.line-controls {
  margin-left: 490px;
  border-radius: 0 0 20px 0
}

.controls {
  display: flex;
  justify-content: center;
  padding: 10px;
}

.controls.links {
  justify-content: space-evenly;
}

.controls.links a {
  color: rgba(255,255,255,0.7);
  display: block;
  border: 2px solid rgba(255,255,255,0.3);
  padding: 0.5rem 2rem;
  border-radius: 5px;
  text-decoration: none;
  font-size: 0.75rem;
}

.controls.links a:hover {
  color: rgba(255,255,255,1);
}

input {
  margin: 0 10px;
}

input[type="number"] {
  font-size: 1.25rem;
  border-radius: 5px;
  padding-left: 10px;
}

input[type="color"] {
  width: 150px;
  height: 40px;
}

button {
  text-transform: uppercase;
  padding: 10px 30px;
  border-radius: 10px;
  margin-right: 20px;
  transition: all .3s;
  font-weight: bold;
}

button.play {
  width: 200px;
}

button.random {
  padding: 5px 10px;
}

button:hover {
  transform: scale(1.1);
}

footer {
  background-color: var(--panelBg);
  padding: 10px;
  max-width: 400px;
  position: fixed;
  top: 1rem;
  z-index: 100;
  margin-left: 1000px;
}
</style>
