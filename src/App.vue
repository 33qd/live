<script setup lang="ts">
import { reactive, onMounted, onBeforeUnmount, nextTick } from "vue";
import Hls from "hls.js";
import { Core } from "p2p-media-loader-core";
import { HlsJsP2PEngine } from "p2p-media-loader-hlsjs";
// 确保在 main.ts/main.js 里 import '@fortawesome/fontawesome-free/css/all.min.css'

// ==================== 常量配置 ====================
// 默认视频流地址 - 只执行一次，避免热重载时重复初始化
const DEFAULT_STREAM =
  "https://pull-hssh1.vzan.com/1455530897/257698490561841415/replay.1749293795.66814879.m3u8";
// 默认Tracker服务器列表，用于P2P网络发现
const DEFAULT_TRACKERS = Core.DEFAULT_STREAM_CONFIG.announceTrackers.join(",");

// ==================== 播放器状态变量 ====================
// 视频播放器DOM元素引用
let videoPlayer: HTMLVideoElement | null = null;
// 当前播放状态：true=播放中，false=暂停中
let isPlaying = false;
// 当前选择的清晰度等级（-1自动，0及以上表示具体等级）
let selectedQuality = -1;
// Hls.js实例，负责视频流解析和播放
let hls: any = null;
// P2P引擎实例，负责P2P网络连接和数据传输
let p2pEngine: any = null;
// 当前连接的P2P节点ID列表
let peers: string[] = [];
// 当前节点的唯一标识符（前8位）
let peerId = "";
// Tracker服务器列表，用于P2络节点发现
let trackers = DEFAULT_TRACKERS.split(",");
// Tracker服务器输入框的值
let trackersInput = DEFAULT_TRACKERS;
// 当前播放的视频流URL
let streamUrl = DEFAULT_STREAM;

// ==================== 响应式数据对象 ====================
// 实时下载统计（每秒重置）
const downloadStats = reactive({
  httpDownloaded: 0, // HTTP下载字节数
  p2pDownloaded: 0, // P2P下载字节数
  p2pUploaded: 0, // P2上传字节数
});

// 累计存储的统计数据
const storedData = reactive({
  totalDownloaded: 0, // 总下载量（MiB）
  httpDownloaded: 0, // HTTP累计下载量（MiB）
  p2pDownloaded: 0, // P2量（MiB）
  p2pUploaded: 0, // P2累计上传量（MiB）
});

// 当前实时速率统计（Mbit/s）
const currentStats = reactive({
  httpDownloaded: 0, // HTTP下载速率
  p2pDownloaded: 0, // P2下载速率
  p2pUploaded: 0, // P2P上传速率
});

// ==================== 计算属性函数 ====================
// 获取总下载量（MiB）
const totalDownloaded = () => storedData.totalDownloaded;
// 获取HTTP下载量（MiB）
const httpDownloaded = () => storedData.httpDownloaded;
// 获取P2P下载量（MiB）
const p2pDownloaded = () => storedData.p2pDownloaded;
// 获取P2P上传量（MiB）
const p2pUploaded = () => storedData.p2pUploaded;

// 计算HTTP下载占比百分比
const httpPercentage = () => {
  const total = storedData.httpDownloaded + storedData.p2pDownloaded;
  return total > 0 ? (storedData.httpDownloaded / total) * 100 : 0;
};

// 计算P2P下载占比百分比
const p2pPercentage = () => {
  const total = storedData.httpDownloaded + storedData.p2pDownloaded;
  return total > 0 ? (storedData.p2pDownloaded / total) * 100 : 0;
};

// ==================== 工具函数 ====================
// 将字节转换为MiB（兆字节）
const convertToMiB = (bytes: number) => bytes / 1024 / 1024;
// 将字节转换为Mbit（兆比特，用于显示速率）
const convertToMbit = (bytes: number) => (bytes * 8) / 1000000;

// ==================== 核心播放器功能 ====================
/**
 * 初始化视频播放器
 * 配置Hls.js和P2P引擎，设置事件监听器
 */
