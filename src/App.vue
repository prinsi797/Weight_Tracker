<template>
  <div id="app">
    <WeightTracker />
  </div>
</template>

<script>
import WeightTracker from './components/WeightTracker.vue'

export default {
  name: 'App',
  components: {
    WeightTracker
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 20px;
}
</style>