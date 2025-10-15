<template>
  <div class="panel">
    <div class="controls">
      <label>
        姓名：
        <!-- 新增 input/blur/enter 事件 -->
        <input
          v-model="name"
          placeholder="请输入姓名"
          @input="onInput"
          @blur="onBlur"
          @keyup.enter="onEnter"
        />
      </label>

      <button @click="downloadCertificate">下载证书</button>
    </div>

    <div ref="wrapperRef" class="canvas-wrapper">
      <canvas ref="canvasRef" :style="{ transform: canvasTransform }"></canvas>
    </div>

    <!-- 不在组织内提示模态 -->
    <div v-if="showNotInList" class="modal-mask" @click.self="closeModal">
      <div class="modal-box">
        <h3>提示</h3>
        <p>您输入的姓名 <strong>{{ nameDisplay }}</strong> 不在组织名单内。</p>
        <p>若有误请检查姓名拼写或联系管理员登记。</p>
        <div class="modal-actions">
          <button @click="closeModal">关闭</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, watch } from 'vue'
// 模板图片路径（Vite 推荐写法）
const certUrl = new URL('../assets/cert.jpg', import.meta.url).href

// -------------------- 白名单（示例，替换为真实名单或改为异步请求） --------------------
//其实这样校验非常不安全（还暴露隐私）
//but，我懒得写后端了，整出来还不一定稳定
const allowedNames = [
  '马雪瑞',
  '胡微',
  '袭荣双',
  '赵睿',
  '吕欣颖',
  '庞圣爱',
  '张馨',
  '郭超',
  '李昱浩',
  '常德润',
  '隋佳泽',
  '孟淳然',
  '张钰彤',
  '姜思莹',
  '张恒龙',
  '付一诺',
  '刘宇航',
  '庄豪森',
  '王欣鹏',
  '蒋宇昕',
  '吕曼清',
  '徐暚',
  '曾荣兴',
  '胡乃凤',
  '洪钰舫',
  '金妍伊',
  '马钰琪',
  '王娜',
  '霍轶凡',
  '张一帆',
  '高睿涵',
  '柳迎会',
  '杨嘉琪',
  '陈倩娜',
  '吕文霞',
  '衣姿洁',
  '刘鑫媛',
  '黄修正',
  '吴承远',
  '奉家麒',
  '陈鹏宇',
  '巩浩然',
  '董家郡',
  '林泓锦',
  '林鸿铭',
  '刘家源',
  '李钦林',
  '聊世豪',
  '李翔',
  '卢政宇',
  '唐廷乐',
  '王怀鑫',
  '连珈弘',
  '薄翔泽',
  '孙楠',
  '佟昊阳',
  '刘啸阳',
  '任天骄',
  '周志昊',
  '刘乐妍',
  '王梓言',
  '王铭晟',
  '谷训行',
  '宫士杰',
  '冯伟恒',
  '张紫阳',
  '甄月圆',
  '宋燕红',
  '邵甜雨',
  '李鸿阳',
  '王策',
  '单英祺',
  '庞杰文',
  '李雨宸',
  '辛泽昊',
  '王炳智',
  '彭屹恒',
  '孙维璐',
  '梁盼盼',
  '刘晓畅',
  '王恺鑫',
  '宋亚璐',
  '宋佳茹',
  '杨浩茹',
  '杨在起',
  '宋欣然',
  '张晓晴',
  '马承宗',
  '谭皓澜',
  '袁玮婧',
  '汪乐乐',
  '亓圣杰',
  '林妍如',
  '马佳慧',
  '沈传菊',
  '刘茹',
  '梁桐宇',
  '陈恬',
  '吴明达',
  '魏劭菲',
  '文茜茜',
  '韦颖希',
  '张博轩',
  '买地努尔·艾买尔',
  '杨芸',
  '李姿莹',
  '田雨',
  '谢雨晗',
  '李泰瑞',
  '杨发元',
  '周晨欣',
  '冉金江',
  '范奕楠',
  '焦润梓',
  '黄伟伟',
  '刘承鑫',
  '周晴晴'
].map(n => n.trim().toLowerCase())
// -------------------------------------------------------------------------

// refs / state
const name = ref('')
const department = ref('')
const canvasRef = ref(null)
const wrapperRef = ref(null)
let ctx = null
let img = null
let naturalW = 0
let naturalH = 0
const canvasTransform = ref('scale(1)')

// modal 控制
const showNotInList = ref(false)
const nameDisplay = ref('')

// 防抖 timer
let debounceTimer = null
const DEBOUNCE_MS = 800

// 判断姓名是否在名单内
function isNameAllowed(rawName) {
  if (!rawName) return true
  const normalized = rawName.trim().toLowerCase()
  return allowedNames.includes(normalized)
}

