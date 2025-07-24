<script setup>
import WeatherWidget from "./components/WeatherWidget.vue";
</script>

<template>
  <div class="container">
    <WeatherWidget />
  </div>
</template>

<style scoped>
.container {
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>
