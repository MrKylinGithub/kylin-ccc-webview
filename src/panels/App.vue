<script setup lang="ts">
import { ref, nextTick, computed, onMounted } from 'vue';
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
const previewUrl = ref('192.168.0.104:7456');

// 获取Cocos Creator预览地址
const getPreviewUrl = async () => {
  try {
    // 尝试多种方式获取预览配置
    let port = null;
    let ip = null;
    
    // 方法1: 尝试从preview配置获取
    try {
      port = await Editor.Profile.getProject('preview', 'port');
      ip = await Editor.Profile.getConfig('general', 'preview-ip');
    } catch (e) {
      console.log('方法1获取失败:', e);
    }
    
    // 方法2: 尝试从项目配置获取
    if (!port) {
      try {
        const previewConfig = await Editor.Profile.getProject('preview', '');
        if (previewConfig && previewConfig.port) {
          port = previewConfig.port;
        }
      } catch (e) {
        console.log('方法2获取失败:', e);
      }
    }
    
    // 方法3: 尝试从偏好设置获取
    if (!port) {
      try {
        port = await Editor.Profile.getConfig('preferences', 'preview.port');
        if (!ip) {
          ip = await Editor.Profile.getConfig('preferences', 'preview.ip');
        }
      } catch (e) {
        console.log('方法3获取失败:', e);
      }
    }
    
    // 方法4: 尝试获取当前编辑器版本信息（可能包含预览信息）
    if (!port) {
      try {
        // 可能的配置位置
        const configs = [
          ['general', 'preview-port'],
          ['general', 'previewPort'],
          ['preview', 'serverPort'],
          ['server', 'port']
        ];
        
        for (const [section, key] of configs) {
          try {
            const value = await Editor.Profile.getConfig(section, key);
            if (value) {
              port = value;
              break;
            }
          } catch (e) {
            // 继续尝试下一个
          }
        }
      } catch (e) {
        console.log('方法4获取失败:', e);
      }
    }
    
    // 设置默认IP（如果没有获取到）
    if (!ip) {
      ip = '192.168.0.104';
    }
    
    // 设置默认端口（如果没有获取到）
    if (!port) {
      port = 7456;
    }
    
    previewUrl.value = `${ip}:${port}`;
    console.log('获取到的预览地址:', previewUrl.value);
    
  } catch (error) {
    console.log('获取预览端口配置失败，使用默认值:', error);
    previewUrl.value = '192.168.0.104:7456';
  }
};

// 常用网站快捷按钮
const quickSites = computed(() => [
  { name: 'Cocos预览', url: `http://${previewUrl.value}` },
  { name: 'Google', url: 'https://www.google.com' },
  { name: 'GitHub', url: 'https://github.com' },
  { name: 'CocosCreator文档', url: 'https://docs.cocos.com/creator/manual/zh/' },
  { name: 'Vue.js', url: 'https://vuejs.org' },
  { name: 'Element Plus', url: 'https://element-plus.org/zh-CN/' }
]);

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

// 快速访问网站
const visitQuickSite = (url: string) => {
  inputUrl.value = url;
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

// 组件挂载时获取预览地址
onMounted(() => {
  getPreviewUrl();
});
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

    <!-- 快捷网站 -->
    <div class="quick-sites">
      <span class="quick-label">快速访问：</span>
      <div class="preview-info">
        <span class="preview-label">预览地址: {{ previewUrl }}</span>
        <el-button
          @click="getPreviewUrl"
          size="small"
          type="text"
          class="refresh-btn"
          :icon="Refresh"
          title="刷新预览地址"
        />
      </div>
      <el-button
        v-for="site in quickSites"
        :key="site.name"
        @click="visitQuickSite(site.url)"
        size="small"
        type="text"
        class="quick-site-btn"
      >
        {{ site.name }}
      </el-button>
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
  height: 100vh;
  width: 100%;
  background-color: #2b2b2b;
  color: #cccccc;
  overflow: hidden;
}

.toolbar {
  display: flex;
  align-items: center;
  padding: 6px 8px;
  background-color: #3c3c3c;
  border-bottom: 1px solid #525252;
  gap: 8px;
}

.nav-buttons {
  display: flex;
  gap: 4px;
}

.nav-buttons .el-button {
  background-color: #4a4a4a;
  border-color: #606060;
  color: #cccccc;
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
  max-width: 600px;
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

.quick-sites {
  display: flex;
  align-items: center;
  padding: 4px 8px;
  background-color: #353535;
  border-bottom: 1px solid #525252;
  gap: 8px;
  overflow-x: auto;
}

.quick-label {
  font-size: 12px;
  color: #999999;
  white-space: nowrap;
}

.preview-info {
  display: flex;
  align-items: center;
  gap: 4px;
  margin-right: 8px;
}

.preview-label {
  font-size: 12px;
  color: #66b3ff;
  white-space: nowrap;
}

.refresh-btn {
  font-size: 12px;
  padding: 2px 4px !important;
  min-width: auto !important;
  color: #409eff !important;
}

.refresh-btn:hover {
  color: #66b3ff !important;
  background-color: #404040 !important;
}

.quick-site-btn {
  font-size: 12px;
  padding: 2px 8px !important;
  white-space: nowrap;
  color: #409eff !important;
}

.quick-site-btn:hover {
  color: #66b3ff !important;
  background-color: #404040 !important;
}

.webview-wrapper {
  flex: 1;
  position: relative;
  background-color: #000000;
  min-height: 0;
  overflow: hidden;
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
  }
  
  .address-bar {
    order: 2;
    width: 100%;
    margin-top: 4px;
  }
  
  .quick-sites {
    padding: 2px 8px;
  }
}

/* 滚动条样式 */
.quick-sites::-webkit-scrollbar {
  height: 4px;
}

.quick-sites::-webkit-scrollbar-track {
  background: #2b2b2b;
}

.quick-sites::-webkit-scrollbar-thumb {
  background: #606060;
  border-radius: 2px;
}

.quick-sites::-webkit-scrollbar-thumb:hover {
  background: #707070;
}
</style>
