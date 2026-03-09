<template>
  <div class="app">
    <h1>生日快乐！</h1>
    
    <!-- 音乐和照片结合的容器 -->
    <div class="music-photo-container">
      <!-- 黑胶唱片播放器 -->
      <div class="vinyl-player">
        <div class="vinyl-container">
          <div class="vinyl" :class="{ 'playing': isPlaying }">
            <div class="vinyl-label">
              <img src="https://trae-api-cn.mchost.guru/api/ide/v1/text_to_image?prompt=Studio%20Ghibli%20logo%20on%20vinyl%20record&image_size=square" alt="Ghibli" />
            </div>
          </div>
          <div class="vinyl-arm" :class="{ 'playing': isPlaying }">
            <div class="needle"></div>
          </div>
        </div>
      </div>
      
      <!-- 照片轮播图 -->
      <div class="carousel-wrapper">
        <div 
          class="carousel-container"
          @touchstart="touchStart"
          @touchmove="touchMove"
          @touchend="touchEnd"
        >
          <div class="carousel" :style="{ transform: `translateX(calc(-${currentSlide * 100}%))`, transition: isTransitioning ? 'transform 0.5s ease-in-out' : 'none' }">
          <!-- 额外的最后一张图片（用于循环） -->
          <div 
            v-if="photos.length > 0" 
            class="carousel-slide"
            :style="{
              boxShadow: `0 10px ${30 + audioLevel * 40}px ${10 + audioLevel * 20}px rgba(0, 0, 0, ${0.2 + audioLevel * 0.3})`
            }"
          >
              <div class="photo-wrapper">
                <img 
                  :src="photos[photos.length - 1].cartoon" 
                  :alt="`Cartoon photo ${photos.length}`"
                  class="cartoon-photo"
                />
                <img 
                  :src="photos[photos.length - 1].original" 
                  :alt="`Original photo ${photos.length}`"
                  class="original-photo"
                />
              </div>
            </div>
          <!-- 实际图片 -->
          <div 
            class="carousel-slide" 
            v-for="(photo, index) in photos" 
            :key="index"
            :style="{
              boxShadow: `0 10px ${30 + audioLevel * 40}px ${10 + audioLevel * 20}px rgba(0, 0, 0, ${0.2 + audioLevel * 0.3})`
            }"
          >
              <div class="photo-wrapper">
                <img 
                  :src="photo.cartoon" 
                  :alt="`Cartoon photo ${index + 1}`" 
                  class="cartoon-photo"
                  :class="{ 'hidden': photo.showOriginal }"
                />
                <template v-if="photo.isVideo">
                  <!-- 视频 -->
                  <video 
                    :ref="el => videoRefs[index] = el" 
                    :src="photo.video" 
                    :alt="`Video ${index + 1}`" 
                    class="original-video"
                    :class="{ 'visible': photo.showOriginal && !photo.showOriginImage }"
                    autoplay
                    muted
                    playsinline
                    preload="auto"
                    @ended="handleVideoEnd(index)"
                  />
                  <!-- 原图 -->
                  <img 
                    :src="photo.original" 
                    :alt="`Original photo ${index + 1}`" 
                    class="original-photo"
                    :class="{ 'visible': photo.showOriginal && photo.showOriginImage }"
                  />
                </template>
                <template v-else>
                  <img 
                    :src="photo.original" 
                    :alt="`Original photo ${index + 1}`" 
                    class="original-photo"
                    :class="{ 'visible': photo.showOriginal }"
                  />
                </template>
              </div>
            </div>
          <!-- 额外的第一张图片（用于循环） -->
          <div 
            v-if="photos.length > 0" 
            class="carousel-slide"
            :style="{
              boxShadow: `0 10px ${30 + audioLevel * 40}px ${10 + audioLevel * 20}px rgba(0, 0, 0, ${0.2 + audioLevel * 0.3})`
            }"
          >
              <div class="photo-wrapper">
                <img 
                  :src="photos[0].cartoon" 
                  :alt="`Cartoon photo 1`"
                  class="cartoon-photo"
                />
                <img 
                  :src="photos[0].original" 
                  :alt="`Original photo 1`"
                  class="original-photo"
                />
              </div>
            </div>
          </div>
        </div>
        <div class="carousel-indicators">
          <div class="indicator-text">
            {{ ((currentSlide - 1) % photos.length + photos.length) % photos.length + 1 }}/{{ photos.length }}
          </div>
          <div class="photo-date">
            {{ ((currentSlide - 1) % photos.length + photos.length) % photos.length === 8 ? '天空之城视频' : formatDate(photoDate[((currentSlide - 1) % photos.length + photos.length) % photos.length]) }}
          </div>
        </div>
      </div>
      
      <!-- 音乐信息和控制 -->
      <div class="player-info">
        <h3>宫崎骏音乐</h3>
        <p>{{ currentMusic.title }}</p>
        <div class="player-controls">
          <button 
            @click="togglePlay" 
            class="play-btn" 
            :style="{
              boxShadow: `0 0 ${20 + audioLevel * 30}px ${5 + audioLevel * 15}px rgba(255, 107, 107, ${0.3 + audioLevel * 0.5})`
            }"
          >{{ isPlaying ? '暂停' : '播放' }}</button>
          <audio ref="audio" @play="handlePlay" @pause="handlePause" @ended="handleEnded">
            <source :src="currentMusic.src" type="audio/mpeg">
            您的浏览器不支持音频播放。
           </audio>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

