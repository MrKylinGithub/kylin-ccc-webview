<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { ElMessage } from 'element-plus';
import { 
  Refresh, 
  ArrowLeft, 
  ArrowRight, 
  HomeFilled,
  Search
} from '@element-plus/icons-vue';
import * as os from 'os';

// 获取当前电脑的LAN地址
const getLANAddress = () => {
  const interfaces = os.networkInterfaces();
  
  for (const name of Object.keys(interfaces)) {
    const nets = interfaces[name];
    if (!nets) continue;
    
    for (const net of nets) {
      // 跳过非IPv4和内部地址
      if (net.family === 'IPv4' && !net.internal) {
        // 优先返回192.168或10.0开头的地址（常见的局域网地址）
        if (net.address.startsWith('192.168.') || net.address.startsWith('10.')) {
          return net.address;
        }
      }
    }
  }
  
  // 如果没找到局域网地址，再找任何非内部IPv4地址
  for (const name of Object.keys(interfaces)) {
    const nets = interfaces[name];
    if (!nets) continue;
    
    for (const net of nets) {
      if (net.family === 'IPv4' && !net.internal) {
        return net.address;
      }
    }
  }
  
  // 最后的回退选项
  return '127.0.0.1';
};

// 响应式数据
const lanAddress = getLANAddress();
const previewUrl = ref(`http://${lanAddress}:7456`);
const currentUrl = ref('');
const inputUrl = ref(`http://${lanAddress}:7456`);
const webviewRef = ref<HTMLElement>();
const loading = ref(false);
const canGoBack = ref(false);
const canGoForward = ref(false);
const hasContent = ref(false);
// 历史记录管理
const history = ref<string[]>([]);
const historyIndex = ref(-1);

// 获取Cocos Creator预览地址
const getPreviewUrl = async (): Promise<void> => {
  try {
    // 使用Cocos Creator的query-preview-url API获取预览地址
    const url = await Editor.Message.request('preview', 'query-preview-url');
    
    if (url) {
      previewUrl.value = url;
      inputUrl.value = url;
      console.log('获取到的预览地址:', url);
    } else {
      // 如果API返回空值，使用默认地址
      const fallbackUrl = `http://${lanAddress}:7456`;
      previewUrl.value = fallbackUrl;
      inputUrl.value = fallbackUrl;
      console.log('API返回空值，使用默认预览地址:', fallbackUrl);
    }
    
  } catch (error) {
    console.log('获取预览地址失败，使用默认值:', error);
    const fallbackUrl = `http://${lanAddress}:7456`;
    previewUrl.value = fallbackUrl;
    inputUrl.value = fallbackUrl;
  }
};

// 组件挂载时获取预览地址并自动加载
onMounted(async () => {
  // 等待场景准备好
  await waitForSceneReady();
  // 场景准备好后再获取预览地址并加载
  await getPreviewUrl();
  // 等待一下确保DOM渲染完成，然后自动加载预览地址
  setTimeout(() => {
    loadUrl();
  }, 100);
});

// 等待场景准备好
const waitForSceneReady = async (): Promise<void> => {
  const maxRetries = 30; // 最多等待30秒
  const retryInterval = 1000; // 每秒检查一次
  
  for (let i = 0; i < maxRetries; i++) {
    try {
      const isReady = await Editor.Message.request('scene', 'query-is-ready');
      if (isReady) {
        console.log('场景已准备好，开始加载webview');
        return;
      }
    } catch (error) {
      console.log('检查场景状态失败:', error);
    }
    
    // 等待一秒后重试
    await new Promise(resolve => setTimeout(resolve, retryInterval));
    console.log(`等待场景准备... (${i + 1}/${maxRetries})`);
  }
  
  console.warn('场景长时间未准备好，强制继续加载webview');
};

