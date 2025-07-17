<script setup lang="ts">
import {
  reactive,
  onMounted,
  onBeforeUnmount,
  nextTick,
} from "vue";
import Hls from "hls.js";
import { Core } from "p2p-media-loader-core";
import { HlsJsP2PEngine } from "p2p-media-loader-hlsjs";
// 确保在 main.ts/main.js 里 import '@fortawesome/fontawesome-free/css/all.min.css'

// 只执行一次
const DEFAULT_STREAM =
  "https://pull-hssh1.vzan.com/1455530897/257698490561841415/replay.1749293795.66814879.m3u8";
const DEFAULT_TRACKERS = Core.DEFAULT_STREAM_CONFIG.announceTrackers.join(",");

// 普通变量
let videoPlayer: HTMLVideoElement | null = null;
let isPlaying = false;
let selectedQuality = -1;
let qualityLevels: any[] = [];
let hls: any = null;
let p2pEngine: any = null;
let peers: string[] = [];
let peerId = "";
let trackers = DEFAULT_TRACKERS.split(",");
let trackersInput = DEFAULT_TRACKERS;
let streamUrl = DEFAULT_STREAM;

// 响应式对象
const downloadStats = reactive({
  httpDownloaded: 0,
  p2pDownloaded: 0,
  p2pUploaded: 0,
});
const storedData = reactive({
  totalDownloaded: 0,
  httpDownloaded: 0,
  p2pDownloaded: 0,
  p2pUploaded: 0,
});
const currentStats = reactive({
  httpDownloaded: 0,
  p2pDownloaded: 0,
  p2pUploaded: 0,
});

const totalDownloaded = () => storedData.totalDownloaded;
const httpDownloaded = () => storedData.httpDownloaded;
const p2pDownloaded = () => storedData.p2pDownloaded;
const p2pUploaded = () => storedData.p2pUploaded;
const httpPercentage = () => {
  const total = storedData.httpDownloaded + storedData.p2pDownloaded;
  return total > 0 ? (storedData.httpDownloaded / total) * 100 : 0;
};
const p2pPercentage = () => {
  const total = storedData.httpDownloaded + storedData.p2pDownloaded;
  return total > 0 ? (storedData.p2pDownloaded / total) * 100 : 0;
};

const convertToMiB = (bytes: number) => bytes / 1024 / 1024;
const convertToMbit = (bytes: number) => (bytes * 8) / 1000000;

const initPlayer = () => {
  if (!Hls.isSupported() || !videoPlayer) {
    console.error("HLS is not supported in this browser");
    return;
  }
  if (hls) hls.destroy();
  // 兼容 dev 热重载：每次都动态注入 mixin，避免原型污染导致卡住
  const HlsWithP2P = HlsJsP2PEngine.injectMixin(Hls);
  hls = new HlsWithP2P({
    p2p: {
      core: {
        announceTrackers: trackers,
        swarmId: "vue-p2p-demo",
      },
      onHlsJsCreated(engine: any) {
        p2pEngine = engine.p2pEngine;
        peerId = (p2pEngine?.peerId ?? "").substr(0, 8);
        p2pEngine.addEventListener("onPeerConnect", (params: any) => {
          if (params.streamType === "main") {
            peers.push(params.peerId.substr(0, 8));
          }
        });
        p2pEngine.addEventListener("onPeerClose", (params: any) => {
          if (params.streamType === "main") {
            peers = peers.filter((peer: string) => peer !== params.peerId.substr(0, 8));
          }
        });
        p2pEngine.addEventListener("onChunkDownloaded", (bytesLength: number, downloadSource: string) => {
          if (downloadSource === "http")
            downloadStats.httpDownloaded += bytesLength;
          else if (downloadSource === "p2p")
            downloadStats.p2pDownloaded += bytesLength;
        });
        p2pEngine.addEventListener("onChunkUploaded", (bytesLength: number) => {
          downloadStats.p2pUploaded += bytesLength;
        });
      },
    },
  });
  hls.attachMedia(videoPlayer);
  hls.loadSource(streamUrl);
  hls.on(Hls.Events.MANIFEST_PARSED, () => {
    qualityLevels = hls.levels;
  });
  hls.on(Hls.Events.ERROR, ( data: any) => {
    console.error("Hls.js error:", data);
  });
  videoPlayer.addEventListener("play", () => {
    isPlaying = true;
  });
  videoPlayer.addEventListener("pause", () => {
    isPlaying = false;
  });
};