// 照片数据 - 从文件夹中自动读取图片
const photos = ref([]);
const photoDate = [20230314,20230323,20230820,20231002,20240203,20240218,20250824,20251012,20260223];

// 格式化日期函数
const formatDate = (dateStr) => {
  const year = dateStr.toString().substring(0, 4);
  const month = dateStr.toString().substring(4, 6);
  const day = dateStr.toString().substring(6, 8);
  return `${year}年${month}月${day}日`;
};

// 初始化照片数据
const initPhotos = () => {
  // 假设图片按照年月日命名，例如：20210101.png
  const photoList = [];
  
  for (let i = 0; i < photoDate.length; i++) {
    const date = photoDate[i];
    // 替换第9张为视频
    if (i === 8) {
      photoList.push({
        cartoon: new URL(`./assets/photos/ghibli/${date}.png`, import.meta.url).href,
        original: new URL(`./assets/photos/original/${date}.png`, import.meta.url).href,
        video: new URL(`./assets/videos/天空之城.mp4`, import.meta.url).href,
        showOriginal: false,
        showOriginImage: false,
        isVideo: true
      });
    } else {
      photoList.push({
        cartoon: new URL(`./assets/photos/ghibli/${date}.png`, import.meta.url).href,
        original: new URL(`./assets/photos/original/${date}.png`, import.meta.url).href,
        showOriginal: false
      });
    }
  }
  
  photos.value = photoList;
};

// 初始化照片数据
initPhotos();

// 视频计时器
const videoTimers = ref({});

// 处理视频结束事件
const handleVideoEnd = (index) => {
  // 清除之前的计时器
  if (videoTimers.value[index]) {
    clearTimeout(videoTimers.value[index]);
  }
  
  // 视频播放完成后，显示原图3秒
  const currentPhoto = photos.value[index];
  if (currentPhoto) {
    // 显示原图
    currentPhoto.showOriginImage = true;
    
    // 3秒后重新播放视频
    videoTimers.value[index] = setTimeout(() => {
      // 隐藏原图，显示视频
      currentPhoto.showOriginImage = false;
      const video = videoRefs.value[index];
      if (video) {
        video.currentTime = 0;
        video.play().catch(error => {
          console.log('视频播放被阻止:', error);
        });
      }
    }, 3000);
  }
};

