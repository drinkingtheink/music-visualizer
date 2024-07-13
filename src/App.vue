<template>
  <div id="app">
    <AudioVisualizer />
  </div>
</template>

<script>
import AudioVisualizer from './components/AudioVisualizer.vue';

export default {
  name: 'App',
  components: {
    AudioVisualizer,
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