function fitFontSize(text, maxWidth, initialPx, minPx, fontFamily) {
  // 返回适合 maxWidth 的字号（px），不小于 minPx
  let px = initialPx;
  ctx.font = `${px}px ${fontFamily}`;
  let metrics = ctx.measureText(text);
  while (px > minPx && metrics.width > maxWidth) {
    px = Math.max(minPx, Math.floor(px * 0.92)); // 缓慢缩小（8%步长）
    ctx.font = `${px}px ${fontFamily}`;
    metrics = ctx.measureText(text);
  }
  return px;
}

function splitNameToLines(nameStr, maxCharsPerLine) {
  // 优先按空格/断词分行；否则按字符数折中分两行
  const s = nameStr.trim();
  if (!s) return [];

  // 如果有空格（拉丁名、带空格的中东名等），优先在空格处分割成两部分，尽量均衡字符数
  if (/\s/.test(s)) {
    const parts = s.split(/\s+/);
    // 寻找一个分割点使得左右长度接近
    let best = 1;
    for (let i = 1; i < parts.length; i++) {
      const left = parts.slice(0, i).join(' ');
      const right = parts.slice(i).join(' ');
      if (Math.abs(left.length - right.length) < Math.abs(parts.slice(0, best).join(' ').length - parts.slice(best).join(' ').length)) {
        best = i;
      }
    }
    return [parts.slice(0, best).join(' '), parts.slice(best).join(' ')];
  }

  // 无空格时（中文/无空格外文），按字符数折半
  const len = s.length;
  const mid = Math.ceil(len / 2);
  // 试着在 mid 前后找一个合理的断点（避免把姓/名截断太短）
  const left = s.slice(0, mid);
  const right = s.slice(mid);
  return [left, right];
}

// 渲染函数（内部像素空间）
function renderToCanvas() {
  if (!ctx || !img || !img.complete) return;
  // 清空并绘制背景模板
  ctx.clearRect(0, 0, naturalW, naturalH);
  ctx.drawImage(img, 0, 0, naturalW, naturalH);

  // 姓名处理
  const rawName = name.value ? name.value.trim() : '';
  if (!rawName) return; // 空名只绘背景
  const fullName = `${rawName} 同学`;
  // 配置：根据模板调整这些参数（可微调）
  const fontFamily = `"KaiTi","Kaiti SC"`;
  const initialFontPx = Math.round(naturalW / 32); // 初始字号（与你现在使用的类似）
  const minFontPx = Math.max(12, Math.round(naturalW / 60)); // 最小字号阈值（以宽度相关）
  const nameMaxWidth = naturalW * 0.6; // 姓名允许的最大宽度（占画布宽度的比率），可根据模板调整
  const centerX = Math.round(naturalW / 4.8);
  const baseY = Math.round(naturalH * 0.663); // 原来放姓名的中心Y

  // 1) 尝试单行：先用初始字号尝试 fit
  ctx.textAlign = 'center';
  ctx.fillStyle = '#000';

  let fontPx = fitFontSize(fullName, nameMaxWidth, initialFontPx, minFontPx, fontFamily);

  // 测试是否 fit（measureText）
  ctx.font = `normal ${fontPx}px ${fontFamily}`;
  const wSingle = ctx.measureText(fullName).width;

  if (wSingle <= nameMaxWidth) {
    // 单行可行 -> 直接绘制
    ctx.fillText(fullName, centerX, baseY);
    return;
  }

  // 2) 单行不可行 -> 尝试分两行
  const lines = splitNameToLines(fullName);
  // 二行都要 fit：为两行找合适字号（可同字号或分别调整）
  // 我们先尝试使用单行字体的一半略微放大比例
  // 先尝试相同字号（稍小）
  let lineFontPx = fitFontSize(lines[0], nameMaxWidth, Math.floor(fontPx * 0.9), minFontPx, fontFamily);
  // 对第二行也适配
  let line2FontPx = fitFontSize(lines[1], nameMaxWidth, lineFontPx, minFontPx, fontFamily);

  // 若第二行字号更小，取更小值保证两行一致视觉（可选）
  const finalFontPx = Math.min(lineFontPx, line2FontPx);

  // 如果最终字号仍小于最小阈值，考虑进一步处理：强制截断或水平缩放（这里选择截断并加省略）
  if (finalFontPx <= minFontPx) {
    // 截断第二行或第一行以防覆盖（简单策略：在宽度允许内截断并加省略）
    ctx.font = `normal ${minFontPx}px ${fontFamily}`;
    function ellipsize(text, maxW) {
      let t = text;
      while (t.length > 0 && ctx.measureText(t + '…').width > maxW) {
        t = t.slice(0, -1);
      }
      return t + (t.length < text.length ? '…' : '');
    }
    const l1 = ellipsize(lines[0], nameMaxWidth);
    const l2 = ellipsize(lines[1], nameMaxWidth);
    const lineSpacing = Math.round(minFontPx * 1.35);
    ctx.fillText(l1, centerX, baseY - lineSpacing / 2);
    ctx.fillText(l2, centerX, baseY + lineSpacing / 2);
    return;
  }

  // 绘制两行：行间距按字号比例
  ctx.font = `normal ${finalFontPx}px ${fontFamily}`;
  const lineSpacing = Math.round(finalFontPx * 1.4);
  ctx.fillText(lines[0], centerX, baseY - lineSpacing / 2);
  ctx.fillText(lines[1], centerX, baseY + lineSpacing / 2);
}

