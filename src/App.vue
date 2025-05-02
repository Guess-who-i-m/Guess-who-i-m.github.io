<template>

  <div class="container">
    
    <div class="app-header">
      <div class="app-header-content">
        <svg class="github-icon" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
          <path d="M12 0C5.37 0 0 5.37 0 12c0 5.3 3.438 9.8 8.205 11.387.6.113.82-.258.82-.577 0-.285-.01-1.04-.015-2.04-3.338.724-4.042-1.61-4.042-1.61-.546-1.387-1.333-1.756-1.333-1.756-1.09-.745.083-.73.083-.73 1.205.085 1.838 1.236 1.838 1.236 1.07 1.835 2.809 1.305 3.495.998.108-.776.417-1.305.76-1.605-2.665-.3-5.466-1.332-5.466-5.93 0-1.31.465-2.38 1.235-3.22-.135-.303-.54-1.523.105-3.176 0 0 1.005-.322 3.3 1.23.96-.267 1.98-.399 3-.405 1.02.006 2.04.138 3 .405 2.28-1.552 3.285-1.23 3.285-1.23.645 1.653.24 2.873.12 3.176.765.84 1.23 1.91 1.23 3.22 0 4.61-2.805 5.625-5.475 5.92.42.36.81 1.096.81 2.22 0 1.606-.015 2.896-.015 3.286 0 .315.21.69.825.57C20.565 21.795 24 17.295 24 12c0-6.63-5.37-12-12-12"/>
        </svg>
        <h1>GitHub 文件目录下载</h1>
      </div>
    </div>

    
    
    <div class="card main-card">
      <div class="card-body">
        <div class="input-group">
          <input 
            v-model="githubUrl" 
            placeholder="请输入GitHub文件或目录URL (例如: https://github.com/username/repo/tree/branch/path)" 
            class="url-input"
          />
          <button @click="downloadFiles" :disabled="isLoading" class="download-btn">
            <svg v-if="isLoading" class="loading-icon" viewBox="0 0 24 24">
              <circle class="spinner" cx="12" cy="12" r="10" fill="none" stroke-width="3" />
            </svg>
            <span v-if="!isLoading">
              <svg class="download-icon" viewBox="0 0 24 24">
                <path d="M19 9h-4V3H9v6H5l7 7 7-7zM5 18v2h14v-2H5z"/>
              </svg>
              下载
            </span>
            <span v-else>下载中...</span>
          </button>
        </div>
        
        <div v-if="error" class="error-message">
          <svg class="message-icon" viewBox="0 0 24 24">
            <path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 15h-2v-2h2v2zm0-4h-2V7h2v6z"/>
          </svg>
          <span>{{ error }}</span>
        </div>
      </div>
    </div>
    

    <div v-if="isLoading" class="card loading-card">
      <div class="card-body">
        <div class="loading-info">
          <div class="progress-container">
            <div class="progress-bar">
              <div class="progress" :style="{width: `${downloadProgress}%`}"></div>
            </div>
            <div class="progress-text">{{ downloadProgress.toFixed(1) }}%</div>
          </div>
          <div class="status-text">正在下载文件 ({{ downloadedFiles }}/{{ totalFiles }})</div>
        </div>
      </div>
    </div>
    
    <div v-if="logs.length > 0" class="card logs-card">
      <div class="card-header">
        <svg class="log-header-icon" viewBox="0 0 24 24">
          <path d="M19 3h-4.18C14.4 1.84 13.3 1 12 1c-1.3 0-2.4.84-2.82 2H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zm-7 0c.55 0 1 .45 1 1s-.45 1-1 1-1-.45-1-1 .45-1 1-1zm0 4c1.66 0 3 1.34 3 3s-1.34 3-3 3-3-1.34-3-3 1.34-3 3-3zm6 12H6v-1.4c0-2 4-3.1 6-3.1s6 1.1 6 3.1V19z"/>
        </svg>
        <h3>下载日志</h3>
      </div>
      <div class="card-body logs-container">
        <div v-for="(log, index) in logs" :key="index" class="log-item" :class="{'fade-in': index === 0}">
          <span class="log-time">{{ log.split(']')[0].substring(1) }}</span>
          <span class="log-message">{{ log.split(']')[1] }}</span>
        </div>
      </div>
    </div>

  </div>
</template>

