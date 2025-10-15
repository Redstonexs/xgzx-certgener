<template>
  <div id="app" :style="{ backgroundImage: `url(${bg})` }">
    <CertificateGenerator />
  </div>
</template>

<script setup>
import CertificateGenerator from './components/CertificateGenerator.vue'
import bg from './assets/bg2.jpg'
</script>

<style>
html,
body,
#app {
  margin: 0;
  padding: 0;
  height: 100%;
  width: 100%;
}

/* ✅ 设置背景图完全覆盖页面，不留白边 */
#app {
  background-repeat: no-repeat;
  background-position: center center;
  background-size: cover; /* 关键：自动等比放大覆盖页面 */
  background-attachment: fixed; /* 页面滚动时背景不动，可去掉 */
  display: flex;
  justify-content: center;
  align-items: center;
}

/* 可选：加个柔和遮罩让内容更清晰 */
#app::before {
  content: "";
  position: absolute;
  inset: 0;
  background: rgba(255, 255, 255, 0.3); /* 半透明遮罩 */
  z-index: 0;
}

.certificate-container {
  position: relative;
  z-index: 1;
}
</style>