<template>
  <div id="app">
    <red-packet-rain></red-packet-rain>
  </div>
</template>

<script>
import redPacketRain from './views/redPacketRain.vue'

export default {
  name: 'App',
  components: {
    redPacketRain
  }
}
</script>

<style>
@import "./style/reset.css";
@import "./style/main.css";
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  width: 100vw;
  height: 100vh;
  background: rgb(2, 0, 36);
  background: linear-gradient(
    0deg,
    rgba(2, 0, 36, 1) 0%,
    rgba(58, 58, 99, 1) 50%,
    rgba(0, 127, 153, 1) 100%
  );
}
</style>
