<template>
  <div class="page-container">
    <div class="card" v-if="!generated">
      <h2>🎓 在线证书生成器</h2>
      <div class="form-group">
        <label>姓名：</label>
        <input v-model="name" placeholder="请输入你的姓名" />
      </div>
      <div class="form-group">
        <label>部门：</label>
        <input v-model="department" placeholder="请输入你的部门" />
      </div>
      <button class="generate-btn" @click="generateCertificate">生成证书</button>
    </div>

    <transition name="fade-zoom">
      <div v-if="generated" class="result-card">
        <h2>🎉 证书生成成功！</h2>
        <canvas ref="canvas" width="800" height="600"></canvas>
        <button class="download-btn" @click="downloadImage">下载证书</button>
        <button class="back-btn" @click="reset">返回修改</button>
      </div>
    </transition>
  </div>
</template>

<script setup>
import { ref } from "vue"

const name = ref("")
const department = ref("")
const generated = ref(false)
const canvas = ref(null)

function generateCertificate() {
  if (!name.value || !department.value) {
    alert("请填写完整信息")
    return
  }

  generated.value = true
  setTimeout(drawCertificate, 200)
}

function drawCertificate() {
  const ctx = canvas.value.getContext("2d")
  const img = new Image()
  img.src = "/certificate-placeholder.jpg" // 背景占位图，可换成你的证书模板
  img.onload = () => {
    ctx.drawImage(img, 0, 0, 800, 600)
    ctx.font = "bold 36px Microsoft YaHei"
    ctx.fillStyle = "#333"
    ctx.textAlign = "center"
    ctx.fillText(name.value, 400, 350)
    ctx.font = "28px Microsoft YaHei"
    ctx.fillText(department.value, 400, 420)
  }
}

function downloadImage() {
  const link = document.createElement("a")
  link.download = `${name.value}-证书.png`
  link.href = canvas.value.toDataURL("image/png")
  link.click()
}

function reset() {
  generated.value = false
}
</script>

<style scoped>
.page-container {
  background: url('/bg-pattern.jpg') center/cover no-repeat, linear-gradient(135deg, #dfe9f3 0%, #ffffff 100%);
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  animation: fadeIn 1.2s ease;
}

.card, .result-card {
  background: rgba(255, 255, 255, 0.85);
  border-radius: 16px;
  box-shadow: 0 8px 25px rgba(0, 0, 0, 0.1);
  padding: 40px 50px;
  text-align: center;
  transition: all 0.3s ease;
  width: 90%;
  max-width: 600px;
}

h2 {
  margin-bottom: 20px;
  color: #333;
}

.form-group {
  margin: 15px 0;
  text-align: left;
}

label {
  display: block;
  font-weight: 600;
  margin-bottom: 6px;
}

input {
  width: 100%;
  padding: 10px;
  border-radius: 8px;
  border: 1px solid #ccc;
  outline: none;
  transition: all 0.3s;
}
input:focus {
  border-color: #007bff;
  box-shadow: 0 0 6px rgba(0, 123, 255, 0.3);
}

.generate-btn, .download-btn, .back-btn {
  margin-top: 20px;
  padding: 12px 28px;
  border: none;
  border-radius: 10px;
  font-size: 16px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.generate-btn {
  background: #007bff;
  color: #fff;
}
.generate-btn:hover {
  background: #0056b3;
}

.download-btn {
  background: #28a745;
  color: white;
  animation: pulse 2s infinite;
}
.back-btn {
  margin-left: 10px;
  background: #ffc107;
  color: #333;
}
.back-btn:hover {
  background: #e0a800;
}

canvas {
  margin-top: 15px;
  border-radius: 12px;
  box-shadow: 0 0 10px rgba(0,0,0,0.15);
}

/* 动画 */
.fade-zoom-enter-active {
  animation: zoomIn 0.8s ease;
}
.fade-zoom-leave-active {
  animation: fadeOut 0.6s ease forwards;
}

@keyframes zoomIn {
  0% { opacity: 0; transform: scale(0.9); }
  100% { opacity: 1; transform: scale(1); }
}

@keyframes fadeOut {
  0% { opacity: 1; transform: scale(1); }
  100% { opacity: 0; transform: scale(0.95); }
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes pulse {
  0%, 100% { transform: scale(1); box-shadow: 0 0 0 rgba(40, 167, 69, 0.6); }
  50% { transform: scale(1.05); box-shadow: 0 0 20px rgba(40, 167, 69, 0.4); }
}
</style>