// 自动切换照片显示状态
const startPhotoAnimation = () => {
  // 清除之前的计时器
  if (animationTimer) {
    clearTimeout(animationTimer);
  }
  
  // 只对当前显示的照片进行变换
  // 调整索引，因为currentSlide从1开始，而photos数组从0开始
  let photoIndex = (currentSlide.value - 1) % photos.value.length;
  // 处理边界情况
  if (photoIndex < 0) {
    photoIndex = photos.value.length - 1;
  }
  const currentPhoto = photos.value[photoIndex];
  if (currentPhoto) {
    // 首先显示卡通照片3秒
    currentPhoto.showOriginal = false;
    // 对于视频，确保显示视频而不是原图
    if (currentPhoto.isVideo) {
      currentPhoto.showOriginImage = false;
    }
    // 3秒后切换到原照片
    setTimeout(() => {
      currentPhoto.showOriginal = true;
      // 非视频内容再3秒后恢复显示卡通照片
      if (!currentPhoto.isVideo) {
        setTimeout(() => {
          currentPhoto.showOriginal = false;
        }, 3000);
      }
      // 视频内容由视频结束事件处理
    }, 3000);
  }
  
  // 循环播放，每6秒检查一次当前照片
  // 只有当当前不是视频时才启动循环
  if (!currentPhoto || !currentPhoto.isVideo) {
    animationTimer = setTimeout(startPhotoAnimation, 6000);
  }
};



// 启动照片动画
onMounted(() => {
  // 随机选择音乐
  selectRandomMusic();
  
  // 页面加载后设置音频事件监听器
  if (audio.value) {
    audio.value.addEventListener('play', handlePlay);
    audio.value.addEventListener('pause', handlePause);
    audio.value.addEventListener('ended', handleEnded);
  }
  
  // 启动照片动画
  startPhotoAnimation();
});

const audio = ref(null);
const videoRefs = ref({});
const isPlaying = ref(false);
const currentSlide = ref(1);
const touchStartX = ref(0);
const touchEndX = ref(0);
const audioLevel = ref(0);
const isTransitioning = ref(true);
let animationTimer = null;

// 音乐列表
const musicList = [
  {
    title: '天空之城',
    src: new URL('./assets/music/天空之城.mp3', import.meta.url).href
  },
  {
    title: 'Always With Me',
    src: new URL('./assets/music/Always With Me.mp3', import.meta.url).href
  },
  {
    title: 'The Wind Forest',
    src: new URL('./assets/music/The Wind Forest.mp3', import.meta.url).href
  }
];

// 当前播放的音乐
const currentMusic = ref({});

// 随机选择音乐
const selectRandomMusic = () => {
  const randomIndex = Math.floor(Math.random() * musicList.length);
  currentMusic.value = musicList[randomIndex];
};

// 音频分析相关
let audioContext = null;
let analyser = null;
let dataArray = null;
let animationId = null;

const togglePlay = () => {
  if (audio.value) {
    if (isPlaying.value) {
      audio.value.pause();
    } else {
      audio.value.play().catch(error => {
        console.log('播放被阻止:', error);
      });
    }
  }
};

const nextSlide = () => {
  isTransitioning.value = true;
  
  // 清除所有视频计时器
  Object.values(videoTimers.value).forEach(timer => {
    clearTimeout(timer);
  });
  videoTimers.value = {};
  
  currentSlide.value++;
  
  // 处理环形逻辑
  if (currentSlide.value > photos.value.length) {
    // 当滑动到最后一张副本时，先完成动画，然后瞬间切换到第一张实际图片
    setTimeout(() => {
      isTransitioning.value = false;
      currentSlide.value = 1;
      // 等待DOM更新后再恢复过渡效果
      setTimeout(() => {
        isTransitioning.value = true;
      }, 50);
    }, 500);
  }
  
  // 重置计时器，开始新图片的动画
  startPhotoAnimation();
};

