<template>
  <div class="video-preview-container">
    <!-- 视频预览 -->
    <div class="video-preview p-4">
      <VideoPlayer
        ref="videoPlayerRef"
        v-if="videoUrl && videoData"
        :video="videoData"
        :dark-mode="darkMode"
        :autoplay="false"
        :volume="0.7"
        :muted="false"
        :loop="false"
        :custom-controls="[]"
        @play="handlePlay"
        @pause="handlePause"
        @error="handleError"
        @canplay="handleCanPlay"
        @ended="handleVideoEnded"
        @timeupdate="handleTimeUpdate"
        @fullscreen="handleFullscreen"
        @fullscreenExit="handleFullscreenExit"
        @ready="handlePlayerReady"
      />
      <div v-else class="loading-indicator text-center py-8">
        <div class="animate-spin rounded-full h-10 w-10 border-b-2 mx-auto" :class="darkMode ? 'border-primary-500' : 'border-primary-600'"></div>
        <p class="mt-2 text-sm" :class="darkMode ? 'text-gray-400' : 'text-gray-600'">{{ $t("mount.videoPreview.loadingVideo") }}</p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, ref, onMounted, onBeforeUnmount, watch } from "vue";
import { useI18n } from "vue-i18n";
import VideoPlayer from "../../common/VideoPlayer.vue";

const { t } = useI18n();

// Props 定义
const props = defineProps({
  // 文件信息
  file: {
    type: Object,
    required: true,
  },
  // 视频URL
  videoUrl: {
    type: String,
    default: null,
  },
  // 是否为深色模式
  darkMode: {
    type: Boolean,
    default: false,
  },
  // 是否为管理员
  isAdmin: {
    type: Boolean,
    default: false,
  },
  // 当前目录路径
  currentPath: {
    type: String,
    default: "",
  },
  // 目录项目列表
  directoryItems: {
    type: Array,
    default: () => [],
  },
});

// Emits 定义
const emit = defineEmits(["play", "pause", "error", "canplay", "loaded", "fullscreen", "fullscreenExit"]);

// 响应式数据
const videoPlayerRef = ref(null);
const isPlaying = ref(false);
const originalTitle = ref("");
const currentTime = ref(0);
const duration = ref(0);

// 当前视频数据（响应式）
const currentVideoData = ref(null);

// 为了兼容性，保留 videoData 计算属性
const videoData = computed(() => currentVideoData.value);

// 更新页面标题
const updatePageTitle = (playing = false, fileName = null) => {
  // 使用传入的文件名，如果没有则使用默认值
  const title = fileName || t("mount.videoPreview.videoPlayer");

  document.title = playing ? `🎬 ${title}` : `${title}`;
};

// 恢复原始页面标题
const restoreOriginalTitle = () => {
  if (originalTitle.value) {
    document.title = originalTitle.value;
  }
};

// 事件处理函数
const handlePlay = (data) => {
  isPlaying.value = true;
  const videoName = data?.video?.name;
  updatePageTitle(true, videoName);
  emit("play", data);
};

const handlePause = (data) => {
  isPlaying.value = false;
  const videoName = data?.video?.name;
  updatePageTitle(false, videoName);
  emit("pause", data);
};

const handleError = (error) => {
  // 忽略Service Worker相关的误报错误
  if (error?.target?.src?.includes(window.location.origin) && currentVideoData.value?.url) {
    console.log("🎬 忽略Service Worker相关的误报错误，视频实际可以正常播放");
    return;
  }

  isPlaying.value = false;
  console.error("视频播放错误:", error);
  emit("error", error);
};

const handleCanPlay = () => {
  emit("canplay");
  emit("loaded");
};

const handleTimeUpdate = (data) => {
  currentTime.value = data.currentTime;
  duration.value = data.duration;
};

// 处理视频播放结束
const handleVideoEnded = () => {
  console.log("视频播放结束");
  isPlaying.value = false;
  updatePageTitle(false, props.file?.name);
};

// 处理全屏事件
const handleFullscreen = () => {
  console.log("进入全屏模式");
  emit("fullscreen");
};

const handleFullscreenExit = () => {
  console.log("退出全屏模式");
  emit("fullscreenExit");
};

// 处理播放器准备就绪
const handlePlayerReady = (player) => {
  console.log("🎬 视频播放器准备就绪:", player);
};

