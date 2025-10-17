<!-- src/App.vue -->
<template>
  <div id="app" :style="bgStyle" class="app-root">
    <!-- 顶部标题图片（外置于证书生成器白框之外） -->
    <header class="app-header" aria-hidden="false" role="banner">
      <img :src="bannerUrl" alt="学工在线 精彩无限" class="title-banner" />
    </header>

    <!-- 证书生成器（保持原状） -->
    <main class="app-main">
      <CertificateGenerator />
    </main>
  </div>
</template>

<script setup>
import { ref } from 'vue'                       // 必须导入 ref
import CertificateGenerator from './components/CertificateGenerator.vue'
import pageBg from './assets/bg2.jpg'
import bannerImg from './assets/title.png'

// 简单背景样式（保持你原来想要的 cover 行为）
const bgStyle = {
  backgroundImage: `url(${pageBg})`,
  backgroundSize: 'cover',
  backgroundPosition: 'center center',
  backgroundRepeat: 'no-repeat',
  minHeight: '120vh',
  width: '100%',
  display: 'flex',
  flexDirection: 'column',
  alignItems: 'center',
  padding: '18px 0'
}

const bannerUrl = bannerImg
const bannerRef = ref(null)
</script>

<style>
/* 基本重置及容器对齐（保持轻量） */
* { box-sizing: border-box; }
html, body, #app { height: 100%; margin: 0; padding: 0; }

/* 顶部标题：居中、保留一点底部间距 */
.app-header {
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 10px 12px 8px;
  pointer-events: none; /* 不需要交互 */
}

/* 标题图片：居中、响应式、不超过一定宽度 */
.title-banner {
  width: min(92%, 720px);
  max-width: 100%;
  height: auto;
  display: block;
  user-select: none;
  -webkit-user-drag: none;
  border-radius: 6px; /* 可去掉，如果你不想圆角 */
}

/* 主体保持默认（你的 CertificateGenerator 自己有白框样式） */
.app-main {
  width: 100%;
  display: flex;
  justify-content: center;
  padding: 8px 12px 40px;
  box-sizing: border-box;
}

/* 移动端微调 */
@media (max-width: 720px) {
  .title-banner { width: min(94%, 640px); border-radius: 4px; }
  .app-main { padding: 6px 8px 28px; }
}
</style>