const togglePlay = () => {
  if (!videoPlayer) return;
  if (isPlaying) videoPlayer.pause();
  else
    videoPlayer.play().catch((e) => {
      console.error("播放失败:", e);
    });
};
const restartPlayer = () => {
  if (hls) hls.destroy();
  peers = [];
  initPlayer();
  videoPlayer?.play();
};
const changeQuality = () => {
  if (hls)
    hls.currentLevel = parseInt(selectedQuality as any);
};
const updatePlayer = () => {
  trackers = trackersInput.split(",");
  restartPlayer();
};
const resetDefaults = () => {
  streamUrl = DEFAULT_STREAM;
  trackersInput = DEFAULT_TRACKERS;
  updatePlayer();
};

let statsInterval: any = null;
onMounted(() => {
  nextTick(() => {
    // 通过ref手动获取video元素
    videoPlayer = document.querySelector("video");
    initPlayer();
    statsInterval = setInterval(() => {
      currentStats.httpDownloaded = convertToMbit(downloadStats.httpDownloaded);
      currentStats.p2pDownloaded = convertToMbit(downloadStats.p2pDownloaded);
      currentStats.p2pUploaded = convertToMbit(downloadStats.p2pUploaded);
      storedData.httpDownloaded += convertToMiB(downloadStats.httpDownloaded);
      storedData.p2pDownloaded += convertToMiB(downloadStats.p2pDownloaded);
      storedData.p2pUploaded += convertToMiB(downloadStats.p2pUploaded);
      storedData.totalDownloaded =
        storedData.httpDownloaded + storedData.p2pDownloaded;
      downloadStats.httpDownloaded = 0;
      downloadStats.p2pDownloaded = 0;
      downloadStats.p2pUploaded = 0;
    }, 1000);
  });
});
onBeforeUnmount(() => {
  if (hls) hls.destroy();
  if (statsInterval) clearInterval(statsInterval);
});
</script>

<template>
  <div class="container">
    <header>
      <h1>Vue P2P视频播放器</h1>
      <p>
        基于Hls.js和P2P技术的分布式视频流解决方案，减少服务器带宽消耗，提高视频加载速度
      </p>
      <div class="tech-badges">
        <div class="tech-badge"><i class="fab fa-vuejs"></i> Vue 3</div>
        <div class="tech-badge"><i class="fas fa-wifi"></i> P2P技术</div>
        <div class="tech-badge"><i class="fas fa-video"></i> Hls.js</div>
        <div class="tech-badge"><i class="fas fa-bolt"></i> WebRTC</div>
      </div>
    </header>
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
          <select class="quality-selector" v-model="selectedQuality" @change="changeQuality">
            <option :value="-1">自动选择质量</option>
            <option v-for="(level, index) in qualityLevels" :value="index" :key="index">
              {{ level.height }}p ({{ Math.round(level.bitrate / 1000) }}kbps)
            </option>
          </select>
          <div class="btn-group">
            <button class="btn btn-primary" @click="togglePlay">
              <i :class="[isPlaying ? 'fa-pause' : 'fa-play']" class="fas"></i>
              {{ isPlaying ? "暂停" : "播放" }}
            </button>
            <button class="btn btn-secondary" @click="restartPlayer">
              <i class="fas fa-redo"></i> 重新开始
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
          <div class="progress progress-http" :style="{ width: httpPercentage() + '%' }"></div>
          <div class="progress progress-p2p" :style="{ width: p2pPercentage() + '%' }"></div>
        </div>
        <div class="legend-container">
          <div class="legend-item">
            <div class="legend-color http-color"></div>
            <span>HTTP下载:
              {{ currentStats.httpDownloaded.toFixed(2) }} Mbit/s</span>
          </div>
          <div class="legend-item">
            <div class="legend-color p2p-download-color"></div>
            <span>P2P下载: {{ currentStats.p2pDownloaded.toFixed(2) }} Mbit/s</span>
          </div>
          <div class="legend-item">
            <div class="legend-color p2p-upload-color"></div>
            <span>P2P上传: {{ currentStats.p2pUploaded.toFixed(2) }} Mbit/s</span>
          </div>
        </div>
      </div>
      <div class="panel">
        <div class="panel-title">
          <i class="fas fa-network-wired"></i>
          <h2>网络状态</h2>
          <span class="status-badge" :class="peers.length > 0 ? 'status-connected' : 'status-disconnected'">
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
            <li class="tracker-item" v-for="(tracker, index) in trackers" :key="index">
              <i class="fas fa-circle"></i>
              <span>{{ tracker }}</span>
            </li>
          </ul>
        </div>
        <div class="input-group">
          <label for="streamUrl"><i class="fas fa-link"></i> 视频流URL</label>
          <input type="text" id="streamUrl" v-model="streamUrl" placeholder="输入视频流地址" />
        </div>
        <div class="input-group">
          <label for="trackers"><i class="fas fa-list"></i> Tracker服务器 (逗号分隔)</label>
          <input type="text" id="trackers" v-model="trackersInput" placeholder="输入tracker服务器" />
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
@import "./assets/p2p-player.css";
</style>