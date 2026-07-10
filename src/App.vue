<template>
  <div class="remote-app">
    <div class="bg-glow"></div>

    <transition name="fade">
      <div class="connect-overlay" v-if="!isConnected">
        <div class="glass-panel connect-box">
          <h2>大屏终端连接</h2>
          <p class="hint">输入主机 IP 地址建立神经链路</p>
          <div class="input-group">
            <span class="prefix">IP</span>
            <input type="text" v-model="serverIp" placeholder="192.168.x.x" @keydown.enter="connectToServer" />
          </div>
          <button class="btn-primary" @click="connectToServer" :disabled="isConnecting">
            {{ isConnecting ? '建立链路中...' : '接入终端' }}
          </button>
          <p class="error-msg" v-show="connectError">{{ connectError }}</p>
        </div>
      </div>
    </transition>

    <header class="status-bar glass-panel">
      <div class="brand" @click="disconnectServer">
        控制中枢 <span class="disconnect-text">断开</span>
      </div>
      <div class="tool-group">
        <div class="tool-btn" @click="openKeyboard">输入</div>
        <div :class="['tool-btn', { active: gyroEnabled }]" @click="toggleGyro">空鼠</div>
        <div :class="['status-dot', isConnected ? 'online' : 'offline']"></div>
      </div>
    </header>

    <input 
      ref="hiddenInput" 
      type="text" 
      class="hidden-input" 
      v-model="inputText" 
      @input="handleTyping"
      @compositionstart="onCompositionStart"
      @compositionend="onCompositionEnd"
      @keydown.enter="handlePress($event, 'Enter')" 
      @keydown.delete="handlePress($event, 'Backspace')" 
    />

    <section class="control-section">
      <div class="d-pad-ring glass-panel">
        <div class="dir-btn up" @touchstart="handlePress($event, 'Up')" @mousedown="handlePress($event, 'Up')">
          <svg class="svg-arrow" viewBox="0 0 24 24"><polygon points="12,4 4,20 20,20" fill="currentColor"/></svg>
        </div>
        <div class="dir-btn right" @touchstart="handlePress($event, 'Right')" @mousedown="handlePress($event, 'Right')">
          <svg class="svg-arrow" viewBox="0 0 24 24"><polygon points="4,4 20,12 4,20" fill="currentColor"/></svg>
        </div>
        <div class="dir-btn down" @touchstart="handlePress($event, 'Down')" @mousedown="handlePress($event, 'Down')">
          <svg class="svg-arrow" viewBox="0 0 24 24"><polygon points="4,4 20,4 12,20" fill="currentColor"/></svg>
        </div>
        <div class="dir-btn left" @touchstart="handlePress($event, 'Left')" @mousedown="handlePress($event, 'Left')">
          <svg class="svg-arrow" viewBox="0 0 24 24"><polygon points="20,4 4,12 20,20" fill="currentColor"/></svg>
        </div>
        <div class="ok-btn" @touchstart="handlePress($event, 'Enter')" @mousedown="handlePress($event, 'Enter')">
          <div class="ok-inner">确定</div>
        </div>
      </div>
    </section>

    <section class="action-section">
      <div class="action-row">
        <div class="action-btn glass-panel" @touchstart="handlePress($event, 'Home')" @mousedown="handlePress($event, 'Home')">主页</div>
        <div class="action-btn glass-panel" @touchstart="handlePress($event, 'Escape')" @mousedown="handlePress($event, 'Escape')">返回</div>
        <div class="action-btn glass-panel" @touchstart="handlePress($event, 'VolumeDown')" @mousedown="handlePress($event, 'VolumeDown')">音量-</div>
        <div class="action-btn glass-panel" @touchstart="handlePress($event, 'VolumeUp')" @mousedown="handlePress($event, 'VolumeUp')">音量+</div>
      </div>
      <div class="action-row">
        <div class="action-btn glass-panel highlight" @touchstart="handlePress($event, 'Space')" @mousedown="handlePress($event, 'Space')">播放/暂停</div>
        <div class="action-btn glass-panel" @touchstart="handlePress($event, 'AudioVolumeMute')" @mousedown="handlePress($event, 'AudioVolumeMute')">静音</div>
        <div class="action-btn glass-panel" @touchstart="handlePress($event, 'M')" @mousedown="handlePress($event, 'M')">菜单</div>
        <div class="action-btn glass-panel" @touchstart="handlePress($event, 'F')" @mousedown="handlePress($event, 'F')">全屏</div>
      </div>
    </section>

    <div class="sensitivity-bar glass-panel">
      <span class="label">神经感知灵敏度</span>
      <input 
        type="range" 
        min="1" max="10" step="1" 
        v-model.number="sensitivity" 
        class="slider" 
        @change="saveSensitivity" 
      />
      <span class="value">Lv {{ sensitivity }}</span>
    </div>

    <transition name="flip" mode="out-in">
      <section 
        v-if="!gyroEnabled" 
        class="touchpad-section glass-panel"
        @touchstart="onTouchStart"
        @touchmove.prevent="onTouchMove"
        @touchend="onTouchEnd"
      >
        <div class="touch-grid"></div>
        <div class="touch-indicator">
          <p class="status-title">全息触控区域</p>
          <span class="status-sub">单指滑动移动光标 · 轻点执行确认</span>
        </div>
      </section>

      <section v-else class="airmouse-section glass-panel">
        <div class="airmouse-header">
          <div class="air-status">
            <span class="pulse-dot"></span> 空鼠链路已激活，挥动手机操作
          </div>
        </div>
        <div class="airmouse-controls">
          <div class="mouse-btn left-click" @touchstart="handleMouse($event, 'left')" @mousedown="handleMouse($event, 'left')">
            <div class="btn-core">左键 (L)</div>
          </div>
          <div class="mouse-btn right-click" @touchstart="handleMouse($event, 'right')" @mousedown="handleMouse($event, 'right')">
            <div class="btn-core">右键</div>
          </div>
        </div>
      </section>
    </transition>
  </div>
