<template>
  <div class="panel">
    <div class="title-wrap" aria-hidden="false">
      <h1 class="title-line1">学工在线(易班)工作站</h1>
      <h2 class="title-line2">干事纪念卡</h2>
    </div>

    <div class="controls">
      <label>
        姓名：
        <input
          class="name-input"
          v-model="name"
          placeholder="请输入姓名"
          @input="onInput"
          @blur="onBlur"
          @keyup.enter="onEnter"
        />
      </label>

      <button @click="downloadCertificate">下载证书</button>
    </div>

    <!-- 简化结构 -->
    <div ref="wrapperRef" class="canvas-wrapper" v-if="showCertificate" aria-hidden="!showCertificate">
      <canvas 
        ref="canvasRef" 
        class="certificate-canvas"
        :class="{ animated: animateIn }"
      ></canvas>
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
import { ref, onMounted, onBeforeUnmount, watch, nextTick } from 'vue'

// 证书模板图片（Vite 推荐写法）
const certUrl = new URL('../assets/cert.jpg', import.meta.url).href

// 白名单（示例）——保留现有数组并全部小写化用于比对
const allowedNames = [
  '马雪瑞','胡微','袭荣双','赵睿','吕欣颖','庞圣爱','张馨','郭超','李昱浩','常德润',
  '隋佳泽','孟淳然','张钰彤','姜思莹','张恒龙','付一诺','刘宇航','庄豪森','王欣鹏','蒋宇昕',
  '吕曼清','徐暚','曾荣兴','胡乃凤','洪钰舫','金妍伊','马钰琪','王娜','霍轶凡','张一帆',
  '高睿涵','柳迎会','杨嘉琪','陈倩娜','吕文霞','衣姿洁','刘鑫媛','黄修正','吴承远','奉家麒',
  '陈鹏宇','巩浩然','董家郡','林泓锦','林鸿铭','刘家源','李钦林','聊世豪','李翔','卢政宇',
  '唐廷乐','王怀鑫','连珈弘','薄翔泽','孙楠','佟昊阳','刘啸阳','任天骄','周志昊','刘乐妍',
  '王梓言','王铭晟','谷训行','宫士杰','冯伟恒','张紫阳','甄月圆','宋燕红','邵甜雨','李鸿阳',
  '王策','单英祺','庞杰文','李雨宸','辛泽昊','王炳智','彭屹恒','孙维璐','梁盼盼','刘晓畅',
  '王恺鑫','宋亚璐','宋佳茹','杨浩茹','杨在起','宋欣然','张晓晴','马承宗','谭皓澜','袁玮婧',
  '汪乐乐','亓圣杰','林妍如','马佳慧','沈传菊','刘茹','梁桐宇','陈恬','吴明达','魏劭菲',
  '文茜茜','韦颖希','张博轩','买地努尔·艾买尔','杨芸','李姿莹','田雨','谢雨晗','李泰瑞','杨发元',
  '周晨欣','冉金江','范奕楠','焦润梓','黄伟伟','刘承鑫','周晴晴'
].map(n => n.trim().toLowerCase())

// refs / state
const name = ref('')
const canvasRef = ref(null)
const wrapperRef = ref(null)
let ctx = null
let img = null
let naturalW = 0
let naturalH = 0

// UI 状态
const showNotInList = ref(false)
const nameDisplay = ref('')
const showCertificate = ref(false)
const animateIn = ref(false)

// composition / 输入状态
let isComposing = false
let inputEl = null

// 防抖
let debounceTimer = null
const DEBOUNCE_MS = 800

// ResizeObserver holder
let ro = null

// 判断姓名是否在名单内
function isNameAllowed(rawName) {
  if (!rawName) return true
  const normalized = rawName.trim().toLowerCase()
  return allowedNames.includes(normalized)
}

/* 字体等待（带超时回退） */
function waitForFont(fontName, timeoutMs = 2000) {
  if (typeof document === 'undefined' || !document.fonts || !document.fonts.load) {
    console.warn('[font] document.fonts not supported; will use fallback fonts')
    return Promise.resolve(false)
  }
  const loadPromise = document.fonts.load(`12px "${fontName}"`).then(() => {
    return document.fonts.check(`12px "${fontName}"`)
  }).catch(e => {
    console.warn('[font] load error', e)
    return false
  })
  const timeoutPromise = new Promise((resolve) => setTimeout(() => resolve(false), timeoutMs))
  return Promise.race([loadPromise, timeoutPromise])
}