/** 在用户“完成输入”时调用：校验名单并绘制或弹模态 */
function validateAndRender() {
  // 关闭任何已有模态（如果用户继续输入，则应先隐藏）
  showNotInList.value = false
  nameDisplay.value = ''

  const enteredName = name.value ? name.value.trim() : ''
  if (!enteredName) {
    // 若为空则仅绘制模板（不显示提示）
    renderToCanvas()
    return
  }

  const allowed = isNameAllowed(enteredName)
  if (!allowed) {
    // 若不在名单：显示模态并只绘制模板（不绘姓名）
    nameDisplay.value = enteredName
    showNotInList.value = true
    // 仍先绘制背景模板
    if (ctx && img && img.complete) {
      ctx.clearRect(0, 0, naturalW, naturalH)
      ctx.drawImage(img, 0, 0, naturalW, naturalH)
    }
    return
  }

  // 在名单内：绘制姓名
  renderToCanvas()
}

// input 事件处理：防抖，且输入时隐藏已存在提示
function onInput() {
  // 每次输入时隐藏已弹的提示（用户还在改）
  if (showNotInList.value) {
    showNotInList.value = false
    nameDisplay.value = ''
  }
  // 重置防抖计时器
  if (debounceTimer) clearTimeout(debounceTimer)
  debounceTimer = setTimeout(() => {
    validateAndRender()
    debounceTimer = null
  }, DEBOUNCE_MS)
}

// blur / enter 事件：立即校验（用户认为已经完成）
function onBlur() {
  if (debounceTimer) {
    clearTimeout(debounceTimer)
    debounceTimer = null
  }
  validateAndRender()
}
function onEnter() {
  if (debounceTimer) {
    clearTimeout(debounceTimer)
    debounceTimer = null
  }
  validateAndRender()
}

// 缩放与下载相关（保持原有逻辑）
function updateScale() {
  const wrapper = wrapperRef.value
  const canvas = canvasRef.value
  if (!wrapper || !canvas || !naturalW || !naturalH) return

  const wrapperW = wrapper.clientWidth
  const wrapperH = wrapper.clientHeight
  const scale = Math.min(wrapperW / naturalW, wrapperH / naturalH, 1)
  canvasTransform.value = `scale(${scale})`
}

// 将下面函数放到你的 <script setup> 里，替换现有下载逻辑。
// 依赖：canvasRef（ref to canvas DOM）和 name（ref 存储姓名）
async function downloadCertificate() {
  const canvas = canvasRef.value;
  if (!canvas) return;

  // 构建要用的中文文件名（保留中文），并做最小清理但不替换中文
  const rawName = (name.value || '').trim() || '证书';
  // 限制长度，避免过长带来的问题（保留中文）
  const maxLen = 40;
  const safeBase = rawName.length > maxLen ? rawName.slice(0, maxLen) : rawName;
  const filename = `${safeBase}.png`;

  // Helper: toBlob -> Promise
  const blob = await new Promise(resolve => {
    canvas.toBlob(b => resolve(b), 'image/png');
  });
  if (!blob) {
    console.error('生成图片失败');
    return;
  }

  // 首选：使用 Web Share API 分享文件（在许多移动端浏览器可保留中文名）
  // try {
  //   const file = new File([blob], filename, { type: blob.type });

  //   // 检查能否分享文件（某些浏览器只支持 share text/url）
  //   if (navigator.canShare && navigator.canShare({ files: [file] })) {
  //     try {
  //       await navigator.share({ files: [file], title: filename });
  //       // 成功或用户取消都会返回到这里
  //       return;
  //     } catch (err) {
  //       // 可能用户取消或 share 失败，继续尝试常规下载
  //       console.warn('Web Share 失败或被取消：', err);
  //     }
  //   }
  // } catch (err) {
  //   // 某些环境创建 File 可能抛错，继续后续回退
  //   console.warn('构建 File 或分享检测失败：', err);
  // }

  // 次选：使用 createObjectURL 与 <a download="中文名"> 触发（多数 Android/Chrome 支持中文名）
  try {
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');

    // 设置 download 属性为中文名（很多浏览器会保留中文）
    a.href = url;
    a.download = filename;

    // 某些浏览器需要把节点插入 DOM 才能触发下载
    document.body.appendChild(a);
    a.click();
    a.remove();

    // 释放资源
    setTimeout(() => URL.revokeObjectURL(url), 1500);
    return;
  } catch (err) {
    console.warn('objectURL 下载失败，准备回退：', err);
  }

  // 最后回退：在新标签打开 Blob URL（用户可长按或保存图片），并提示用户如何保存并重命名
  try {
    const url2 = URL.createObjectURL(blob);
    window.open(url2, '_blank');
    // 可选：显示一个提示，告诉用户“长按图片 -> 保存到相册，然后可在文件或相册中重命名为：<中文名>”
    setTimeout(() => URL.revokeObjectURL(url2), 5000);
    return;
  } catch (err) {
    console.error('最终回退也失败：', err);
  }
}