<script>
import axios from 'axios';
import JSZip from 'jszip';
export default {
  name: 'App',
  data() {
    return {
      githubUrl: '',
      isLoading: false,
      error: '',
      logs: [],
      zip: null,
      downloadProgress: 0,
      totalFiles: 0,
      downloadedFiles: 0
    };
  },
  methods: {
    async downloadFiles() {
      if (!this.githubUrl) {
        this.error = "请输入GitHub URL";
        return;
      }
      
      this.isLoading = true;
      this.error = '';
      this.logs = [];
      this.zip = new JSZip();
      this.downloadProgress = 0;
      this.totalFiles = 0;
      this.downloadedFiles = 0;
      
      try {
        // 解析GitHub URL
        const { owner, repo, branch, path } = this.parseGithubUrl(this.githubUrl);
        
        // 计算总文件数
        await this.countFiles(owner, repo, branch, path);
        
        // 获取并下载文件
        await this.fetchAndDownloadFiles(owner, repo, branch, path);
        
        // 生成ZIP并触发下载
        this.addLog('正在生成ZIP文件...');
        const zipContent = await this.zip.generateAsync({type: 'blob'});
        const downloadLink = document.createElement('a');
        downloadLink.href = URL.createObjectURL(zipContent);
        downloadLink.download = `${repo}${path ? '-' + path.replace(/\//g, '-') : ''}.zip`;
        downloadLink.click();
        
        this.addLog('下载完成!');
      } catch (err) {
        this.error = `发生错误: ${err.message}`;
        console.error('Error:', err);
      } finally {
        this.isLoading = false;
      }
    },
    
    parseGithubUrl(url) {
      try {
        // 提取 GitHub URL 的各个部分
        const urlPattern = /https:\/\/github\.com\/([^\/]+)\/([^\/]+)(?:\/(?:tree|blob)\/([^\/]+)\/?(.*)|)/;
        const matches = url.match(urlPattern);
        
        if (!matches) {
          throw new Error('无效的 GitHub URL');
        }
        
        return {
          owner: matches[1],
          repo: matches[2],
          branch: matches[3] || 'main',
          path: matches[4] || ''
        };
      } catch (err) {
        throw new Error('解析 URL 时出错: ' + err.message);
      }
    },
    
    async countFiles(owner, repo, branch, path) {
      const apiUrl = `https://api.github.com/repos/${owner}/${repo}/contents/${path}?ref=${branch}`;
      
      try {
        const response = await axios.get(apiUrl);
        const contents = Array.isArray(response.data) ? response.data : [response.data];
        
        for (const item of contents) {
          if (item.type === 'file') {
            this.totalFiles++;
          } else if (item.type === 'dir') {
            await this.countFiles(owner, repo, branch, item.path);
          }
        }
      } catch (err) {
        console.log(err);
        throw new Error(`计算文件数量出错: ${err.message}`);
      }
    },
    
    async fetchAndDownloadFiles(owner, repo, branch, path) {
      const apiUrl = `https://api.github.com/repos/${owner}/${repo}/contents/${path}?ref=${branch}`;
      
      this.addLog(`获取路径内容: ${path || 'root'}`);
      
      try {
        const response = await axios.get(apiUrl);
        const contents = Array.isArray(response.data) ? response.data : [response.data];
        
        for (const item of contents) {
          if (item.type === 'file') {
            await this.downloadFile(item.path, item.download_url);
          } else if (item.type === 'dir') {
            await this.fetchAndDownloadFiles(owner, repo, branch, item.path);
          }
        }
      } catch (err) {
        throw new Error(`获取目录内容出错: ${err.message}`);
      }
    },
    
    async downloadFile(path, url) {
      this.addLog(`下载文件: ${path}`);
      try {
        const response = await axios.get(url, { responseType: 'arraybuffer' });
        this.zip.file(path, response.data);
        this.downloadedFiles++;
        this.downloadProgress = (this.downloadedFiles / this.totalFiles) * 100;
      } catch (err) {
        this.addLog(`下载 ${path} 失败: ${err.message}`);
      }
    },
    
    addLog(message) {
      this.logs.unshift(`[${new Date().toLocaleTimeString()}] ${message}`);
    }
  }
};
</script>