/* 自动缩放字体以适配最大宽度 */
function fitFontSize(text, maxWidth, initialPx, minPx, fontFamily) {
  let px = initialPx;
  ctx.font = `${px}px ${fontFamily}`;
  let metrics = ctx.measureText(text);
  while (px > minPx && metrics.width > maxWidth) {
    px = Math.max(minPx, Math.floor(px * 0.92));
    ctx.font = `${px}px ${fontFamily}`;
    metrics = ctx.measureText(text);
  }
  return px;
}

/* 分行逻辑 */
function splitNameToLines(nameStr) {
  const s = nameStr.trim();
  if (!s) return [];
  if (/\s/.test(s)) {
    const parts = s.split(/\s+/);
    let best = 1;
    for (let i = 1; i < parts.length; i++) {
      if (Math.abs(parts.slice(0,i).join(' ').length - parts.slice(i).join(' ').length) < Math.abs(parts.slice(0,best).join(' ').length - parts.slice(best).join(' ').length)) {
        best = i;
      }
    }
    return [parts.slice(0,best).join(' '), parts.slice(best).join(' ')];
  }
  const len = s.length;
  const mid = Math.ceil(len / 2);
  return [s.slice(0, mid), s.slice(mid)];
}

/* 强制重绘 Canvas */
function forceRepaintCanvas() {
  const c = canvasRef.value
  if (!c) return
  try {
    void document.body.offsetHeight
    c.style.display = 'none'
    void c.offsetWidth
    c.style.display = ''
  } catch (e) {}
}

/* 创建圆角路径并绘制图片 */
function drawRoundedImage(ctx, img, x, y, width, height, radius) {
  ctx.save();
  ctx.beginPath();
  ctx.moveTo(x + radius, y);
  ctx.lineTo(x + width - radius, y);
  ctx.quadraticCurveTo(x + width, y, x + width, y + radius);
  ctx.lineTo(x + width, y + height - radius);
  ctx.quadraticCurveTo(x + width, y + height, x + width - radius, y + height);
  ctx.lineTo(x + radius, y + height);
  ctx.quadraticCurveTo(x, y + height, x, y + height - radius);
  ctx.lineTo(x, y + radius);
  ctx.quadraticCurveTo(x, y, x + radius, y);
  ctx.closePath();
  ctx.clip();
  
  ctx.drawImage(img, x, y, width, height);
  ctx.restore();
}