</template>

<script setup>
import { ref, watch, onMounted, onUnmounted } from 'vue';
import { io } from 'socket.io-client';

const serverIp = ref('');
const isConnected = ref(false);
const isConnecting = ref(false);
const connectError = ref('');
let socket = null;
let pingTimer = null; 
const sensitivity = ref(5); 

onMounted(() => {
  const savedIp = localStorage.getItem('remoteServerIp');
  if (savedIp) {
    serverIp.value = savedIp;
    connectToServer();
  } else {
    const host = window.location.hostname;
    if (host !== 'localhost' && host !== '127.0.0.1') {
      const parts = host.split('.');
      if (parts.length === 4) serverIp.value = `${parts[0]}.${parts[1]}.${parts[2]}.`;
    }
  }
  const savedSens = localStorage.getItem('remoteSensitivity');
  if (savedSens) sensitivity.value = parseInt(savedSens);

  pingTimer = setInterval(() => {
    if (socket && !socket.connected && isConnected.value) {
      isConnected.value = false;
    }
  }, 5000);
});

const saveSensitivity = () => {
  localStorage.setItem('remoteSensitivity', sensitivity.value);
  vibrate(10);
};

const connectToServer = () => {
  if (!serverIp.value) return;
  isConnecting.value = true; connectError.value = '';
  if (socket) socket.disconnect();
  
  const targetUrl = `http://${serverIp.value.replace('http://', '').replace('https://', '')}:3000`;
  console.log("【排查】🎯 开始尝试向目标通道建立连接:", targetUrl);

  // 维持直接锁定 websocket 传输层的策略
  socket = io(targetUrl, { 
    timeout: 3500, 
    reconnectionAttempts: Infinity,
    transports: ['websocket']
  });

  // ====================================================
  // 🌟 新增：Socket.io 底层引擎探针群 (专杀原生 WebView 拦截)
  // ====================================================
  
  // 1. 监测底层的第一个包交互 (如果有)
  socket.io.engine.on("initial_headers", (headers) => {
    console.log("【排查】📦 底层正在尝试发起初始握手，Header 报文:", JSON.stringify(headers));
  });

  // 2. 监测从 HTTP 升级为 WS 时的协议熔断报错
  socket.io.engine.on("upgradeError", (err) => {
    console.error("【排查】❌ 协议升级层发生异常断开:", err.message || err);
  });

  // 3. 截获最关键的致命错误堆栈
  socket.on('connect_error', (err) => {
    console.error("【排查】🚨 链路断开，捕获到核心报错对象:", err);
    console.error("👉 错误核心简述 (Message):", err.message);
    console.error("👉 错误底层描述 (Description):", err.description || "无具体描述");
    
    // 提取极其重要的原生 Error Cause (混合内容拦截、跨域拒绝经常藏在这里)
    if (err.cause) {
      console.error("👉 致命元凶底层 Cause 细节:", err.cause.message || err.cause);
    } else {
      console.error("👉 底层未附带 Cause 原因，可能属于物理超时或 IP 无法路由。");
    }

    isConnecting.value = false; 
    isConnected.value = false;
    connectError.value = `链路受阻: ${err.message}`;
  });

  // ====================================================

  socket.on('connect', () => {
    console.log("【排查】🚀 神经链路建立成功！已进入长连接状态！");
    isConnected.value = true; isConnecting.value = false; connectError.value = '';
    localStorage.setItem('remoteServerIp', serverIp.value);
    vibrate(30);
  });
  
  socket.on('disconnect', (reason) => {
    console.warn(`【排查】🔌 链路主动/被动断开，原因代码: ${reason}`);
    isConnected.value = false;
  });
};