<style>
:root {
  --primary-color: #2563eb;
  --primary-hover: #1d4ed8;
  --primary-light: rgba(37, 99, 235, 0.1);
  --success-color: #10b981;
  --error-color: #ef4444;
  --text-color: #1f2937;
  --text-secondary: #6b7280;
  --text-light: #9ca3af;
  --light-gray: #f3f4f6;
  --medium-gray: #e5e7eb;
  --dark-gray: #9ca3af;
  --bg-color: #f9fafb;
  --card-bg: #ffffff;
  --border-radius: 10px;
  --box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
  --transition: all 0.3s ease;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  line-height: 1.6;
  color: var(--text-color);
  background-color: var(--bg-color);
  margin: 0;
  padding: 0;
  min-height: 100vh; /* Ensures body takes at least full viewport height */
  display: flex;
  flex-direction: column;
  align-items: center;    /* Centers horizontally (cross axis) */
  justify-content: center; /* Add this: Centers vertically (main axis) */
  flex-grow: 1; /* 填满剩余空间 */
  
  width: 100%;
  /* width: auto; */ /* Optional: You might want width: 100%; or remove it, auto is less common here */
}


/* .container {
  max-width: 900px;
  width: 95%;
  margin: 2rem auto;
  padding: 0;
  
} */

.container {
  width: 900px;
  margin: 2rem auto;
  padding: 0;
  display: flex;           /* 设置为 Flex 容器 */
  flex-direction: column;  /* 子元素垂直排列 */
  align-items: stretch;    /* 子元素在水平方向拉伸以填充容器 */
}


/* *{
    outline: solid #f00 1px !important;
    background: #000 !important;
    color: #fff !important;
} */


/* .app-header {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 16px;
  margin-bottom: 36px;
  animation: fadeInDown 0.6s ease;
} */

.github-icon {
  width: 40px;
  height: 40px;
  fill: var(--primary-color);
}

h1 {
  color: var(--text-color);
  font-size: 32px;
  font-weight: 700;
  margin: 0;
  text-align: center;
  background: linear-gradient(135deg, var(--primary-color), #4f46e5);
  background-clip: text;
  -webkit-text-fill-color: transparent;
  width: 100%; /* 明确设置宽度 */
}

/* .card {
  background-color: var(--card-bg);
  border-radius: var(--border-radius);
  box-shadow: var(--box-shadow);
  margin-bottom: 28px;
  overflow: hidden;
  transition: var(--transition);
  border: 1px solid var(--medium-gray);
  animation: fadeInUp 0.5s ease;
} */

/* .app-header {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 16px;
  margin-bottom: 36px;
  animation: fadeInDown 0.6s ease;
  width: 100%;
  box-sizing: border-box; 
} */

.app-header {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0; /* 移除默认的间隙 */
  margin-bottom: 36px;
  animation: fadeInDown 0.6s ease;
  width: 100%;
  box-sizing: border-box;
}
/* 添加一个内部容器来包裹图标和标题 */
.app-header-content {
  display: flex;
  align-items: center;
  gap: 8px; /* 这里设置你想要的图标和标题之间的间距 */
}

.card {
  background-color: var(--card-bg);
  border-radius: var(--border-radius);
  box-shadow: var(--box-shadow);
  margin-bottom: 28px;
  overflow: hidden;
  transition: var(--transition);
  border: 1px solid var(--medium-gray);
  animation: fadeInUp 0.5s ease;
  width: 100%; /* 明确设置宽度 */
  box-sizing: border-box; /* 推荐加上 */
}


.card:hover {
  box-shadow: 0 10px 25px -5px rgba(0, 0, 0, 0.1), 0 10px 10px -5px rgba(0, 0, 0, 0.04);
  transform: translateY(-2px);
}

.main-card {
  border-top: 4px solid var(--primary-color);
}

.card-header {
  display: flex;
  align-items: center;
  padding: 18px 24px;
  border-bottom: 1px solid var(--medium-gray);
  background-color: var(--light-gray);
}

.log-header-icon {
  width: 24px;
  height: 24px;
  margin-right: 12px;
  fill: var(--primary-color);
}

.card-header h3 {
  margin: 0;
  color: var(--text-color);
  font-size: 18px;
  font-weight: 600;
}

.card-body {
  padding: 24px;
}

.input-group {
  display: flex;
  position: relative;
  border-radius: var(--border-radius);
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05);
}

