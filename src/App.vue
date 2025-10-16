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
  minHeight: '100vh',
  width: '100%',
  display: 'flex',
  justifyContent: 'center',
  alignItems: 'flex-start', // 改为 flex-start 避免居中压缩
  padding: '20px 0' // 添加上下内边距
}
</script>

<!-- 不要用 scoped，这里希望背景样式能全局生效 -->
<style>
* {
  box-sizing: border-box;
}

html, body, #app {
  margin: 0;
  padding: 0;
  height: 100%;
  width: 100%;
}

#app {
  display: flex;
  justify-content: center;
  align-items: flex-start; /* 改为顶部对齐 */
  overflow-y: auto; /* 允许垂直滚动 */
}
</style>