const disconnectServer = () => {
  if (socket) { socket.disconnect(); isConnected.value = false; }
};

const vibrate = (duration = 15) => {
  try { if (navigator.vibrate) navigator.vibrate(duration); } catch (e) {}
};

let lastKeyTime = 0;
const handlePress = (e, keyName) => {
  const now = Date.now();
  if (now - lastKeyTime < 50) return;
  lastKeyTime = now;
  vibrate(20); 
  if (socket && isConnected.value) socket.emit('key', keyName);
};

const handleMouse = (e, button) => {
  const now = Date.now();
  if (now - lastKeyTime < 50) return;
  lastKeyTime = now;
  vibrate(button === 'left' ? 25 : 15);
  if (socket && isConnected.value) {
    socket.emit(button === 'left' ? 'click' : 'rightClick');
  }
};

const hiddenInput = ref(null);
const inputText = ref('');
let lastText = '';
let isComposing = false; 

const openKeyboard = () => {
  if (hiddenInput.value) { hiddenInput.value.focus(); vibrate(20); }
};

const onCompositionStart = () => { isComposing = true; };
const onCompositionEnd = (e) => {
  isComposing = false;
  processInput(e.target.value);
};
const handleTyping = (e) => {
  if (isComposing) return;
  processInput(e.target.value);
};

const processInput = (currentText) => {
  if (currentText.length > lastText.length) {
    const newText = currentText.slice(lastText.length);
    if (newText && socket && isConnected.value) socket.emit('typeText', newText);
  }
  lastText = currentText;
  if (lastText.length > 50) {
    lastText = lastText.slice(-20);
    inputText.value = lastText;
  }
};

const startX = ref(0), startY = ref(0);
let touchStartTime = 0, isSwipe = false;

const onTouchStart = (e) => {
  startX.value = e.touches[0].clientX; startY.value = e.touches[0].clientY;
  touchStartTime = Date.now(); isSwipe = false;
};

const onTouchMove = (e) => {
  isSwipe = true; 
  const currentX = e.touches[0].clientX, currentY = e.touches[0].clientY;
  if (socket && isConnected.value) {
    const scale = sensitivity.value / 5.0; 
    socket.emit('mousemove', { 
      x: (currentX - startX.value) * scale, 
      y: (currentY - startY.value) * scale 
    });
  }
  startX.value = currentX; startY.value = currentY;
};

const onTouchEnd = () => {
  if (!isSwipe && (Date.now() - touchStartTime) < 200) {
    vibrate(10);
    if (socket && isConnected.value) socket.emit('click');
  }
};

// ==========================================
// 空鼠控制逻辑
// ==========================================
const gyroEnabled = ref(false);
let lastBeta = null;
let lastAlpha = null;
let gyroInitialized = false;
let lastGyroSend = 0;

let smoothedVelocityX = 0;
let smoothedVelocityY = 0;
const LPF_ALPHA = 0.45; 

const toggleGyro = async () => {
  if (!gyroEnabled.value) {
    gyroEnabled.value = true;
    gyroInitialized = false; 
    smoothedVelocityX = 0;
    smoothedVelocityY = 0;
    vibrate([15, 30, 15]);
  } else {
    gyroEnabled.value = false;
    vibrate([15, 30, 15]);
  }
};