const initPlayer = () => {
  // 检查浏览器是否支持HLS
  if (!Hls.isSupported() || !videoPlayer) {
    console.error("HLS is not supported in this browser");
    return;
  }

  // 销毁之前的Hls实例，避免内存泄漏
  if (hls) hls.destroy();

  // 兼容开发环境热重载：每次都动态注入mixin，避免原型污染导致卡住
  const HlsWithP2P = HlsJsP2PEngine.injectMixin(Hls);

  // 创建新的Hls实例，配置P2功能
  hls = new HlsWithP2P({
    p2p: {
      core: {
        announceTrackers: trackers, // P2P网络发现服务器
        swarmId: "vue-p2p-demo", // 流媒体唯一标识
      },
      // P2擎创建完成后的回调
      onHlsJsCreated(engine: any) {
        p2pEngine = engine.p2pEngine;
        // 获取当前节点ID的前8位作为显示标识
        peerId = (p2pEngine?.peerId ?? "").substr(0, 8);

        // 监听P2P节点连接事件
        p2pEngine.addEventListener("onPeerConnect", (params: any) => {
          if (params.streamType === "main") {
            peers.push(params.peerId.substr(0, 8));
          }
        });

        // 监听P2P节点断开事件
        p2pEngine.addEventListener("onPeerClose", (params: any) => {
          if (params.streamType === "main") {
            peers = peers.filter(
              (peer: string) => peer !== params.peerId.substr(0, 8)
            );
          }
        });

        // 监听数据块下载事件，统计下载来源
        p2pEngine.addEventListener(
          "onChunkDownloaded",
          (bytesLength: number, downloadSource: string) => {
            if (downloadSource === "http")
              downloadStats.httpDownloaded += bytesLength;
            else if (downloadSource === "p2p")
              downloadStats.p2pDownloaded += bytesLength;
          }
        );

        // 监听数据块上传事件，统计P2P贡献
        p2pEngine.addEventListener("onChunkUploaded", (bytesLength: number) => {
          downloadStats.p2pUploaded += bytesLength;
        });
      },
    },
  });

  // 将Hls实例绑定到视频元素
  hls.attachMedia(videoPlayer);
  // 加载视频流
  hls.loadSource(streamUrl);

  // 监听清单解析完成事件
  hls.on(Hls.Events.MANIFEST_PARSED, () => {
    // 获取可用的清晰度等级列表
    // qualityLevels = hls.levels; // This line was commented out in the original file, so it's commented out here.
  });

  // 监听Hls错误事件
  hls.on(Hls.Events.ERROR, (data: any) => {
    console.error("Hls.js error:", data);
  });

  // 监听视频播放事件
  videoPlayer.addEventListener("play", () => {
    isPlaying = true;
  });

  // 监听视频暂停事件
  videoPlayer.addEventListener("pause", () => {
    isPlaying = false;
  });
};

/**
 * 切换播放/暂停状态
 */
const togglePlay = () => {
  if (!videoPlayer) return;
  if (isPlaying) videoPlayer.pause();
  else
    videoPlayer.play().catch((e) => {
      console.error("播放失败:", e);
    });
};

/**
 * 重新启动播放器
 * 清理P2重新初始化播放器
 */
const restartPlayer = () => {
  if (hls) hls.destroy();
  peers = [];
  initPlayer();
  videoPlayer?.play();
};

/**
 * 切换视频清晰度
 */
const changeQuality = () => {
  if (hls) hls.currentLevel = parseInt(selectedQuality as any);
};

/**
 * 更新播放器配置
 * 重新解析tracker服务器列表并重启播放器
 */
const updatePlayer = () => {
  trackers = trackersInput.split(",");
  restartPlayer();
};

/**
 * 重置为默认配置
 */
const resetDefaults = () => {
  streamUrl = DEFAULT_STREAM;
  trackersInput = DEFAULT_TRACKERS;
  updatePlayer();
};

// ==================== 生命周期钩子 ====================
// 统计更新定时器
let statsInterval: any = null;

/**
 * 组件挂载后的初始化
 */
onMounted(() => {
  nextTick(() => {
    // 通过DOM查询获取video元素
    videoPlayer = document.querySelector("video");
    // 初始化播放器
    initPlayer();

    // 设置统计更新定时器，每秒更新一次
    statsInterval = setInterval(() => {
      // 更新实时速率统计（Mbit/s）
      currentStats.httpDownloaded = convertToMbit(downloadStats.httpDownloaded);
      currentStats.p2pDownloaded = convertToMbit(downloadStats.p2pDownloaded);
      currentStats.p2pUploaded = convertToMbit(downloadStats.p2pUploaded);

      // 累加到存储数据（MiB）
      storedData.httpDownloaded += convertToMiB(downloadStats.httpDownloaded);
      storedData.p2pDownloaded += convertToMiB(downloadStats.p2pDownloaded);
      storedData.p2pUploaded += convertToMiB(downloadStats.p2pUploaded);

      // 计算总下载量
      storedData.totalDownloaded =
        storedData.httpDownloaded + storedData.p2pDownloaded;

      // 重置实时统计，准备下一秒的统计
      downloadStats.httpDownloaded = 0;
      downloadStats.p2pDownloaded = 0;
      downloadStats.p2pUploaded = 0;
    }, 1000);
  });
});

/**
 * 组件卸载前的清理工作
 */