// 加载网页
const loadUrl = () => {
  if (!inputUrl.value.trim()) {
    ElMessage.warning('请输入有效的网址');
    return;
  }
  
  let url = inputUrl.value.trim();
  
  // 如果不包含协议，默认添加https://
  if (!url.match(/^https?:\/\//)) {
    url = 'https://' + url;
  }
  
  // 管理历史记录
  if (currentUrl.value && currentUrl.value !== url) {
    // 如果当前不在历史记录末尾，清除后面的记录
    if (historyIndex.value < history.value.length - 1) {
      history.value = history.value.slice(0, historyIndex.value + 1);
    }
    // 添加当前URL到历史记录
    history.value.push(currentUrl.value);
    historyIndex.value = history.value.length - 1;
  } else if (!currentUrl.value) {
    // 第一次加载，初始化历史记录
    history.value = [];
    historyIndex.value = -1;
  }
  
  currentUrl.value = url;
  inputUrl.value = url;
  loading.value = true;
  hasContent.value = true;
  
  // 更新导航按钮状态
  canGoBack.value = history.value.length > 0;
  canGoForward.value = false; // 加载新页面后不能前进
  
  // 更新webview src
  if (webviewRef.value) {
    (webviewRef.value as HTMLIFrameElement).src = url;
  }
};

// 刷新页面
const refresh = () => {
  if (!currentUrl.value) {
    ElMessage.warning('请先输入网址');
    return;
  }
  loading.value = true;
  if (webviewRef.value) {
    // iframe没有reload方法，重新设置src来刷新
    const iframe = webviewRef.value as HTMLIFrameElement;
    const currentSrc = iframe.src;
    iframe.src = '';
    setTimeout(() => {
      iframe.src = currentSrc;
    }, 100);
  }
};

// 后退
const goBack = () => {
  if (canGoBack.value && historyIndex.value > 0) {
    historyIndex.value--;
    const url = history.value[historyIndex.value];
    currentUrl.value = url;
    inputUrl.value = url;
    loading.value = true;
    
    // 更新导航按钮状态
    canGoBack.value = historyIndex.value > 0;
    canGoForward.value = historyIndex.value < history.value.length - 1;
    
    // 更新iframe src
    if (webviewRef.value) {
      (webviewRef.value as HTMLIFrameElement).src = url;
    }
  }
};

// 前进
const goForward = () => {
  if (canGoForward.value && historyIndex.value < history.value.length - 1) {
    historyIndex.value++;
    const url = history.value[historyIndex.value];
    currentUrl.value = url;
    inputUrl.value = url;
    loading.value = true;
    
    // 更新导航按钮状态
    canGoBack.value = historyIndex.value > 0;
    canGoForward.value = historyIndex.value < history.value.length - 1;
    
    // 更新iframe src
    if (webviewRef.value) {
      (webviewRef.value as HTMLIFrameElement).src = url;
    }
  }
};

// 回到首页
const goHome = async () => {
  // 每次点击主页时重新获取最新的预览地址
  await getPreviewUrl();
  loadUrl();
};

// webview事件处理
const onWebviewLoad = () => {
  loading.value = false;
  // iframe加载完成后，导航按钮状态已经在loadUrl等函数中更新了
  // 这里不需要额外的操作，因为iframe没有canGoBack/canGoForward/getURL等方法
};

const onWebviewLoadStart = () => {
  loading.value = true;
};

const onWebviewLoadStop = () => {
  loading.value = false;
};

// 处理Enter键
const handleEnter = () => {
  loadUrl();
};


</script>

<template>
  <div class="webview-container">
    <!-- 工具栏 -->
    <div class="toolbar">
      <!-- 导航按钮 -->
      <div class="nav-buttons">
        <el-button 
          :disabled="!canGoBack" 
          @click="goBack"
          size="small"
          :icon="ArrowLeft"
        />
        <el-button 
          :disabled="!canGoForward" 
          @click="goForward"
          size="small"
          :icon="ArrowRight"
        />
        <el-button 
          @click="refresh"
          size="small"
          :icon="Refresh"
          :loading="loading"
        />
        <el-button 
          @click="goHome"
          size="small"
          :icon="HomeFilled"
        />
      </div>
      
      <!-- 地址栏 -->
      <div class="address-bar">
        <el-input
          v-model="inputUrl"
          placeholder="请输入网址..."
          @keyup.enter="handleEnter"
          size="small"
        >
          <template #append>
            <el-button 
              @click="loadUrl"
              :icon="Search"
              size="small"
            />
          </template>
        </el-input>
      </div>
    </div>

    <!-- WebView显示区域 -->
    <div class="webview-wrapper">
      <!-- 空状态 -->
      <div v-if="!hasContent" class="empty-state">
        <div class="empty-content">
          <el-icon class="empty-icon">
            <Search />
          </el-icon>
          <p class="empty-text">请输入网址开始浏览</p>
        </div>
      </div>
      
      <!-- 加载状态 -->
      <div v-if="loading && hasContent" class="loading-overlay">
        <el-icon class="loading-icon">
          <Refresh />
        </el-icon>
        <span>加载中...</span>
      </div>
      
      <!-- WebView内容 -->
      <iframe
        v-if="hasContent"
        ref="webviewRef"
        :src="currentUrl"
        class="webview"
        @load="onWebviewLoad"
        frameborder="0"
        allowfullscreen
      />
    </div>
  </div>
</template>

<style scoped>
.webview-container {
  display: flex;
  flex-direction: column;
  height: 100%;
  width: 100%;
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  background-color: #2b2b2b;
  color: #cccccc;
  overflow: hidden;
  position: relative;
}

.toolbar {
  display: flex;
  align-items: center;
  padding: 6px 0;
  background-color: #3c3c3c;
  border-bottom: 1px solid #525252;
  gap: 8px;
  flex-shrink: 0;
  min-height: 40px;
}

.nav-buttons {
  display: flex;
  gap: 4px;
  flex-shrink: 0;
  margin-left: 8px;
}

.nav-buttons .el-button {
  background-color: #4a4a4a;
  border-color: #606060;
  color: #cccccc;
  min-width: 32px;
  height: 28px;
  padding: 0 8px;
}

.nav-buttons .el-button:hover {
  background-color: #5a5a5a;
  border-color: #707070;
}

.nav-buttons .el-button:disabled {
  background-color: #3a3a3a;
  border-color: #4a4a4a;
  color: #666666;
}

.address-bar {
  flex: 1;
  min-width: 200px;
  margin-right: 8px;
}

.address-bar :deep(.el-input) {
  --el-input-bg-color: #4a4a4a;
  --el-input-border-color: #606060;
  --el-input-hover-border-color: #707070;
  --el-input-focus-border-color: #409eff;
  --el-input-text-color: #cccccc;
  --el-input-placeholder-color: #999999;
}

.address-bar :deep(.el-input__wrapper) {
  background-color: #4a4a4a;
  border-color: #606060;
}

.address-bar :deep(.el-input__wrapper:hover) {
  border-color: #707070;
}

.address-bar :deep(.el-input__inner) {
  color: #cccccc;
}

.address-bar :deep(.el-input__inner::placeholder) {
  color: #999999;
}

.address-bar :deep(.el-input-group__append) {
  background-color: #4a4a4a;
  border-color: #606060;
}

.address-bar :deep(.el-input-group__append .el-button) {
  background-color: #4a4a4a;
  border-color: #606060;
  color: #cccccc;
}

.address-bar :deep(.el-input-group__append .el-button:hover) {
  background-color: #5a5a5a;
  border-color: #707070;
}

.webview-wrapper {
  flex: 1;
  position: relative;
  background-color: #000000;
  min-height: 0;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  width: 100%;
  pointer-events: auto;
}

.empty-state {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #000000;
}

.empty-content {
  text-align: center;
  color: #666666;
}

.empty-icon {
  font-size: 48px;
  margin-bottom: 16px;
  color: #404040;
}

.empty-text {
  font-size: 14px;
  color: #666666;
  margin: 0;
}

.webview {
  width: 100%;
  height: 100%;
  border: none;
  display: block;
  flex: 1;
  min-height: 0;
  pointer-events: auto;
  position: relative;
  z-index: 1;
}

.loading-overlay {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  z-index: 100;
  color: #cccccc;
  background-color: rgba(43, 43, 43, 0.8);
  padding: 16px;
  border-radius: 4px;
  pointer-events: none;
}

.loading-icon {
  font-size: 24px;
  animation: spin 1s linear infinite;
  color: #409eff;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

/* 响应式设计 */
@media (max-width: 600px) {
  .toolbar {
    flex-wrap: wrap;
    padding: 4px 0;
  }
  
  .nav-buttons {
    order: 1;
  }
  
  .address-bar {
    order: 2;
    width: 100%;
    margin-top: 4px;
    margin-left: 8px;
  }
}

@media (max-width: 400px) {
  .nav-buttons .el-button {
    min-width: 28px;
    height: 24px;
    padding: 0 4px;
  }
  
  .toolbar {
    padding: 4px 0;
    gap: 4px;
  }
}


</style>
