<script setup>
import Header from './components/Header.vue'
import ImageCompression from './components/ImageCompression.vue'
</script>

<template>
  <div class="app">
    <Header />
    <ImageCompression />
  </div>
</template>

<style scoped>
.app {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

main {
  flex-grow: 1;
  padding: 1rem;
}
</style>