// 初始化当前视频数据
const initializeCurrentVideo = async () => {
  if (!props.file) {
    console.log("❌ 无法初始化当前视频：文件信息为空");
    return;
  }

  console.log("🎬 开始初始化当前视频:", props.file.name);

  // 使用S3预签名URL或传入的视频URL
  if (props.videoUrl) {
    console.log("🎬 使用传入的视频URL:", props.videoUrl);
    currentVideoData.value = {
      name: props.file.name || "unknown",
      title: props.file.name || "unknown",
      url: props.videoUrl,
      poster: generateDefaultPoster(props.file.name),
      contentType: props.file.contentType,
      originalFile: props.file,
    };
    return;
  }

  // 降级方案：理论上不应该到达这里，因为videoUrl应该总是存在
  console.warn("⚠️ videoUrl为空，这表明上游有问题");
  currentVideoData.value = {
    name: props.file.name || "unknown",
    title: props.file.name || "unknown",
    url: null,
    poster: generateDefaultPoster(props.file.name),
    contentType: props.file.contentType,
    originalFile: props.file,
  };
};

// 生成默认封面
const generateDefaultPoster = (name) => {
  const firstChar = (name || "V")[0].toUpperCase();
  const canvas = document.createElement("canvas");
  canvas.width = 320;
  canvas.height = 180;
  const ctx = canvas.getContext("2d");

  // 背景色
  ctx.fillStyle = props.darkMode ? "#374151" : "#6b7280";
  ctx.fillRect(0, 0, 320, 180);

  // 播放按钮背景
  ctx.fillStyle = props.darkMode ? "#60a5fa" : "#3b82f6";
  ctx.beginPath();
  ctx.arc(160, 90, 30, 0, 2 * Math.PI);
  ctx.fill();

  // 播放按钮三角形
  ctx.fillStyle = "#ffffff";
  ctx.beginPath();
  ctx.moveTo(150, 75);
  ctx.lineTo(150, 105);
  ctx.lineTo(175, 90);
  ctx.closePath();
  ctx.fill();

  // 文件名
  ctx.fillStyle = "#ffffff";
  ctx.font = "bold 16px Arial";
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  ctx.fillText(firstChar, 160, 140);

  return canvas.toDataURL();
};

// 监听 videoUrl 变化，当准备好时初始化当前视频
watch(
  () => props.videoUrl,
  async (newVideoUrl) => {
    // 当videoUrl存在且文件信息存在时，初始化视频数据
    if (newVideoUrl && props.file) {
      console.log("🎬 检测到 videoUrl 变化，开始重新初始化当前视频:", newVideoUrl);
      await initializeCurrentVideo();
    }
  },
  { immediate: true } // 立即执行，确保首次加载时也会触发
);

// 快捷键处理
const handleKeydown = (event) => {
  // 如果用户正在输入框中输入，不处理快捷键
  if (event.target.tagName === "INPUT" || event.target.tagName === "TEXTAREA") {
    return;
  }

  const player = videoPlayerRef.value?.getInstance();
  if (!player) return;

  switch (event.code) {
    case "Space":
      event.preventDefault();
      player.toggle(); // 播放/暂停
      break;
    case "ArrowLeft":
      event.preventDefault();
      player.seek = Math.max(0, player.currentTime - 10); // 后退10秒
      break;
    case "ArrowRight":
      event.preventDefault();
      player.seek = Math.min(player.duration, player.currentTime + 10); // 前进10秒
      break;
    case "ArrowUp":
      event.preventDefault();
      player.volume = Math.min(1, player.volume + 0.1); // 音量+10%
      break;
    case "ArrowDown":
      event.preventDefault();
      player.volume = Math.max(0, player.volume - 0.1); // 音量-10%
      break;
    case "KeyF":
      event.preventDefault();
      player.fullscreen = !player.fullscreen; // 切换全屏
      break;
  }
};

// 生命周期钩子
onMounted(() => {
  // 保存原始页面标题
  originalTitle.value = document.title;

  // 添加键盘事件监听
  document.addEventListener("keydown", handleKeydown);

  // 不需要在这里初始化视频，watch 会处理
});

onBeforeUnmount(() => {
  // 恢复原始页面标题
  restoreOriginalTitle();

  // 移除键盘事件监听
  document.removeEventListener("keydown", handleKeydown);

  console.log("🧹 视频预览组件已卸载");
});
</script>

<style scoped>
@import "@/styles/pages/mount-explorer/video-preview.css";
</style>