onBeforeUnmount(() => {
  // 销毁Hls实例，释放资源
  if (hls) hls.destroy();
  // 清除统计定时器
  if (statsInterval) clearInterval(statsInterval);
});
</script>

<template>
  <div class="container">
    <header></header>
    <main class="dashboard">
      <div class="panel">
        <div class="panel-title">
          <i class="fas fa-play-circle"></i>
          <h2>视频播放器</h2>
        </div>
        <div class="video-container">
          <video controls></video>
        </div>
        <div class="controls">
          <div class="btn-group">
            <button class="btn btn-primary" @click="togglePlay">
              <i :class="[isPlaying ? 'fa-pause' : 'fa-play']" class="fas"></i>
              {{ isPlaying ? "暂停" : "播放" }}
            </button>
          </div>
        </div>
        <div class="stats-grid">
          <div class="stat-card">
            <i class="fas fa-cloud-download-alt"></i>
            <div class="stat-value">{{ totalDownloaded().toFixed(2) }} MiB</div>
            <div class="stat-label">总下载量</div>
          </div>
          <div class="stat-card">
            <i class="fas fa-server"></i>
            <div class="stat-value">{{ httpDownloaded().toFixed(2) }} MiB</div>
            <div class="stat-label">HTTP下载</div>
          </div>
          <div class="stat-card">
            <i class="fas fa-users"></i>
            <div class="stat-value">{{ p2pDownloaded().toFixed(2) }} MiB</div>
            <div class="stat-label">P2P下载</div>
          </div>
          <div class="stat-card">
            <i class="fas fa-cloud-upload-alt"></i>
            <div class="stat-value">{{ p2pUploaded().toFixed(2) }} MiB</div>
            <div class="stat-label">P2P上传</div>
          </div>
        </div>
        <div class="progress-bar">
          <div
            class="progress progress-http"
            :style="{ width: httpPercentage() + '%' }"
          ></div>
          <div
            class="progress progress-p2p"
            :style="{ width: p2pPercentage() + '%' }"
          ></div>
        </div>
        <div class="legend-container">
          <div class="legend-item">
            <div class="legend-color http-color"></div>
            <span
              >HTTP下载:
              {{ currentStats.httpDownloaded.toFixed(2) }} Mbit/s</span
            >
          </div>
          <div class="legend-item">
            <div class="legend-color p2p-download-color"></div>
            <span
              >P2P下载: {{ currentStats.p2pDownloaded.toFixed(2) }} Mbit/s</span
            >
          </div>
          <div class="legend-item">
            <div class="legend-color p2p-upload-color"></div>
            <span
              >P2P上传: {{ currentStats.p2pUploaded.toFixed(2) }} Mbit/s</span
            >
          </div>
        </div>
      </div>
      <div class="panel">
        <div class="panel-title">
          <i class="fas fa-network-wired"></i>
          <h2>p2p连接状态</h2>
          <span
            class="status-badge"
            :class="
              peers.length > 0 ? 'status-connected' : 'status-disconnected'
            "
          >
            {{ peers.length > 0 ? "已连接" : "未连接" }}
          </span>
        </div>
        <div class="network-info">
          <div class="stat-card">
            <i class="fas fa-user"></i>
            <div class="stat-value">Peer-{{ peerId }}</div>
            <div class="stat-label">您的节点ID</div>
          </div>
          <div class="stat-card">
            <i class="fas fa-link"></i>
            <div class="stat-value">{{ peers.length }}</div>
            <div class="stat-label">连接节点数</div>
          </div>
        </div>
        <div class="network-container" v-if="peers.length > 0">
          <h3><i class="fas fa-users"></i> 已连接节点</h3>
          <div class="peer-list">
            <div class="peer-card" v-for="(peer, index) in peers" :key="index">
              <div class="peer-icon">
                <i class="fas fa-laptop"></i>
              </div>
              <div>Peer-{{ peer }}</div>
            </div>
          </div>
        </div>
        <div v-else class="alert">
          <i class="fas fa-info-circle"></i>
          <div>正在等待节点连接... 请确保您的网络支持WebRTC</div>
        </div>
        <div class="trackers-container">
          <h3><i class="fas fa-server"></i> Tracker服务器</h3>
          <ul class="tracker-list">
            <li
              class="tracker-item"
              v-for="(tracker, index) in trackers"
              :key="index"
            >
              <i class="fas fa-circle"></i>
              <span>{{ tracker }}</span>
            </li>
          </ul>
        </div>
        <div class="input-group">
          <label for="streamUrl"><i class="fas fa-link"></i> 视频流URL</label>
          <input
            type="text"
            id="streamUrl"
            v-model="streamUrl"
            placeholder="输入视频流地址"
          />
        </div>
        <div class="input-group">
          <label for="trackers"
            ><i class="fas fa-list"></i> Tracker服务器 (逗号分隔)</label
          >
          <input
            type="text"
            id="trackers"
            v-model="trackersInput"
            placeholder="输入tracker服务器"
          />
        </div>
        <div class="btn-group">
          <button class="btn btn-primary" @click="updatePlayer">
            <i class="fas fa-sync"></i> 更新配置
          </button>
          <button class="btn btn-secondary" @click="resetDefaults">
            <i class="fas fa-undo"></i> 恢复默认
          </button>
        </div>
      </div>
    </main>
    <footer class="footer">
      <p>© 2023 Vue P2P视频播放器 | 基于WebRTC技术的分布式视频流解决方案</p>
    </footer>
  </div>
