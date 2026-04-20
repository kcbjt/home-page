<template>
  <div :class="store.backgroundShow ? 'cover show' : 'cover'">
    <img
      v-show="store.imgLoadStatus"
      :src="bgUrl"
      class="bg"
      :style="bgStyle"
      alt="cover"
      @load="imgLoadComplete"
      @error="imgLoadError"
      @animationend="imgAnimationEnd"
    />
    <div :class="store.backgroundShow ? 'gray hidden' : 'gray'" />
    <Transition name="fade" mode="out-in">
      <a
        v-if="store.backgroundShow && store.coverType != '3'"
        class="down"
        href="javascript:void(0)"
        @click.prevent="downloadCurrentWallpaper"
      >
        下载壁纸
      </a>
    </Transition>
  </div>
</template>

<script setup>
import { mainStore } from "@/store";
import { Error } from "@icon-park/vue-next";
import { ElMessage } from "element-plus";
import { h } from "vue";

const store = mainStore();

// 可配置的虚化参数（也可以从 store 或 props 读取）
const props = defineProps({
  blurAmount: {
    type: Number,
    default: 20,        // 默认 20px 模糊
  },
  brightness: {
    type: Number,
    default: 0.3,
  },
  // 是否启用动态模糊过渡
  enableBlurTransition: {
    type: Boolean,
    default: true,
  },
});

const emit = defineEmits(["loadComplete"]);

const bgUrl = ref(null);
const imgTimeout = ref(null);
const loadErrorCount = ref(0); // 错误重试计数

// 动态生成 filter 样式
const bgStyle = computed(() => ({
  filter: `blur(${props.blurAmount}px) brightness(${props.brightness})`,
  transition: props.enableBlurTransition ? 'filter 0.3s ease' : 'none',
}));

// 本地壁纸列表（实际应通过 import 或配置获得）
const localBgList = computed(() => {
  // 假设本地有 10 张，可改成动态获取
  return Array.from({ length: 10 }, (_, i) => `/images/background${i + 1}.jpg`);
});

// 获取一个有效的本地随机壁纸
const getRandomLocalBg = () => {
  const validList = localBgList.value;
  if (!validList.length) return '/images/fallback.jpg';
  const randomIndex = Math.floor(Math.random() * validList.length);
  return validList[randomIndex];
};

// 更换壁纸链接（保留原始接口，增加本地 fallback 策略）
const changeBg = async (type) => {
  // 重置加载状态，防止旧图片残留
  store.setImgLoadStatus(false);
  clearTimeout(imgTimeout.value);

  let newUrl = null;
  try {
    if (type == 0) {
      newUrl = getRandomLocalBg();
    } else if (type == 1) {
      newUrl = "https://api.dujin.org/bing/1920.php";
    } else if (type == 2) {
      newUrl = "https://api.vvhan.com/api/wallpaper/views";
    } else if (type == 3) {
      newUrl = "https://api.vvhan.com/api/wallpaper/acg";
    } else {
      newUrl = getRandomLocalBg();
    }
    bgUrl.value = newUrl;
  } catch (err) {
    console.error("切换壁纸错误", err);
    bgUrl.value = getRandomLocalBg();
  }
};

// 图片加载完成
const imgLoadComplete = () => {
  // 清除之前的超时，避免重复设置
  if (imgTimeout.value) clearTimeout(imgTimeout.value);
  imgTimeout.value = setTimeout(
    () => {
      store.setImgLoadStatus(true);
    },
    Math.floor(Math.random() * (600 - 300 + 1)) + 300,
  );
};

// 图片动画完成（保证发射事件）
const imgAnimationEnd = () => {
  console.log("壁纸加载且动画完成");
  emit("loadComplete");
};

// 图片加载失败
const imgLoadError = () => {
  console.error("壁纸加载失败：", bgUrl.value);
  loadErrorCount.value++;

  // 如果当前是本地图片且已经重试过，则不再切换
  if (bgUrl.value?.startsWith('/images/') && loadErrorCount.value > 1) {
    ElMessage.error("壁纸加载失败，已使用纯色背景");
    bgUrl.value = ''; // 触发一个透明/无图状态，可在父级增加纯色背景
    return;
  }

  ElMessage({
    message: "壁纸加载失败，已切换回默认壁纸",
    icon: h(Error, { theme: "filled", fill: "#efefef" }),
  });
  // 降级：切换成本地随机壁纸
  bgUrl.value = getRandomLocalBg();
};

// 下载原图（通过 Blob 方式，解决 API 直接保存问题）
const downloadCurrentWallpaper = async () => {
  if (!bgUrl.value) return;
  try {
    const response = await fetch(bgUrl.value);
    if (!response.ok) throw new Error("下载失败");
    const blob = await response.blob();
    const link = document.createElement("a");
    const objectUrl = URL.createObjectURL(blob);
    link.href = objectUrl;
    link.download = `wallpaper_${Date.now()}.jpg`; // 可提取原图扩展名
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    URL.revokeObjectURL(objectUrl);
    ElMessage.success("壁纸下载开始");
  } catch (err) {
    console.error("下载失败", err);
    ElMessage.error("下载失败，可尝试右键图片另存为");
  }
};

// 监听壁纸切换
watch(
  () => store.coverType,
  (value) => {
    changeBg(value);
  },
);

onMounted(() => {
  changeBg(store.coverType);
});

onBeforeUnmount(() => {
  if (imgTimeout.value) clearTimeout(imgTimeout.value);
});
</script>

<style lang="scss" scoped>
.cover {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  transition: 0.25s;
  z-index: -1;

  &.show {
    z-index: 1;
  }

  .bg {
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
    backface-visibility: hidden;
    /* filter 移到行内 style 动态控制，但保留动画 */
    animation: fade-blur-in 0.8s cubic-bezier(0.25, 0.46, 0.45, 0.94) forwards;
    animation-delay: 0.45s;
  }

  .gray {
    opacity: 1;
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    background-image: radial-gradient(rgba(0, 0, 0, 0) 0, rgba(0, 0, 0, 0.5) 100%),
      radial-gradient(rgba(0, 0, 0, 0) 33%, rgba(0, 0, 0, 0.3) 166%);
    transition: 1.5s;

    &.hidden {
      opacity: 0;
      transition: 1.5s;
    }
  }

  .down {
    font-size: 16px;
    color: white;
    position: absolute;
    bottom: 30px;
    left: 0;
    right: 0;
    margin: 0 auto;
    display: block;
    padding: 20px 26px;
    border-radius: 8px;
    background-color: #00000030;
    width: 120px;
    height: 30px;
    display: flex;
    justify-content: center;
    align-items: center;
    cursor: pointer;
    transition: 0.2s;

    &:hover {
      transform: scale(1.05);
      background-color: #00000060;
    }
    &:active {
      transform: scale(1);
    }
  }
}

@keyframes fade-blur-in {
  0% {
    opacity: 0;
    transform: scale(1.05);
  }
  100% {
    opacity: 1;
    transform: scale(1);
  }
}
</style>