const handleOrientation = (event) => {
  if (event.beta === null || event.alpha === null) return;

  const now = Date.now();
  if (now - lastGyroSend < 16) return; 
  lastGyroSend = now;

  if (!gyroInitialized || lastBeta === null || lastAlpha === null) {
    lastBeta = event.beta;
    lastAlpha = event.alpha;
    gyroInitialized = true;
    return;
  }

  let deltaBeta = event.beta - lastBeta;
  let deltaAlpha = event.alpha - lastAlpha;

  if (deltaAlpha > 180) deltaAlpha -= 360;
  else if (deltaAlpha < -180) deltaAlpha += 360;

  if (Math.abs(deltaBeta) > 30 || Math.abs(deltaAlpha) > 30) {
    lastBeta = event.beta;
    lastAlpha = event.alpha;
    return; 
  }

  lastBeta = event.beta;
  lastAlpha = event.alpha;

  let rawVelocityX = 0;
  let rawVelocityY = 0;
  
  const gyroBaseSens = 18 * (sensitivity.value / 5.0); 
  
  if (Math.abs(deltaBeta) > 0.08 || Math.abs(deltaAlpha) > 0.08) {
    rawVelocityX = -deltaAlpha * gyroBaseSens;
    rawVelocityY = -deltaBeta * gyroBaseSens;
  }

  smoothedVelocityX = (smoothedVelocityX * (1 - LPF_ALPHA)) + (rawVelocityX * LPF_ALPHA);
  smoothedVelocityY = (smoothedVelocityY * (1 - LPF_ALPHA)) + (rawVelocityY * LPF_ALPHA);

  if (Math.abs(smoothedVelocityX) < 0.25) smoothedVelocityX = 0;
  if (Math.abs(smoothedVelocityY) < 0.25) smoothedVelocityY = 0;

  if (smoothedVelocityX !== 0 || smoothedVelocityY !== 0) {
    if (socket && isConnected.value) {
      socket.emit('mousemove', { x: smoothedVelocityX, y: smoothedVelocityY });
    }
  }
};

watch(gyroEnabled, (val) => {
  if (val) window.addEventListener('deviceorientation', handleOrientation);
  else {
    window.removeEventListener('deviceorientation', handleOrientation);
    gyroInitialized = false;
  }
});

onUnmounted(() => {
  window.removeEventListener('deviceorientation', handleOrientation);
  if (pingTimer) clearInterval(pingTimer);
  if (socket) socket.disconnect();
});
</script>

<style scoped>
.remote-app {
  height: 100vh; display: flex; flex-direction: column;
  background: #020617; color: #f1f5f9; padding: 3vh 15px 2vh; 
  box-sizing: border-box; font-family: -apple-system, sans-serif;
  user-select: none; -webkit-user-select: none; position: relative; overflow: hidden;
  touch-action: none; -webkit-tap-highlight-color: transparent; 
}

