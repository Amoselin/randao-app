<template>
  <div class="flex flex-col items-center justify-center min-h-screen bg-stone-50 p-6 font-serif">
    <video ref="videoRef" autoplay playsinline class="hidden"></video>

    <div class="max-w-md w-full text-center space-y-12">
      <div class="relative flex justify-center">
        <h1 class="text-8xl text-stone-800 transition-all duration-700"
            :style="{ opacity: isPlaying ? 0.3 + (brightness / 255) : 0.2, transform: `scale(${1 + brightness/1000})` }">
          À
        </h1>
      </div>

      <div class="space-y-4">
        <h2 class="text-xl tracking-[0.4em] text-stone-600">仁．REN</h2>
        <p class="text-xs text-stone-400 tracking-widest uppercase">
          {{ isPlaying ? '隨機頻率共振中' : '準備進入冥想時程' }}
        </p>
      </div>

      <button @click="toggleMeditation" 
              class="group relative px-12 py-4 border border-stone-800 rounded-full overflow-hidden transition-all duration-500 hover:bg-stone-800">
        <span class="relative z-10 text-sm tracking-[0.3em] group-hover:text-white transition-colors">
          {{ isPlaying ? '結束引導' : '啟動 15 分鐘時程' }}
        </span>
      </button>

      <div v-if="isPlaying" class="pt-8">
        <div class="w-full h-[1px] bg-stone-200 relative">
          <div class="absolute h-full bg-stone-800 transition-all duration-300" 
               :style="{ width: `${(brightness / 255) * 100}%` }"></div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onUnmounted } from 'vue';
import * as Tone from 'tone';

const videoRef = ref(null);
const brightness = ref(0);
const isPlaying = ref(false);

// 聲學引擎變數
let player = null; // 固定音頻 (仁)
let synth = null;  // 隨機生成合成器
let animationFrame = null;

// 1. 初始化音訊系統
const initAudio = async () => {
  await Tone.start();

  // A. 加載固定音頻 (請確保檔案放在 public/audio/ren.mp3)
  player = new Tone.Player({
    url: "/audio/ren.mp3",
    loop: false,
    fadeOut: 3
  }).toDestination();

  // B. 建立隨機合成器 (FM 合成器最能表現「氣」的流動)
  synth = new Tone.FMSynth({
    harmonicity: 3.01,
    modulationIndex: 5,
    oscillator: { type: 'sine' },
    envelope: { attack: 0.5, release: 2 }
  }).toDestination();

  // 加入深遠的混響
  const reverb = new Tone.Reverb(12).toDestination();
  synth.connect(reverb);
};

// 2. 啟動感應與 RANDAO 隨機邏輯
const analyzeLight = () => {
  const video = videoRef.value;
  if (video && video.readyState === video.HAVE_ENOUGH_DATA) {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');
    canvas.width = 40; canvas.height = 30;
    ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
    const data = ctx.getImageData(0, 0, canvas.width, canvas.height).data;
    
    let total = 0;
    for (let i = 0; i < data.length; i += 4) {
      total += (data[i] + data[i + 1] + data[i + 2]) / 3;
    }
    brightness.value = total / (data.length / 4);

    // RANDAO 邏輯：光值改變合成器的「調變指數」與「隨機音高偏移」
    if (synth && isPlaying.value) {
      const modLevel = Tone.util.map(brightness.value, 0, 255, 2, 25);
      synth.modulationIndex.rampTo(modLevel, 0.2);
      
      // 隨機觸發高頻「靈光」音符 (模擬 RANDAO 隨機性)
      if (Math.random() > 0.98) { // 隨機機率
        const notes = ["C5", "E5", "G5", "A5"]; // 仁的五音階
        const randomNote = notes[Math.floor(Math.random() * notes.length)];
        synth.triggerAttackRelease(randomNote, "8n");
      }
    }
  }
  animationFrame = requestAnimationFrame(analyzeLight);
};

const toggleMeditation = async () => {
  if (isPlaying.value) {
    stopAll();
    return;
  }

  await initAudio();
  
  // 啟動相機
  const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'user' } });
  videoRef.value.srcObject = stream;
  
  // 開始播放與偵測
  player.start();
  isPlaying.value = true;
  analyzeLight();
};

const stopAll = () => {
  isPlaying.value = false;
  player?.stop();
  synth?.triggerRelease();
  cancelAnimationFrame(animationFrame);
  if (videoRef.value?.srcObject) {
    videoRef.value.srcObject.getTracks().forEach(t => t.stop());
  }
};

onUnmounted(stopAll);
</script>