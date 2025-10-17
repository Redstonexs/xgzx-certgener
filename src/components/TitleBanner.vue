<template>
  <figure class="title-banner" role="img" aria-label="学工在线 精彩无限">
    <img
      :src="src"
      :alt="alt"
      class="title-img"
      loading="lazy"
      @load="onLoad"
    />
  </figure>
</template>

<script setup>
import { ref } from 'vue'

// Props：可以在父组件传入不同图片或标题文本
const props = defineProps({
  src: { type: String, default: new URL('../assets/title.png', import.meta.url).href },
  alt: { type: String, default: '学工在线 精彩无限' },
  maxWidth: { type: [String, Number], default: 760 } // px 或 css 值
})

const loaded = ref(false)
function onLoad() {
  loaded.value = true
}
</script>

<style scoped>
.title-banner {
  width: 100%;
  display: flex;
  justify-content: center;
  margin-bottom: 12px;
  pointer-events: none; /* 如果不需要交互 */
}
.title-img {
  width: min(90%, var(--tb-max-width, 760px));
  max-width: 100%;
  height: auto;
  display: block;
  opacity: 0;
  transform: translateY(-8px);
  transition: opacity 480ms ease, transform 480ms ease;
  user-select: none;
}

/* 当图片加载完毕（由 JS 添加类或使用 attribute），淡入 */
.title-img[loading="eager"], .title-img.loaded {
  opacity: 1;
  transform: translateY(0);
}
</style>