// ResizeObserver 支持
let ro = null
function setupResizeObservers() {
  window.addEventListener('resize', updateScale)
  if (window.ResizeObserver) {
    ro = new ResizeObserver(updateScale)
    if (wrapperRef.value) ro.observe(wrapperRef.value)
  }
}
function cleanupResizeObservers() {
  window.removeEventListener('resize', updateScale)
  if (ro && wrapperRef.value) ro.disconnect()
  ro = null
}

// 挂载时加载图片、初始化 canvas
onMounted(() => {
  const canvas = canvasRef.value
  if (!canvas) return
  ctx = canvas.getContext('2d')

  img = new Image()
  img.crossOrigin = 'anonymous'
  img.src = certUrl
  img.onload = () => {
    naturalW = img.naturalWidth || img.width
    naturalH = img.naturalHeight || img.height

    canvas.width = naturalW
    canvas.height = naturalH

    canvas.style.transformOrigin = 'top center'
    canvas.style.display = 'block'

    // 先绘模板
    ctx.clearRect(0, 0, naturalW, naturalH)
    ctx.drawImage(img, 0, 0, naturalW, naturalH)

    updateScale()
    setupResizeObservers()
  }
  img.onerror = (e) => {
    console.error('证书模板加载失败：', e)
  }
})

onBeforeUnmount(() => {
  if (debounceTimer) clearTimeout(debounceTimer)
  cleanupResizeObservers()
})

// 关闭模态
function closeModal() {
  showNotInList.value = false
  nameDisplay.value = ''
}
</script>

<style scoped>
.panel {
  width: 100%;
  max-width: 980px;
  margin: 24px;
  display: flex;
  flex-direction: column;
  align-items: center;
}

/* 控件区 */
.controls {
  width: 100%;
  display: flex;
  gap: 12px;
  align-items: center;
  justify-content: center;
  margin-bottom: 12px;
  flex-wrap: wrap;
}
.controls label {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 14px;
}
.controls input {
  padding: 6px 10px;
  border-radius: 6px;
  border: 1px solid #bbb;
  font-size: 14px;
}

/* 限制 wrapper 的高度（视口内完整显示） */
.canvas-wrapper {
  width: 100%;
  height: calc(72vh);
  display: flex;
  justify-content: center;
  align-items: flex-start;
  overflow: hidden;
  background: rgba(255,255,255,0.02);
  border-radius: 10px;
  padding-top: 12px;
}

canvas {
  display: block;
  transform-origin: top center;
  image-rendering: optimizeQuality;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  box-shadow: 0 8px 28px rgba(0,0,0,0.18);
  border-radius: 8px;
}

button {
  padding: 8px 14px;
  margin-top: 12px;
  border: none;
  border-radius: 8px;
  background: linear-gradient(135deg,#007bff,#4bc0c8);
  color: #fff;
  cursor: pointer;
}
button:hover { transform: translateY(-2px); }

/* 模态样式 */
.modal-mask {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.45);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 9999;
}
.modal-box {
  background: #fff;
  padding: 18px 20px;
  border-radius: 10px;
  width: min(420px, 90%);
  box-shadow: 0 8px 30px rgba(0,0,0,0.25);
  text-align: center;
}
.modal-box h3 { margin: 0 0 8px 0; }
.modal-box p { margin: 6px 0; color: #333; }
.modal-actions { margin-top: 12px; }
.modal-actions button {
  padding: 8px 14px;
  border-radius: 8px;
  background: #007bff;
  color: #fff;
  border: none;
  cursor: pointer;
}

/* 响应式微调 */
@media (max-width: 720px) {
  .panel { max-width: 95%; margin: 12px; }
  .canvas-wrapper { height: calc(68vh); }
  .controls input { width: 140px; }
}
</style>
