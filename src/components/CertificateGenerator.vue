<!-- src/components/CertificateGenerator.vue -->
<template>
  <div class="generator">
    <!-- 输入区域 -->
    <div class="input-group">
      <label>姓名：</label>
      <input v-model="name" placeholder="请输入姓名" />
    </div>
    <div class="input-group">
      <label>部门：</label>
      <input v-model="department" placeholder="请输入部门" />
    </div>
    <button @click="generateCertificate">生成证书</button>

    <!-- Canvas 画布 -->
    <div class="canvas-container">
      <canvas ref="canvasRef"></canvas>
    </div>

    <!-- 下载按钮 -->
    <button @click="downloadCertificate" :disabled="!certificateGenerated">下载证书</button>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue';

export default {
  name: 'CertificateGenerator',
  setup() {
    const name = ref('');              // 存储用户输入的姓名
    const department = ref('');        // 存储用户输入的部门
    const canvasRef = ref(null);       // 引用 Canvas 元素
    const ctx = ref(null);            // Canvas 2D 上下文
    const certificateGenerated = ref(false); // 是否已生成证书

    // 在组件挂载后获取 Canvas 2D 上下文
    onMounted(() => {
      if (canvasRef.value) {
        ctx.value = canvasRef.value.getContext('2d');
      }
    });

    // 生成证书图像：绘制背景图和文字
    const generateCertificate = () => {
      if (!ctx.value) return;
      const canvas = canvasRef.value;
      const context = ctx.value;

      const img = new Image();
      img.src = '/certificate.png';  // 证书背景图，放在 public 目录
      img.onload = () => {
        // 设置 Canvas 尺寸为图像尺寸
        canvas.width = img.width;
        canvas.height = img.height;

        // 绘制背景图
        context.clearRect(0, 0, canvas.width, canvas.height);
        context.drawImage(img, 0, 0);

        // 设置文字样式并绘制姓名和部门
        context.fillStyle = '#000';           // 字体颜色：黑色
        context.font = '28px serif';          // 字体大小和样式
        context.textAlign = 'center';         // 文本居中对齐
        context.fillText(name.value, canvas.width / 2, canvas.height / 2);           // 绘制姓名
        context.fillText(department.value, canvas.width / 2, canvas.height / 2 + 40); // 绘制部门

        certificateGenerated.value = true;
      };
      img.onerror = () => {
        alert('证书背景图片加载失败！');
      };
    };

    // 下载证书：将 Canvas 转为图片并触发下载
    const downloadCertificate = () => {
      if (!canvasRef.value) return;
      const canvas = canvasRef.value;
      const dataURL = canvas.toDataURL('image/png');

      // 创建临时链接并点击实现下载
      const link = document.createElement('a');
      link.href = dataURL;
      link.download = `certificate_${name.value}.png`;
      link.click();
    };

    return {
      name,
      department,
      canvasRef,
      generateCertificate,
      downloadCertificate,
      certificateGenerated
    };
  }
};
</script>

<style scoped>
.generator {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
}
.input-group {
  display: flex;
  align-items: center;
  gap: 8px;
}
.canvas-container {
  margin: 20px 0;
  border: 2px solid #ccc;
}
canvas {
  max-width: 100%;
}
button {
  padding: 8px 16px;
  background-color: #409eff;
  color: #fff;
  border: none;
  cursor: pointer;
}
button:disabled {
  background-color: #aaa;
  cursor: not-allowed;
}
</style>
