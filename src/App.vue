<script setup lang="ts">
import { ref, watch, onMounted } from 'vue';

const vanishingPointX = ref(50);
const vanishingPointY = ref(50);
const radius = ref(5);
const speedLineCount = ref(100);

const canvasWidth = 800;
const canvasHeight = 600;

const canvas = ref<HTMLCanvasElement | null>(null);
const speedLineCanvas = ref<HTMLCanvasElement | null>(null);

onMounted(() => {
  drawSpeedLines();
});

function drawSpeedLines() {
  const ctx = speedLineCanvas.value?.getContext('2d');
  if (ctx) {
    ctx.clearRect(0, 0, canvasWidth, canvasHeight);
    
    const vpX = vanishingPointX.value * canvasWidth / 100;
    const vpY = vanishingPointY.value * canvasHeight / 100;
    const r = radius.value;
    
    ctx.strokeStyle = '#000';
    ctx.lineWidth = 1;
    
    for (let i = 0; i < speedLineCount.value; i++) {
      // pick a random point on the edge of the canvas
      let startX: number, startY: number;
      const edge = Math.floor(Math.random() * 4); // 0: top, 1: right, 2: bottom, 3: left
      
      if (edge === 0) {
        // top edge
        startX = Math.random() * canvasWidth;
        startY = 0;
      } else if (edge === 1) {
        // right edge
        startX = canvasWidth;
        startY = Math.random() * canvasHeight;
      } else if (edge === 2) {
        // bottom edge
        startX = Math.random() * canvasWidth;
        startY = canvasHeight;
      } else {
        // left edge
        startX = 0;
        startY = Math.random() * canvasHeight;
      }
      
      // draw towards the vanishing point, stopping at the radius
      const dx = vpX - startX;
      const dy = vpY - startY;
      const distance = Math.sqrt(dx * dx + dy * dy);
      
      if (distance > 0) {
        // Normalize direction vector
        const dirX = dx / distance;
        const dirY = dy / distance;
        
        // Calculate end point at the edge of the radius circle
        const endX = vpX - dirX * r;
        const endY = vpY - dirY * r;
        
        ctx.beginPath();
        ctx.moveTo(startX, startY);
        ctx.lineTo(endX, endY);
        ctx.stroke();
      }
    }
  }
}


</script>

<template>
  <h1>You did it!</h1>
  <p>
    Visit <a href="https://vuejs.org/" target="_blank" rel="noopener">vuejs.org</a> to read the
    documentation
  </p>
  <div class="vanishing-point">
    <input type="range" min="0" max="100" v-model="vanishingPointX" /><span>{{ vanishingPointX }}</span>
    <input type="range" min="0" max="100" v-model="vanishingPointY" /><span>{{ vanishingPointY }}</span>
    <input type="range" min="0" max="1000" v-model="radius" /><span>{{ radius }}</span>
    <input type="range" min="10" max="1000" v-model="speedLineCount" /><span>{{ speedLineCount }}</span>
  </div>
  <div>
    <button @click="drawSpeedLines">Draw Speed Lines</button>
  </div>
  <svg :width="canvasWidth" :height="canvasHeight" :viewBox="`0 0 ${canvasWidth} ${canvasHeight}`">
    <circle :cx="vanishingPointX * canvasWidth / 100" :cy="vanishingPointY * canvasHeight / 100" :r="radius" />
  </svg>
  <canvas ref="speedLineCanvas" id="speed-lines" :width="canvasWidth" :height="canvasHeight"></canvas>
</template>

<style scoped>
svg {
  width: 800px;
  height: 600px;
  border: 1px solid black;
}

#speed-lines {
  border: 1px solid red;
}
</style>
