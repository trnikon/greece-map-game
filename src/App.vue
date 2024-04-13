<script setup>
import { RouterLink, RouterView } from 'vue-router'
import HomeView from "@/views/HomeView.vue";
</script>

<template>

    <div class="wrapper">
      <HomeView />
    </div>

</template>

<style scoped>
.wrapper {
  display: flex;
  justify-content: center;
  height: 100vh;
  width: 100vw;
  background: #e2d1a0;
}
</style>
