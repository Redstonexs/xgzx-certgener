<template>
  <!-- 直接绑定背景图，避免 document.querySelector 操作 -->
  <div id="app" :style="bgStyle">
    <CertificateGenerator />
  </div>
</template>

<script setup>
import CertificateGenerator from './components/CertificateGenerator.vue'
import pageBg from './assets/bg2.jpg' // 确保文件存在于 src/assets

// 绑定背景样式（保证 cover，无白边）
const bgStyle = {
  backgroundImage: `url(${pageBg})`,
  backgroundSize: 'cover',
  backgroundPosition: 'center center',
  backgroundRepeat: 'no-repeat',
  minHeight: '100vh'
}
</script>

<!-- 不要用 scoped，这里希望背景样式能全局生效 -->
<style>
html, body, #app {
  margin: 0;
  padding: 0;
  height: 100%;
  width: 100%;
}
#app {
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>