/* 渲染证书 */
function renderToCanvas() {
  if (!ctx || !img || !img.complete) return
  ctx.clearRect(0, 0, naturalW, naturalH)
  
  const radius = Math.min(naturalW, naturalH) * 0.02;
  drawRoundedImage(ctx, img, 0, 0, naturalW, naturalH, radius);

  const rawName = name.value ? name.value.trim() : ''
  if (!rawName) return
  const fullName = `${rawName} 同学`

  const fontFamily = `"FangZhengKaiTi", "KaiTi", "Microsoft YaHei", serif`
  const initialFontPx = Math.round(naturalW / 32)
  const minFontPx = Math.max(12, Math.round(naturalW / 60))
  const nameMaxWidth = naturalW * 0.6
  const centerX = Math.round(naturalW / 2)
  const baseY = Math.round(naturalH * 0.665)

  ctx.textAlign = 'center'
  ctx.fillStyle = '#000'

  // 单行尝试
  let fontPx = fitFontSize(fullName, nameMaxWidth, initialFontPx, minFontPx, fontFamily)
  ctx.font = `bold ${fontPx}px ${fontFamily}`
  const wSingle = ctx.measureText(fullName).width
  if (wSingle <= nameMaxWidth) {
    const strokeW = Math.max(1, Math.round(fontPx / 16))
    ctx.lineWidth = strokeW
    ctx.strokeStyle = 'rgba(255,255,255,0.85)'
    ctx.strokeText(fullName, centerX, baseY)
    ctx.fillText(fullName, centerX, baseY)
    ctx.lineWidth = 1
    return
  }

  // 两行显示
  const rawLines = splitNameToLines(rawName)
  let lines = rawLines.slice(0,2)
  if (lines.length === 1) lines[0] = `${lines[0]} 同学`
  else lines[1] = `${lines[1]} 同学`

  let lineFontPx = fitFontSize(lines[0], nameMaxWidth, Math.floor(fontPx * 0.9), minFontPx, fontFamily)
  let line2FontPx = fitFontSize(lines[1] || '', nameMaxWidth, lineFontPx, minFontPx, fontFamily)
  const finalFontPx = Math.min(lineFontPx, line2FontPx)

  if (finalFontPx <= minFontPx) {
    ctx.font = `bold ${minFontPx}px ${fontFamily}`
    function ellipsize(text, maxW) {
      let t = text
      while (t.length > 0 && ctx.measureText(t + '…').width > maxW) t = t.slice(0, -1)
      return t + (t.length < text.length ? '…' : '')
    }
    const l1 = ellipsize(lines[0], nameMaxWidth)
    const l2 = ellipsize(lines[1], nameMaxWidth)
    const spacing = Math.round(minFontPx * 1.35)
    ctx.strokeText(l1, centerX, baseY - spacing / 2)
    ctx.fillText(l1, centerX, baseY - spacing / 2)
    ctx.strokeText(l2, centerX, baseY + spacing / 2)
    ctx.fillText(l2, centerX, baseY + spacing / 2)
    ctx.lineWidth = 1
    return
  }

  ctx.font = `bold ${finalFontPx}px ${fontFamily}`
  const lineSpacing = Math.round(finalFontPx * 1.4)
  ctx.strokeText(lines[0], centerX, baseY - lineSpacing / 2)
  ctx.fillText(lines[0], centerX, baseY - lineSpacing / 2)
  ctx.strokeText(lines[1], centerX, baseY + lineSpacing / 2)
  ctx.fillText(lines[1], centerX, baseY + lineSpacing / 2)
  ctx.lineWidth = 1
}

/* 完成输入后校验并显示证书 */
function validateAndShow() {
  showNotInList.value = false
  nameDisplay.value = ''

  const enteredName = name.value ? name.value.trim() : ''
  if (!enteredName) {
    showCertificate.value = false
    return
  }

  const allowed = isNameAllowed(enteredName)
  if (!allowed) {
    showCertificate.value = false
    nameDisplay.value = enteredName
    showNotInList.value = true
    return
  }

  showCertificate.value = true
  nextTick(() => {
    animateIn.value = true
    setTimeout(() => {
      initCanvasWhenMounted()
    }, 40)
    setTimeout(() => { animateIn.value = false }, 700)
  })
}

/* 输入时隐藏证书 */
function onInput() {
  if (showNotInList.value) {
    showNotInList.value = false
    nameDisplay.value = ''
  }
  showCertificate.value = false
}

/* blur / enter / compositionend 触发完成校验 */
function onBlur() {
  if (debounceTimer) { clearTimeout(debounceTimer); debounceTimer = null }
  validateAndShow()
}
function onEnter() {
  if (debounceTimer) { clearTimeout(debounceTimer); debounceTimer = null }
  validateAndShow()
}

/* 下载逻辑 */
async function downloadCertificate() {
  if (!showCertificate.value) {
    validateAndShow()
    await nextTick()
  }

  const canvas = canvasRef.value
  if (!canvas) return
  const rawName = (name.value || '').trim() || '证书'
  const maxLen = 40
  const safeBase = rawName.length > maxLen ? rawName.slice(0, maxLen) : rawName
  const filename = `${safeBase}.png`

  const blob = await new Promise(resolve => canvas.toBlob(b => resolve(b), 'image/png'))
  if (!blob) {
    console.error('生成图片失败')
    return
  }

  try {
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    a.download = filename
    document.body.appendChild(a)
    a.click()
    a.remove()
    setTimeout(() => URL.revokeObjectURL(url), 1500)
    return
  } catch (err) {
    console.warn('objectURL 下载失败，准备回退：', err)
  }

  try {
    const url2 = URL.createObjectURL(blob)
    window.open(url2, '_blank')
    setTimeout(() => URL.revokeObjectURL(url2), 5000)
    return
  } catch (err) {
    console.error('最终回退也失败：', err)
  }
}

