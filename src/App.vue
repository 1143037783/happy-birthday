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
      <div 
        class="carousel-container"
        @touchstart="touchStart"
        @touchmove="touchMove"
        @touchend="touchEnd"
      >
        <div class="carousel" :style="{ transform: `translateX(-${currentSlide * 100}%)` }">
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
              />
              <img 
                :src="photo.original" 
                :alt="`Original photo ${index + 1}`" 
                class="original-photo"
              />
            </div>
          </div>
        </div>
        <div class="carousel-controls">
          <button @click="prevSlide" class="control-btn">‹</button>
          <button @click="nextSlide" class="control-btn">›</button>
        </div>
        <div class="carousel-indicators">
          <span 
            v-for="(photo, index) in photos" 
            :key="index"
            class="indicator"
            :class="{ active: index === currentSlide }"
            @click="goToSlide(index)"
          ></span>
        </div>
      </div>
      
      <!-- 音乐信息和控制 -->
      <div class="player-info">
        <h3>宫崎骏音乐</h3>
        <p>《龙猫》主题曲</p>
        <div class="player-controls">
          <button 
            @click="togglePlay" 
            class="play-btn" 
            :style="{
              boxShadow: `0 0 ${20 + audioLevel * 30}px ${5 + audioLevel * 15}px rgba(255, 107, 107, ${0.3 + audioLevel * 0.5})`
            }"
          >{{ isPlaying ? '暂停' : '播放' }}</button>
          <audio ref="audio" @play="handlePlay" @pause="handlePause" @ended="handleEnded">
            <source src="https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3" type="audio/mpeg">
            您的浏览器不支持音频播放。
          </audio>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

// 照片数据 - 后续替换为实际的照片
const photos = ref([
  {
    cartoon: 'https://trae-api-cn.mchost.guru/api/ide/v1/text_to_image?prompt=Studio%20Ghibli%20style%20cartoon%20of%20a%20happy%20girl%20smiling&image_size=square',
    original: 'https://trae-api-cn.mchost.guru/api/ide/v1/text_to_image?prompt=photo%20of%20a%20happy%20girl%20smiling&image_size=square'
  },
  {
    cartoon: 'https://trae-api-cn.mchost.guru/api/ide/v1/text_to_image?prompt=Studio%20Ghibli%20style%20cartoon%20of%20a%20girl%20laughing&image_size=square',
    original: 'https://trae-api-cn.mchost.guru/api/ide/v1/text_to_image?prompt=photo%20of%20a%20girl%20laughing&image_size=square'
  },
  {
    cartoon: 'https://trae-api-cn.mchost.guru/api/ide/v1/text_to_image?prompt=Studio%20Ghibli%20style%20cartoon%20of%20a%20cheerful%20girl&image_size=square',
    original: 'https://trae-api-cn.mchost.guru/api/ide/v1/text_to_image?prompt=photo%20of%20a%20cheerful%20girl&image_size=square'
  }
]);

const audio = ref(null);
const isPlaying = ref(false);
const currentSlide = ref(0);
const touchStartX = ref(0);
const touchEndX = ref(0);
const audioLevel = ref(0);

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
  currentSlide.value = (currentSlide.value + 1) % photos.value.length;
};

const prevSlide = () => {
  currentSlide.value = (currentSlide.value - 1 + photos.value.length) % photos.value.length;
};

const goToSlide = (index) => {
  currentSlide.value = index;
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
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem 0;
}

/* 音乐和照片结合的容器 */
.music-photo-container {
  position: relative;
  width: 90%;
  max-width: 900px;
  margin: 3rem auto;
  padding: 2rem;
  background: linear-gradient(135deg, #1a1a1a 0%, #2d2d2d 100%);
  border-radius: 20px;
  box-shadow: 0 15px 35px rgba(0, 0, 0, 0.6);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2rem;
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

/* 轮播图样式 */
.carousel-container {
  position: relative;
  width: 100%;
  max-width: 600px;
  overflow: hidden;
  border-radius: 15px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
  margin-top: 30px;
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
  touch-action: pan-y;
}

.carousel {
  display: flex;
  transition: transform 0.5s ease-in-out;
}

.carousel-slide {
  flex: 0 0 100%;
  position: relative;
  cursor: pointer;
}

.photo-wrapper {
  position: relative;
  width: 100%;
  padding-top: 75%; /* 4:3 比例 */
}

.photo-wrapper img {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
  transition: opacity 0.5s ease;
}

.photo-wrapper .original-photo {
  opacity: 0;
}

/* 触摸设备上的悬停效果 */
@media (hover: hover) {
  .carousel-slide:hover .cartoon-photo {
    opacity: 0;
  }
  
  .carousel-slide:hover .original-photo {
    opacity: 1;
  }
}

/* 触摸设备上的点击效果 */
.carousel-slide:active .cartoon-photo {
  opacity: 0;
}

.carousel-slide:active .original-photo {
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
  position: absolute;
  bottom: 15px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 8px;
  z-index: 10;
}

.indicator {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.5);
  cursor: pointer;
  transition: all 0.3s ease;
  -webkit-tap-highlight-color: transparent;
}

.indicator.active {
  background: rgba(255, 107, 107, 1);
  transform: scale(1.2);
}

/* 音乐信息和控制 */
.player-info {
  width: 100%;
  max-width: 600px;
  color: #fff;
  text-align: center;
  margin-top: 1rem;
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
  padding: 0.8rem 2rem;
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