.bg-glow { position: absolute; top: -10%; left: -10%; width: 120%; height: 120%; background: radial-gradient(circle at 50% 0%, rgba(14, 165, 233, 0.15) 0%, transparent 60%); pointer-events: none; }
.glass-panel { background: rgba(255, 255, 255, 0.05); backdrop-filter: blur(24px); -webkit-backdrop-filter: blur(24px); border: 1px solid rgba(255, 255, 255, 0.1); box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2); z-index: 1; }
.hidden-input { position: absolute; top: -9999px; opacity: 0; }
.status-bar { display: flex; justify-content: space-between; align-items: center; margin-bottom: 2vh; padding: 10px 16px; border-radius: 100px; }
.brand { font-size: 16px; color: #fff; font-weight: 600; letter-spacing: 1px; cursor: pointer; }
.disconnect-text { font-size: 12px; color: #94a3b8; margin-left: 4px; }
.tool-group { display: flex; align-items: center; gap: 10px; }
.tool-btn { font-size: 13px; padding: 5px 10px; border-radius: 20px; background: rgba(255,255,255,0.08); color: #cbd5e1; transition: all 0.2s; cursor: pointer; }
.tool-btn:active { transform: scale(0.9); }
.tool-btn.active { background: #0ea5e9; color: #fff; box-shadow: 0 0 16px rgba(14,165,233,0.4); border: 1px solid rgba(14,165,233,0.8); }
.status-dot { width: 8px; height: 8px; border-radius: 50%; }
.status-dot.online { background-color: #10b981; box-shadow: 0 0 10px #10b981; }
.status-dot.offline { background-color: #ef4444; }

.control-section { display: flex; justify-content: center; margin-bottom: 2vh; }
.d-pad-ring { position: relative; width: 220px; height: 220px; border-radius: 50%; } 
.dir-btn { position: absolute; width: 60px; height: 60px; display: flex; justify-content: center; align-items: center; color: #94a3b8; transition: all 0.1s; border-radius: 50%; cursor: pointer; }
.svg-arrow { width: 22px; height: 22px; transition: transform 0.1s; }
.dir-btn:active { background: rgba(14, 165, 233, 0.25); color: #0ea5e9; box-shadow: 0 0 20px rgba(14, 165, 233, 0.4); }
.dir-btn:active .svg-arrow { transform: scale(1.3); }
.dir-btn.up { top: 5px; left: 80px; } .dir-btn.down { bottom: 5px; left: 80px; }
.dir-btn.left { top: 80px; left: 5px; } .dir-btn.right { top: 80px; right: 5px; }
.ok-btn { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 84px; height: 84px; border-radius: 50%; display: flex; justify-content: center; align-items: center; padding: 6px; box-sizing: border-box; cursor: pointer; }
.ok-inner { width: 100%; height: 100%; border-radius: 50%; background: rgba(14, 165, 233, 0.15); border: 1px solid rgba(14, 165, 233, 0.5); display: flex; justify-content: center; align-items: center; font-weight: 600; font-size: 16px; color: #38bdf8; transition: all 0.1s; }
.ok-btn:active .ok-inner { background: #0ea5e9; color: #fff; box-shadow: 0 0 30px rgba(14, 165, 233, 0.8); transform: scale(0.85); }

.action-section { display: flex; flex-direction: column; gap: 10px; margin: 0 2px 2vh; }
.action-row { display: flex; justify-content: space-between; width: 100%; }
.action-btn { flex: 1; margin: 0 4px; height: 44px; border-radius: 10px; display: flex; justify-content: center; align-items: center; font-size: 13px; color: #e2e8f0; font-weight: 500; transition: all 0.1s; white-space: nowrap; cursor: pointer; }
.action-btn.highlight { color: #38bdf8; font-weight: 600; }
.action-btn:active { background: rgba(14, 165, 233, 0.3); border: 1px solid rgba(14, 165, 233, 0.8); color: #fff; transform: scale(0.92) translateY(2px); }

.sensitivity-bar {
  display: flex; align-items: center; justify-content: space-between;
  padding: 10px 14px; border-radius: 14px; margin: 0 5px 1.5vh;
  background: rgba(14, 165, 233, 0.05); border: 1px solid rgba(14, 165, 233, 0.2);
}
.sensitivity-bar .label { font-size: 12px; color: #94a3b8; font-weight: 600; }
.sensitivity-bar .value { font-size: 12px; color: #38bdf8; font-weight: bold; width: 40px; text-align: right; }
.slider { flex: 1; margin: 0 12px; -webkit-appearance: none; background: transparent; outline: none; }
.slider::-webkit-slider-runnable-track { width: 100%; height: 6px; background: rgba(255,255,255,0.1); border-radius: 3px; cursor: pointer; }
.slider::-webkit-slider-thumb { -webkit-appearance: none; height: 16px; width: 16px; border-radius: 50%; background: #0ea5e9; margin-top: -5px; box-shadow: 0 0 10px rgba(14,165,233,0.6); cursor: pointer; transition: transform 0.1s; }
.slider:active::-webkit-slider-thumb { transform: scale(1.2); }

.flip-enter-active, .flip-leave-active { transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1); transform-style: preserve-3d; }
.flip-enter-from { opacity: 0; transform: rotateX(-90deg) scale(0.9); }
.flip-leave-to { opacity: 0; transform: rotateX(90deg) scale(0.9); }

.touchpad-section { flex: 1; border-radius: 20px; display: flex; justify-content: center; align-items: center; position: relative; overflow: hidden; min-height: 180px; }
.touch-grid { position: absolute; top: 0; left: 0; right: 0; bottom: 0; z-index: 0; background-image: linear-gradient(rgba(255,255,255,0.03) 1px, transparent 1px), linear-gradient(90deg, rgba(255,255,255,0.03) 1px, transparent 1px); background-size: 20px 20px; }
.touch-indicator { text-align: center; z-index: 1; pointer-events: none; }
.status-title { margin: 0 0 8px; font-size: 16px; font-weight: 600; color: #38bdf8; letter-spacing: 1px; }
.status-sub { font-size: 11px; color: #64748b; letter-spacing: 0.5px; }

.airmouse-section { flex: 1; border-radius: 20px; display: flex; flex-direction: column; padding: 16px; background: rgba(14, 165, 233, 0.05); border: 1px solid rgba(14, 165, 233, 0.2); min-height: 180px; }
.airmouse-header { display: flex; justify-content: center; align-items: center; margin-bottom: 12px; }
.air-status { font-size: 12px; color: #38bdf8; font-weight: 600; display: flex; align-items: center; }
.pulse-dot { width: 8px; height: 8px; background: #0ea5e9; border-radius: 50%; margin-right: 8px; animation: pulse 1.5s infinite; }
@keyframes pulse { 0% { box-shadow: 0 0 0 0 rgba(14, 165, 233, 0.4); } 70% { box-shadow: 0 0 0 8px rgba(14, 165, 233, 0); } 100% { box-shadow: 0 0 0 0 rgba(14, 165, 233, 0); } }

.airmouse-controls { flex: 1; display: flex; gap: 12px; }
.mouse-btn { border-radius: 16px; display: flex; justify-content: center; align-items: center; cursor: pointer; transition: all 0.1s cubic-bezier(0.4, 0, 0.2, 1); }
.mouse-btn.left-click { flex: 2; background: rgba(255, 255, 255, 0.05); border: 1px solid rgba(255, 255, 255, 0.1); }
.mouse-btn.right-click { flex: 1; background: rgba(255, 255, 255, 0.05); border: 1px solid rgba(255, 255, 255, 0.1); }
.mouse-btn:active { transform: scale(0.96) translateY(2px); }
.mouse-btn.left-click:active { background: #0ea5e9; box-shadow: 0 0 20px rgba(14, 165, 233, 0.6); border-color: #0ea5e9; }
.mouse-btn.left-click:active .btn-core { color: #fff; }
.mouse-btn.right-click:active { background: rgba(255, 255, 255, 0.15); border-color: #fff; }
.mouse-btn.right-click:active .btn-core { color: #fff; }
.mouse-btn.left-click .btn-core { font-size: 20px; font-weight: bold; color: #e2e8f0; }
.mouse-btn.right-click .btn-core { font-size: 15px; color: #94a3b8; font-weight: 500; }

.fade-enter-active, .fade-leave-active { transition: opacity 0.3s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
.connect-overlay { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background: rgba(2, 6, 23, 0.8); backdrop-filter: blur(10px); z-index: 999; display: flex; justify-content: center; align-items: center; }
.connect-box { width: 80%; padding: 35px 25px; border-radius: 24px; text-align: center; }
.connect-box h2 { margin: 0 0 10px; color: #fff; font-size: 22px; font-weight: 600; }
.connect-box .hint { color: #94a3b8; font-size: 13px; margin-bottom: 30px; }
.input-group { display: flex; align-items: center; background: rgba(0, 0, 0, 0.2); border: 1px solid rgba(255, 255, 255, 0.15); border-radius: 16px; padding: 12px 16px; margin-bottom: 25px; }
.input-group .prefix { color: #38bdf8; font-weight: bold; margin-right: 10px; }
.input-group input { flex: 1; background: transparent; border: none; color: #fff; font-size: 16px; outline: none; text-align: center; }
.btn-primary { background: #0ea5e9; color: #fff; border: none; padding: 14px 0; font-size: 16px; font-weight: 600; border-radius: 16px; width: 100%; transition: all 0.2s; }
.btn-primary:active { background: #0284c7; transform: scale(0.98); }
.btn-primary:disabled { background: rgba(255,255,255,0.1); color: #64748b; }
.error-msg { color: #ef4444; font-size: 13px; margin-top: 15px; margin-bottom: 0; }
</style>