/* 缩放管理 */
function updateScale() {
  const wrapper = wrapperRef.value
  const canvas = canvasRef.value
  if (!wrapper || !canvas || !naturalW || !naturalH) return
  
  const wrapperW = wrapper.clientWidth
  const wrapperH = wrapper.clientHeight
  
  const scaleX = (wrapperW - 40) / naturalW
  const scaleY = (wrapperH - 40) / naturalH
  const scale = Math.min(scaleX, scaleY, 1)
  
  canvas.style.width = `${naturalW * scale}px`
  canvas.style.height = `${naturalH * scale}px`
}

function setupResizeObservers() {
  if (ro && wrapperRef.value) {
    try { ro.disconnect() } catch (e) {}
    ro = null
  }
  window.addEventListener('resize', updateScale)
  if (window.ResizeObserver && wrapperRef.value) {
    ro = new ResizeObserver(updateScale)
    ro.observe(wrapperRef.value)
  }
}
function cleanupResizeObservers() {
  window.removeEventListener('resize', updateScale)
  if (ro && wrapperRef.value) ro.disconnect()
  ro = null
}

/* 初始化 canvas */
function initCanvasWhenMounted() {
  const canvasEl = canvasRef.value
  if (!canvasEl) return
  ctx = canvasEl.getContext('2d')

  if (naturalW && naturalH) {
    canvasEl.width = naturalW
    canvasEl.height = naturalH
  }

  renderToCanvas()

  if (typeof document !== 'undefined' && document.fonts && document.fonts.ready) {
    document.fonts.ready.then(() => {
      setTimeout(() => {
        forceRepaintCanvas()
        renderToCanvas()
      }, 60)
    }).catch(() => {})
  }

  updateScale()
  setupResizeObservers()
}

/* 组件挂载 */
onMounted(() => {
  inputEl = document.querySelector('.panel .name-input')
  function handleCompositionStart() { isComposing = true }
  function handleCompositionEnd() {
    isComposing = false
    setTimeout(() => validateAndShow(), 0)
  }
  if (inputEl) {
    inputEl.addEventListener('compositionstart', handleCompositionStart)
    inputEl.addEventListener('compositionend', handleCompositionEnd)
  }

  waitForFont('FangZhengKaiTi', 2000).then((fontOk) => {
    console.log('[font] FangZhengKaiTi available:', fontOk)
    img = new Image()
    img.crossOrigin = 'anonymous'
    img.src = certUrl
    img.onload = () => {
      naturalW = img.naturalWidth || img.width
      naturalH = img.naturalHeight || img.height
    }
    img.onerror = (e) => console.error('证书模板加载失败：', e)
  })

  onBeforeUnmount(() => {
    if (inputEl) {
      inputEl.removeEventListener('compositionstart', handleCompositionStart)
      inputEl.removeEventListener('compositionend', handleCompositionEnd)
      inputEl = null
    }
  })
})

onBeforeUnmount(() => {
  if (debounceTimer) clearTimeout(debounceTimer)
  cleanupResizeObservers()
})

/* 关闭模态 */
function closeModal() {
  showNotInList.value = false
  nameDisplay.value = ''
}
</script>

<style scoped>
/* 提高面板的显示优先级 */
.panel {
  width: 100%;
  max-width: 980px;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
  box-sizing: border-box;
  background: rgba(255, 255, 255, 0.92); /* 半透明白色背景提高可读性 */
  border-radius: 12px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
  backdrop-filter: blur(10px);
  position: relative;
  z-index: 1; /* 确保面板在背景之上 */
  min-height: auto;
}

