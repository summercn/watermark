<template>
  <div id="app">
    <water-mark />
  </div>
</template>

<script>
import WaterMark from './components/WaterMark.vue'

export default {
  name: 'app',
  components: {
    WaterMark
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
