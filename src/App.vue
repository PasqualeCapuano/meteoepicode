<script>
export default {
  name: 'App'
}
</script>

<template>
  <v-app theme="light">
    <div class="banner-container">
      <img src="@/assets/banner.png" alt="Meteo Epicode Banner" class="banner-image"/>
    </div>
    <v-main>
      <router-view />
    </v-main>
  </v-app>
</template>

<style>
/* Optional: ensure the background is white */
.v-application {
  background: #ffffff !important;
}

.banner-container {
  width: 100vw;
  background: white;
  position: relative;
  left: 50%;
  right: 50%;
  margin-left: -50vw;
  margin-right: -50vw;
}

.banner-image {
  width: 100%;
  height: 350px;
  object-fit: cover;
  display: block;
}
</style>
