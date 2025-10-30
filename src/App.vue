<!-- src/App.vue -->
<template>
  <div class="page">
    <!-- Fondo fijo y centrado -->
    <img class="bg" src="@/assets/Fondo1.jpg" alt="" />

    <!-- Scrim para mejorar contraste (podés ajustar la opacidad) -->
    <div class="scrim"></div>

    <AppNavbar />

    <!-- Contenido centrado verticalmente -->
    <main class="content">
      <AppConvertidor />
    </main>
  </div>
</template>

<script>
import AppNavbar from './components/AppNavbar.vue'
import AppConvertidor from './components/AppConvertidor.vue'

export default {
  name: 'App',
  components: { AppNavbar, AppConvertidor }
}
</script>

<style scoped>
/* Ajustá si tu navbar es más alta */
.page { 
  --nav-h: 64px;                 /* altura aprox. de la navbar */
  min-height: 100vh; 
  position: relative;
}

/* Fondo fijo, cubre toda la pantalla, centrado */
.bg {
  position: fixed;
  inset: 0;                      /* top/right/bottom/left = 0 */
  width: 100%;
  height: 100%;
  object-fit: cover;             /* llena sin deformar */
  object-position: center center;
  z-index: -2;
}

/* Capa de brillo/sombra (ajustá el color/alpha a gusto) */
.scrim {
  position: fixed;
  inset: 0;
  background: rgba(255, 255, 255, 0.6); /* o rgba(0,0,0,0.35) si querés oscurecer */
  backdrop-filter: blur(2px);
  z-index: -1;
}

/* Centrado vertical/horizontal del convertidor, dejando lugar a la navbar */
.content {
  min-height: calc(100vh - var(--nav-h));
  display: flex;
  align-items: center;
  justify-content: center;
  padding-top: 0.5rem;
}
</style>