const prevSlide = () => {
  isTransitioning.value = true;
  
  // 清除所有视频计时器
  Object.values(videoTimers.value).forEach(timer => {
    clearTimeout(timer);
  });
  videoTimers.value = {};
  
  // 处理环形逻辑
  if (currentSlide.value === 1) {
    // 当滑动到第一张实际图片时，先瞬间切换到最后一张副本，然后开始动画
    isTransitioning.value = false;
    currentSlide.value = photos.value.length + 1;
    // 等待DOM更新后再开始动画
    setTimeout(() => {
      isTransitioning.value = true;
      currentSlide.value--;
    }, 50);
  } else {
    currentSlide.value--;
  }
  
  // 重置计时器，开始新图片的动画
  startPhotoAnimation();
};

const goToSlide = (index) => {
  // 清除所有视频计时器
  Object.values(videoTimers.value).forEach(timer => {
    clearTimeout(timer);
  });
  videoTimers.value = {};
  
  // 调整索引，因为currentSlide从1开始
  currentSlide.value = index + 1;
  // 重置计时器，开始新图片的动画
  startPhotoAnimation();
};

// 触摸事件处理
const touchStart = (e) => {
  touchStartX.value = e.changedTouches[0].screenX;
};

const touchMove = (e) => {
  touchEndX.value = e.changedTouches[0].screenX;
};

const touchEnd = () => {
  const minSwipeDistance = 50;
  if (touchEndX.value < touchStartX.value - minSwipeDistance) {
    // 向左滑动
    nextSlide();
  } else if (touchEndX.value > touchStartX.value + minSwipeDistance) {
    // 向右滑动
    prevSlide();
  }
};

// 音频分析初始化
const initAudioAnalysis = () => {
  if (!audio.value) return;
  
  try {
    audioContext = new (window.AudioContext || window.webkitAudioContext)();
    const source = audioContext.createMediaElementSource(audio.value);
    analyser = audioContext.createAnalyser();
    analyser.fftSize = 256;
    const bufferLength = analyser.frequencyBinCount;
    dataArray = new Uint8Array(bufferLength);
    
    source.connect(analyser);
    analyser.connect(audioContext.destination);
    
    // 开始分析
    updateAudioLevel();
  } catch (error) {
    console.log('音频分析初始化失败:', error);
  }
};

// 更新音频水平
const updateAudioLevel = () => {
  if (!analyser) return;
  
  analyser.getByteFrequencyData(dataArray);
  
  // 计算平均音量
  let sum = 0;
  for (let i = 0; i < dataArray.length; i++) {
    sum += dataArray[i];
  }
  const average = sum / dataArray.length;
  
  // 归一化到0-1范围
  audioLevel.value = average / 255;
  
  animationId = requestAnimationFrame(updateAudioLevel);
};

// 监听音频播放状态
const handlePlay = () => {
  isPlaying.value = true;
  if (!audioContext) {
    initAudioAnalysis();
  } else if (audioContext.state === 'suspended') {
    audioContext.resume();
  }
};

const handlePause = () => {
  isPlaying.value = false;
};

const handleEnded = () => {
  isPlaying.value = false;
};

onMounted(() => {
  // 页面加载后自动播放音乐（如果浏览器允许）
  if (audio.value) {
    audio.value.addEventListener('play', handlePlay);
    audio.value.addEventListener('pause', handlePause);
    audio.value.addEventListener('ended', handleEnded);
    
    audio.value.play().catch(error => {
      console.log('自动播放被阻止:', error);
    });
  }
});

onUnmounted(() => {
  if (audio.value) {
    audio.value.removeEventListener('play', handlePlay);
    audio.value.removeEventListener('pause', handlePause);
    audio.value.removeEventListener('ended', handleEnded);
  }
  
  if (animationId) {
    cancelAnimationFrame(animationId);
  }
  
  if (audioContext) {
    audioContext.close();
  }
});
</script>

