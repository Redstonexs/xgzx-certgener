<template>
  <div class="certificate-container">
    <!-- 用户输入 -->
    <div class="inputs">
      <label>姓名：<input v-model="name" placeholder="请输入姓名" /></label>
      <!-- <label>部门：<input v-model="department" placeholder="请输入部门" /></label> -->
    </div>

    <!-- Canvas -->
    <div class="canvas-wrapper">
      <canvas ref="canvas"></canvas>
    </div>

    <!-- 下载按钮 -->
    <button @click="downloadCertificate">下载证书</button>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue'
import certBg from '../assets/cert.jpg' // ✅ 正确引入图片路径

const name = ref('')
const department = ref('')
const canvas = ref(null)
let ctx = null
let bgImage = new Image()
let canvasWidth = 0
let canvasHeight = 0

// 绘制证书函数
const drawCanvas = () => {
  if (!ctx || !bgImage.complete) return
  ctx.clearRect(0, 0, canvasWidth, canvasHeight)
  ctx.drawImage(bgImage, 0, 0, canvasWidth, canvasHeight)

  // 绘制文字
  ctx.fillStyle = '#000'
  ctx.textAlign = 'center'
  ctx.font = 'bold 36px KaiTi'

  // 姓名
  ctx.fillText(name.value || '', canvasWidth / 5.76, 1125)
  // 部门
  ctx.font = '28px KaiTi'
  ctx.fillText(department.value || '', canvasWidth / 2, 460)
}

// 页面加载时执行
onMounted(() => {
  const c = canvas.value
  ctx = c.getContext('2d')

  bgImage.src = certBg
  bgImage.onload = () => {
    canvasWidth = bgImage.width
    canvasHeight = bgImage.height
    c.width = canvasWidth
    c.height = canvasHeight
    drawCanvas()
  }
})

// 实时更新
watch([name, department], drawCanvas)

// 下载功能
const downloadCertificate = () => {
  const link = document.createElement('a')
  link.download = 'certificate.png'
  link.href = canvas.value.toDataURL('image/png')
  link.click()
}
</script>

<style scoped>
.certificate-container {
  text-align: center;
  padding: 20px;
  background: rgba(255, 255, 255, 0.85);
  border-radius: 16px;
  box-shadow: 0 4px 25px rgba(0, 0, 0, 0.2);
  max-width: 1000px;
  margin: 30px auto;
  backdrop-filter: blur(6px);
  animation: fadeIn 1s ease;
}

.inputs {
  margin-bottom: 15px;
}

.inputs label {
  margin: 0 15px;
  font-size: 18px;
}

.inputs input {
  padding: 5px 10px;
  border-radius: 6px;
  border: 1px solid #aaa;
  font-size: 16px;
  outline: none;
}

.canvas-wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
}

canvas {
  max-width: 90%;
  height: auto;
  border-radius: 12px;
  box-shadow: 0 0 20px rgba(0, 0, 0, 0.25);
  transition: transform 0.4s ease;
}

canvas:hover {
  transform: scale(1.02);
}

button {
  margin-top: 15px;
  padding: 10px 25px;
  font-size: 18px;
  color: white;
  background: linear-gradient(135deg, #007bff, #4bc0c8);
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.3s, transform 0.3s;
}

button:hover {
  background: linear-gradient(135deg, #0056b3, #2a9d8f);
  transform: scale(1.05);
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(15px); }
  to { opacity: 1; transform: translateY(0); }
}
</style>