/* 标题 */
.title-wrap { 
  text-align: center; 
  margin-bottom: 20px; 
  user-select: none; 
}
.title-line1 { 
  margin: 0; 
  font-size: 24px; 
  font-weight: 700; 
  color: #2c3e50;
}
.title-line2 { 
  margin: 8px 0 0 0; 
  font-size: 18px; 
  font-weight: 600; 
  color: #34495e; 
}

/* 控件区 */
.controls { 
  width: 100%; 
  display: flex; 
  gap: 16px; 
  align-items: center; 
  justify-content: center; 
  margin-bottom: 24px; 
  flex-wrap: wrap; 
}
.controls label { 
  display: flex; 
  align-items: center; 
  gap: 8px; 
  font-size: 16px; 
  font-weight: 500;
  color: #2c3e50;
}
.controls input { 
  padding: 10px 14px; 
  border-radius: 8px; 
  border: 2px solid #e1e8ed; 
  font-size: 16px; 
  min-width: 200px; 
  transition: border-color 0.3s;
}
.controls input:focus {
  border-color: #3498db;
  outline: none;
  box-shadow: 0 0 0 3px rgba(52, 152, 219, 0.1);
}

/* 证书容器 - 关键修改 */
.canvas-wrapper {
  width: 100%;
  min-height: 400px;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 20px;
  box-sizing: border-box;
  position: relative;
  z-index: 2; /* 高于面板背景 */
}

/* 证书画布 */
.certificate-canvas {
  border-radius: 12px;
  box-shadow: 0 12px 40px rgba(0,0,0,0.15), 0 0 0 1px rgba(220,180,80,0.1);
  display: block;
  max-width: 100%;
  max-height: 70vh; /* 限制最大高度 */
  object-fit: contain;
  background: white;
}

/* 入场动画 */
@keyframes fadeScaleIn {
  0% { opacity: 0; transform: scale(0.96); }
  60% { opacity: 1; transform: scale(1.02); }
  100% { opacity: 1; transform: scale(1); }
}

.certificate-canvas.animated {
  animation: fadeScaleIn 420ms cubic-bezier(.2,.8,.2,1) both;
}

/* 按钮 */
button { 
  padding: 10px 20px; 
  border-radius: 8px; 
  border: none; 
  background: linear-gradient(135deg,#007bff,#4bc0c8); 
  color: #fff; 
  cursor: pointer; 
  font-size: 16px;
  font-weight: 500;
  transition: all 0.3s;
}
button:hover { 
  transform: translateY(-2px); 
  box-shadow: 0 6px 20px rgba(0, 123, 255, 0.3);
}

/* 模态 */
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
  padding: 24px; 
  border-radius: 12px; 
  width: min(420px, 90%); 
  box-shadow: 0 8px 30px rgba(0,0,0,0.25); 
  text-align: center; 
}
.modal-box h3 { 
  margin: 0 0 12px 0; 
  color: #2c3e50;
}
.modal-actions { 
  margin-top: 16px; 
}
.modal-actions button { 
  padding: 10px 20px; 
  border-radius: 8px; 
  background: #007bff; 
  color: #fff; 
  border: none; 
  cursor: pointer; 
}

/* 响应式调整 */
@media (max-width: 768px) {
  .panel {
    margin: 10px;
    padding: 16px;
    max-width: calc(100% - 20px);
  }
  
  .canvas-wrapper {
    padding: 15px;
    min-height: 300px;
  }
  
  .certificate-canvas {
    max-height: 60vh;
  }
  
  .controls {
    flex-direction: column;
    gap: 12px;
  }
  
  .controls input {
    min-width: 100%;
    max-width: 280px;
  }
  
  .title-line1 { font-size: 20px; }
  .title-line2 { font-size: 16px; }
}

@media (max-width: 480px) {
  .panel {
    padding: 12px;
    margin: 8px;
  }
  
  .canvas-wrapper {
    padding: 10px;
    min-height: 250px;
  }
  
  .certificate-canvas {
    max-height: 50vh;
  }
}
</style>

<!-- 全局字体声明 -->
<style>
@font-face {
  font-family: 'FangZhengKaiTi';
  src: url('/fonts/FangZhengKaiTiJianTi-1.ttf') format('truetype');
  font-weight: 400 700;
  font-style: normal;
  font-display: swap;
}
</style>