<style scoped>
.app {
  width: 100%;
  max-width: 1440px;
  margin: 0 auto;
  overflow-x: auto;
  text-align: center;
}

.app h1 {
  margin-bottom: 2rem;
  font-size: 3rem;
  color: #ff6b6b;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.3);
}

/* 音乐和照片结合的容器 */
.music-photo-container {
  position: relative;
  width: 100%;
  max-width: 1440px;
  margin: 3rem auto;
  background: linear-gradient(135deg, #1a1a1a 0%, #2d2d2d 100%);
  border-radius: 20px;
  box-shadow: 0 15px 35px rgba(0, 0, 0, 0.6);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
  box-sizing: border-box;
}

/* 黑胶唱片播放器样式 */
.vinyl-player {
  position: absolute;
  top: -50px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 10;
}

.vinyl-container {
  position: relative;
  width: 100px;
  height: 100px;
}

.vinyl {
  width: 100%;
  height: 100%;
  border-radius: 50%;
  background: radial-gradient(circle, #333 0%, #000 70%);
  position: relative;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.8);
  transition: transform 0.3s ease;
}

.vinyl.playing {
  animation: rotate 10s linear infinite;
}

.vinyl-label {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 40px;
  height: 40px;
  border-radius: 50%;
  overflow: hidden;
  border: 2px solid #333;
}

.vinyl-label img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.vinyl-arm {
  position: absolute;
  top: -10px;
  right: -15px;
  width: 60px;
  height: 5px;
  background: #555;
  transform-origin: left center;
  transform: rotate(-30deg);
  transition: transform 0.5s ease;
}

.vinyl-arm.playing {
  transform: rotate(0deg);
}

.needle {
  position: absolute;
  right: 0;
  top: 50%;
  transform: translateY(-50%);
  width: 10px;
  height: 2px;
  background: #888;
  border-radius: 1px;
}

.carousel-wrapper {
  width: 100%;
  max-width: 1400px;
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 10px auto 0;
}

/* 轮播图样式 */
.carousel-container {
  position: relative;
  width: 100%;
  max-width: 1400px;
  overflow: hidden;
  border-radius: 15px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
  touch-action: pan-y;
  padding: 10px 0;
}

.carousel {
  display: flex;
  transition: transform 0.5s ease-in-out;
  align-items: center;
}

.carousel-slide {
  flex: 0 0 100%;
  position: relative;
  cursor: pointer;
  margin: 0;
  border-radius: 10px;
  overflow: hidden;
  transition: all 0.5s ease-in-out;
  z-index: 1;
}



.photo-wrapper {
  position: relative;
  width: 100%;
  padding-top: 133.33%; /* 3:4 比例 */
}

.photo-wrapper img,
.photo-wrapper video {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
  transition: opacity 0.5s ease;
}

.photo-wrapper .cartoon-photo {
  opacity: 1;
  transition: opacity 0.5s ease;
}

.photo-wrapper .original-photo,
.photo-wrapper .original-video {
  opacity: 0;
  transition: opacity 0.5s ease;
}

/* 自动切换效果 */
.photo-wrapper .cartoon-photo.hidden {
  opacity: 0;
}

.photo-wrapper .original-photo.visible,
.photo-wrapper .original-video.visible {
  opacity: 1;
}

/* 轮播控制按钮 */
.carousel-controls {
  position: absolute;
  top: 50%;
  left: 0;
  right: 0;
  transform: translateY(-50%);
  display: flex;
  justify-content: space-between;
  padding: 0 10px;
  z-index: 10;
}

.control-btn {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: rgba(0, 0, 0, 0.5);
  color: white;
  border: none;
  font-size: 1.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.3s ease;
  -webkit-tap-highlight-color: transparent;
}

.control-btn:hover {
  background: rgba(0, 0, 0, 0.8);
  transform: scale(1.1);
}

.control-btn:active {
  background: rgba(0, 0, 0, 0.9);
  transform: scale(0.95);
}

/* 轮播指示器 */
.carousel-indicators {
  position: relative;
  margin-top: 15px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 5px;
  z-index: 10;
  background: rgba(0, 0, 0, 0.6);
  padding: 10px 20px;
  border-radius: 20px;
  width: fit-content;
  margin-left: auto;
  margin-right: auto;
}

.indicator-text {
  color: white;
  font-size: 14px;
  font-weight: 500;
}

.photo-date {
  color: #ff6b6b;
  font-size: 12px;
  font-weight: 400;
}

/* 音乐信息和控制 */
.player-info {
  width: 100%;
  max-width: 600px;
  color: #fff;
  text-align: center;
  margin-top: 0.5rem;
}

.player-info h3 {
  margin: 0 0 0.5rem 0;
  font-size: 1.5rem;
  color: #ff6b6b;
}

.player-info p {
  margin: 0 0 1.5rem 0;
  color: #ccc;
  font-size: 1.1rem;
}

.player-controls {
  display: flex;
  align-items: center;
  gap: 1rem;
  justify-content: center;
  flex-wrap: wrap;
}

.play-btn {
  padding: 0.8rem;
  background: #ff6b6b;
  color: white;
  border: none;
  border-radius: 25px;
  font-size: 1rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
  -webkit-tap-highlight-color: transparent;
}

.play-btn:hover {
  background: #ff5252;
  transform: scale(1.05);
}

.play-btn:active {
  background: #ff3838;
  transform: scale(0.95);
}

.player-controls audio {
  flex: 1;
  height: 40px;
  max-width: 400px;
  min-width: 200px;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .app {
    padding: 1rem 0;
  }
  
  .music-photo-container {
    width: 95%;
    padding: 1.5rem;
    gap: 1.5rem;
  }
  
  .vinyl-container {
    width: 80px;
    height: 80px;
  }
  
  .vinyl-label {
    width: 32px;
    height: 32px;
  }
  
  .vinyl-arm {
    top: -8px;
    right: -12px;
    width: 48px;
    height: 4px;
  }
  
  .needle {
    width: 8px;
    height: 1.5px;
  }
  
  .carousel-container {
    margin-top: 20px;
  }
  
  .player-info h3 {
    font-size: 1.3rem;
  }
  
  .player-info p {
    font-size: 1rem;
    margin-bottom: 1rem;
  }
  
  .play-btn {
    padding: 0.7rem 1.5rem;
    font-size: 0.9rem;
  }
  
  .player-controls {
    flex-direction: column;
    align-items: stretch;
    gap: 1rem;
  }
  
  .player-controls audio {
    width: 100%;
    max-width: 100%;
  }
}

@media (max-width: 480px) {
  .music-photo-container {
    padding: 1rem;
  }
  
  .vinyl-container {
    width: 60px;
    height: 60px;
  }
  
  .vinyl-label {
    width: 24px;
    height: 24px;
  }
  
  .vinyl-arm {
    top: -6px;
    right: -9px;
    width: 36px;
    height: 3px;
  }
  
  .needle {
    width: 6px;
    height: 1px;
  }
  
  .control-btn {
    width: 35px;
    height: 35px;
    font-size: 1.2rem;
  }
  
  .indicator {
    width: 6px;
    height: 6px;
    gap: 6px;
  }
  
  .player-info h3 {
    font-size: 1.1rem;
  }
  
  .player-info p {
    font-size: 0.9rem;
  }
  
  .play-btn {
    padding: 0.6rem 1.2rem;
    font-size: 0.8rem;
  }
}

/* 动画 */
@keyframes rotate {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

/* 响应式设计 */
@media (max-width: 768px) {
  .vinyl-player {
    flex-direction: column;
    text-align: center;
    gap: 2rem;
  }
  
  .player-info {
    text-align: center;
  }
  
  .gallery {
    grid-template-columns: 1fr;
  }
}
</style>