.url-input {
  flex: 1;
  padding: 16px 20px;
  font-size: 16px;
  color: var(--text-color);
  background-color: var(--light-gray);
  border: 2px solid var(--medium-gray);
  border-right: none;
  border-radius: var(--border-radius) 0 0 var(--border-radius);
  outline: none;
  transition: var(--transition);
}

.url-input:focus {
  border-color: var(--primary-color);
  background-color: var(--card-bg);
  box-shadow: 0 0 0 4px rgba(37, 99, 235, 0.2);
}

.url-input::placeholder {
  color: var(--text-light);
  opacity: 0.7;
}

.download-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: 120px;
  padding: 0 24px;
  font-size: 16px;
  font-weight: 600;
  color: white;
  background-color: var(--primary-color);
  border: none;
  border-radius: 0 var(--border-radius) var(--border-radius) 0;
  cursor: pointer;
  transition: var(--transition);
  outline: none;
}

.download-btn:hover, .download-btn:focus {
  background-color: var(--primary-hover);
  box-shadow: 0 4px 12px rgba(37, 99, 235, 0.25);
}

.download-btn:disabled {
  opacity: 0.7;
  cursor: not-allowed;
  background-color: var(--dark-gray);
}

.download-icon {
  width: 20px;
  height: 20px;
  fill: white;
  margin-right: 8px;
}

.loading-icon {
  width: 24px;
  height: 24px;
  animation: spin 1.2s linear infinite;
}

.spinner {
  stroke: white;
  stroke-dasharray: 60, 150;
  stroke-dashoffset: 0;
  animation: dash 1.5s ease-in-out infinite;
}

.error-message {
  display: flex;
  align-items: center;
  margin-top: 16px;
  padding: 12px 16px;
  background-color: rgba(239, 68, 68, 0.1);
  border-left: 4px solid var(--error-color);
  border-radius: 4px;
  color: var(--error-color);
  animation: fadeIn 0.3s ease;
}

.message-icon {
  width: 20px;
  height: 20px;
  margin-right: 10px;
  fill: var(--error-color);
}

.loading-card {
  background: linear-gradient(135deg, var(--card-bg), #f5faff);
}

.loading-info {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.progress-container {
  display: flex;
  align-items: center;
  gap: 16px;
}

.progress-bar {
  flex: 1;
  height: 12px;
  background-color: var(--light-gray);
  border-radius: 10px;
  overflow: hidden;
}

.progress {
  height: 100%;
  background: linear-gradient(90deg, var(--primary-color), #4f46e5);
  border-radius: 10px;
  transition: width 0.3s ease;
}

.progress-text {
  font-size: 16px;
  font-weight: 600;
  color: var(--primary-color);
  min-width: 60px;
  text-align: right;
}

.status-text {
  font-size: 15px;
  color: var(--text-secondary);
  text-align: center;
}

.logs-container {
  max-height: 300px;
  overflow-y: auto;
  padding: 16px;
}

.log-item {
  display: flex;
  margin-bottom: 10px;
  padding: 10px 12px;
  border-radius: 6px;
  background-color: var(--light-gray);
  border-left: 3px solid var(--primary-color);
  font-size: 14px;
  line-height: 1.5;
}

.log-time {
  color: var(--text-secondary);
  margin-right: 10px;
  font-family: monospace;
  white-space: nowrap;
}

.log-message {
  color: var(--text-color);
}

.fade-in {
  animation: fadeIn 0.3s ease;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(5px); }
  to { opacity: 1; transform: translateY(0); }
}

@keyframes fadeInUp {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

@keyframes fadeInDown {
  from { opacity: 0; transform: translateY(-10px); }
  to { opacity: 1; transform: translateY(0); }
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

@keyframes dash {
  0% {
    stroke-dasharray: 1, 150;
    stroke-dashoffset: 0;
  }
  50% {
    stroke-dasharray: 90, 150;
    stroke-dashoffset: -35;
  }
  100% {
    stroke-dasharray: 90, 150;
    stroke-dashoffset: -120;
  }
}

/* 响应式设计 */
/* @media (max-width: 768px) {
  .card-body {
    padding: 20px 16px;
  }
  
  .input-group {
    flex-direction: column;
  }
  
  .url-input {
    border-right: 2px solid var(--medium-gray);
    border-radius: var(--border-radius) var(--border-radius) 0 0;
  }
  
  .download-btn {
    width: 100%;
    border-radius: 0 0 var(--border-radius) var(--border-radius);
    padding: 14px;
  }
} */
</style>