</template>

<style scoped>
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0;
  box-sizing: border-box;
  font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
  color: #000;
}

.tech-badges {
  display: flex;
  gap: 10px;
  margin: 10px 0 20px 0;
}

.tech-badge {
  background: #f3f3f3;
  color: #000;
  border-radius: 4px;
  padding: 4px 10px;
  font-size: 14px;
  display: flex;
  align-items: center;
  gap: 4px;
}

.dashboard {
  display: flex;
  flex-wrap: wrap;
  gap: 24px;
}

.panel {
  background: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
  padding: 24px;
  flex: 1 1 400px;
  min-width: 340px;
  margin-bottom: 24px;
}

.panel-title {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 16px;
}

.video-container {
  width: 100%;
  background: #000;
  border-radius: 8px;
  overflow: hidden;
  margin-bottom: 16px;
}

.video-container video {
  width: 100%;
  height: 360px;
  background: #000;
}

.controls {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 16px;
}

.quality-selector {
  padding: 4px 8px;
  border-radius: 4px;
  border: 1px solid #ddd;
  margin-right: 8px;
}

.btn-group {
  display: flex;
  gap: 8px;
}

.btn {
  padding: 6px 16px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 15px;
  display: flex;
  align-items: center;
  gap: 4px;
}

.btn-primary {
  background: #4caf50;
  color: #fff;
}

.btn-secondary {
  background: #f3f3f3;
  color: #000;
}

.stats-grid {
  display: flex;
  gap: 16px;
  margin: 16px 0;
}

.stat-card {
  background: #fafafa;
  border-radius: 6px;
  padding: 12px 16px;
  flex: 1 1 80px;
  text-align: center;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.03);
}

.stat-value {
  font-size: 20px;
  font-weight: bold;
  margin: 4px 0;
}

.stat-label {
  font-size: 13px;
  color: #000;
}

.progress-bar {
  width: 100%;
  height: 10px;
  background: #eee;
  border-radius: 5px;
  overflow: hidden;
  margin: 12px 0 20px 0;
  display: flex;
}

.progress {
  height: 100%;
  transition: width 0.3s;
}

.progress-http {
  background: #ffeb3b;
}

.progress-p2p {
  background: #ff9800;
}

.legend-container {
  display: flex;
  gap: 16px;
  margin-bottom: 12px;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 6px;
}

.legend-color {
  width: 14px;
  height: 14px;
  border-radius: 3px;
  display: inline-block;
}

.http-color {
  background: #ffeb3b;
}

.p2p-download-color {
  background: #ff9800;
}

.p2p-upload-color {
  background: #add8e6;
}

.status-badge {
  padding: 2px 10px;
  border-radius: 12px;
  font-size: 13px;
  margin-left: 12px;
  font-weight: bold;
}

.status-connected {
  background: rgba(76, 175, 80, 0.2);
  color: #4caf50;
}

.status-disconnected {
  background: rgba(244, 67, 54, 0.2);
  color: #f44336;
}

.network-info {
  display: flex;
  gap: 16px;
  margin: 16px 0;
}

.network-container {
  margin: 16px 0;
}

.peer-list {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.peer-card {
  background: #f3f3f3;
  border-radius: 6px;
  padding: 8px 12px;
  display: flex;
  align-items: center;
  gap: 6px;
}

.peer-icon {
  color: #888;
}

.alert {
  background: #fff3cd;
  color: #856404;
  border-radius: 4px;
  padding: 10px 16px;
  margin: 12px 0;
  display: flex;
  align-items: center;
  gap: 8px;
}

.trackers-container {
  margin: 18px 0 0 0;
}

.tracker-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.tracker-item {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 14px;
  margin-bottom: 2px;
}

.input-group {
  margin: 12px 0;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.input-group input {
  padding: 6px 10px;
  border-radius: 4px;
  border: 1px solid #ddd;
  font-size: 15px;
}

.footer {
  margin: 32px 0 0 0;
  text-align: center;
  color: #000;
  font-size: 14px;
}
</style>