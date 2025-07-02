<script setup lang="ts">
import { ref } from 'vue';
import { ElMessage } from 'element-plus';
import { 
  Refresh, 
  ArrowLeft, 
  ArrowRight, 
  HomeFilled,
  Search
} from '@element-plus/icons-vue';

// 响应式数据
const currentUrl = ref('');
const inputUrl = ref('');
const webviewRef = ref<HTMLElement>();
const loading = ref(false);
const canGoBack = ref(false);
const canGoForward = ref(false);
const hasContent = ref(false);



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
  
  currentUrl.value = url;
  inputUrl.value = url;
  loading.value = true;
  hasContent.value = true;
  
  // 更新webview src
  if (webviewRef.value) {
    (webviewRef.value as any).src = url;
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
    (webviewRef.value as any).reload();
  }
};

// 后退
const goBack = () => {
  if (webviewRef.value && canGoBack.value) {
    (webviewRef.value as any).goBack();
  }
};

// 前进
const goForward = () => {
  if (webviewRef.value && canGoForward.value) {
    (webviewRef.value as any).goForward();
  }
};

// 回到首页
const goHome = () => {
  inputUrl.value = 'https://www.google.com';
  loadUrl();
};

// webview事件处理
const onWebviewLoad = () => {
  loading.value = false;
  if (webviewRef.value) {
    const webview = webviewRef.value as any;
    canGoBack.value = webview.canGoBack();
    canGoForward.value = webview.canGoForward();
    currentUrl.value = webview.getURL();
    inputUrl.value = webview.getURL();
  }
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
        sandbox="allow-same-origin allow-scripts allow-forms allow-popups allow-top-navigation"
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
  background-color: #2b2b2b;
  color: #cccccc;
  overflow: hidden;
  position: relative;
}

.toolbar {
  display: flex;
  align-items: center;
  padding: 6px 8px;
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
  z-index: 10;
  color: #cccccc;
  background-color: rgba(43, 43, 43, 0.8);
  padding: 16px;
  border-radius: 4px;
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
    padding: 4px 6px;
  }
  
  .nav-buttons {
    order: 1;
  }
  
  .address-bar {
    order: 2;
    width: 100%;
    margin-top: 4px;
  }
}

@media (max-width: 400px) {
  .nav-buttons .el-button {
    min-width: 28px;
    height: 24px;
    padding: 0 4px;
  }
  
  .toolbar {
    padding: 4px;
    gap: 4px;
  }
}


</style>
