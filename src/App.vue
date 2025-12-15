<script setup lang="ts">
import { ref, watch, onMounted } from 'vue';

const vanishingPointX = ref(50);
const vanishingPointY = ref(50);
const radius = ref(150);
const speedLineCount = ref(100);
const minWidth = ref(1);
const maxWidth = ref(5);
const lengthLeniency = ref(50); // number of pixels that can be above or below the ideal length

const canvasWidth = 800;
const canvasHeight = 600;

const speedLineCanvas = ref<HTMLCanvasElement | null>(null);
const svgElement = ref<SVGSVGElement | null>(null);
const isDragging = ref(false);
const isDraggingLeniency = ref(false);
const isDraggingRadius = ref(false);

onMounted(() => {
  drawSpeedLines();
});

watch([vanishingPointX, vanishingPointY, radius, speedLineCount, minWidth, maxWidth, lengthLeniency], () => {
  drawSpeedLines();
});

function getSVGPoint(event: MouseEvent | TouchEvent): { x: number; y: number } | null {
  if (!svgElement.value) return null;

  const svg = svgElement.value;
  const rect = svg.getBoundingClientRect();

  let clientX: number, clientY: number;

  if (event instanceof MouseEvent) {
    clientX = event.clientX;
    clientY = event.clientY;
  } else {
    const touch = event.touches[0] || event.changedTouches[0];
    if (!touch) return null;
    clientX = touch.clientX;
    clientY = touch.clientY;
  }

  // Convert screen coordinates to SVG coordinates
  // Scale based on the ratio of SVG viewBox to actual rendered size
  const scaleX = canvasWidth / rect.width;
  const scaleY = canvasHeight / rect.height;

  const x = (clientX - rect.left) * scaleX;
  const y = (clientY - rect.top) * scaleY;

  return { x, y };
}

function startDrag(event: MouseEvent | TouchEvent) {
  if (isDraggingLeniency.value || isDraggingRadius.value) return;
  event.preventDefault();
  isDragging.value = true;
}

function drag(event: MouseEvent | TouchEvent) {
  if (!isDragging.value || isDraggingLeniency.value || isDraggingRadius.value) return;
  event.preventDefault();

  const point = getSVGPoint(event);
  if (!point) return;

  // Convert SVG coordinates to percentage values
  const xPercent = (point.x / canvasWidth) * 100;
  const yPercent = (point.y / canvasHeight) * 100;

  // Clamp values to valid range (0-100)
  vanishingPointX.value = Math.max(0, Math.min(100, xPercent));
  vanishingPointY.value = Math.max(0, Math.min(100, yPercent));
}

function stopDrag(event: MouseEvent | TouchEvent) {
  if (isDraggingLeniency.value || isDraggingRadius.value) return;
  event.preventDefault();
  isDragging.value = false;
}

function startDragLeniency(event: MouseEvent | TouchEvent) {
  event.preventDefault();
  event.stopPropagation();
  isDraggingLeniency.value = true;
}

function startDragRadius(event: MouseEvent | TouchEvent) {
  event.preventDefault();
  event.stopPropagation();
  isDraggingRadius.value = true;
}

function dragLeniency(event: MouseEvent | TouchEvent) {
  if (!isDraggingLeniency.value) return;
  event.preventDefault();
  event.stopPropagation();

  const point = getSVGPoint(event);
  if (!point) return;

  const vpX = vanishingPointX.value * canvasWidth / 100;
  const vpY = vanishingPointY.value * canvasHeight / 100;

  // Calculate distance from vanishing point to mouse position
  const dx = point.x - vpX;
  const dy = point.y - vpY;
  const distance = Math.sqrt(dx * dx + dy * dy);

  // Update lengthLeniency based on distance from vanishing point
  // lengthLeniency = distance - radius
  const newLeniency = distance - radius.value;

  // Clamp to valid range (1-100 based on input range)
  lengthLeniency.value = Math.max(1, Math.min(100, newLeniency));
}

function dragRadius(event: MouseEvent | TouchEvent) {
  if (!isDraggingRadius.value) return;
  event.preventDefault();
  event.stopPropagation();

  const point = getSVGPoint(event);
  if (!point) return;

  const vpX = vanishingPointX.value * canvasWidth / 100;
  const vpY = vanishingPointY.value * canvasHeight / 100;

  // Calculate distance from vanishing point to mouse position
  const dx = point.x - vpX;
  const dy = point.y - vpY;
  const distance = Math.sqrt(dx * dx + dy * dy);

  // Update radius based on distance from vanishing point
  // Clamp to valid range (0-1000 based on input range)
  radius.value = Math.max(0, Math.min(1000, distance));
}

function stopDragLeniency(event: MouseEvent | TouchEvent) {
  event.preventDefault();
  event.stopPropagation();
  isDraggingLeniency.value = false;
}

function stopDragRadius(event: MouseEvent | TouchEvent) {
  event.preventDefault();
  event.stopPropagation();
  isDraggingRadius.value = false;
}

function handleMouseMove(event: MouseEvent | TouchEvent) {
  if (isDraggingLeniency.value) {
    dragLeniency(event);
  } else if (isDraggingRadius.value) {
    dragRadius(event);
  } else if (isDragging.value) {
    drag(event);
  }
}

function handleMouseUp(event: MouseEvent | TouchEvent) {
  if (isDraggingLeniency.value) {
    stopDragLeniency(event);
  } else if (isDraggingRadius.value) {
    stopDragRadius(event);
  } else if (isDragging.value) {
    stopDrag(event);
  }
}

function drawSpeedLines() {
  const ctx = speedLineCanvas.value?.getContext('2d');
  if (ctx) {
    ctx.clearRect(0, 0, canvasWidth, canvasHeight);

    const vpX = vanishingPointX.value * canvasWidth / 100;
    const vpY = vanishingPointY.value * canvasHeight / 100;
    const r = radius.value;

    ctx.fillStyle = '#000';

    // Calculate perimeter for golden ratio distribution
    const perimeter = 2 * canvasWidth + 2 * canvasHeight;
    const goldenRatio = 1.618033988749895; // Golden ratio (phi)

    for (let i = 0; i < speedLineCount.value; i++) {
      // Use golden ratio to distribute points evenly along the perimeter
      // This creates a sequence that naturally fills gaps (Fibonacci-style)
      // Multiply by golden ratio and take modulo 1 to get normalized position, then scale to perimeter
      const normalizedPosition = (i * goldenRatio) % 1.0;
      const perimeterPosition = normalizedPosition * perimeter;

      // Convert perimeter position to x,y coordinates on the edge
      let startX: number, startY: number;

      if (perimeterPosition < canvasWidth) {
        // Top edge (left to right)
        startX = perimeterPosition;
        startY = 0;
      } else if (perimeterPosition < canvasWidth + canvasHeight) {
        // Right edge (top to bottom)
        startX = canvasWidth;
        startY = perimeterPosition - canvasWidth;
      } else if (perimeterPosition < 2 * canvasWidth + canvasHeight) {
        // Bottom edge (right to left)
        startX = canvasWidth - (perimeterPosition - canvasWidth - canvasHeight);
        startY = canvasHeight;
      } else {
        // Left edge (bottom to top)
        startX = 0;
        startY = canvasHeight - (perimeterPosition - 2 * canvasWidth - canvasHeight);
      }

      // draw towards the vanishing point, stopping at the radius
      const dx = vpX - startX;
      const dy = vpY - startY;
      const distance = Math.sqrt(dx * dx + dy * dy);

      if (distance > 0) {
        // Normalize direction vector
        const dirX = dx / distance;
        const dirY = dy / distance;

        // Calculate perpendicular vector for width offset
        const perpX = -dirY;
        const perpY = dirX;

        // Calculate the goal travel distance (stopping at radius distance from vanishing point)
        const goalTravelDistance = Math.max(0, distance - r);

        // Add random leniency to the travel distance (can be longer or shorter)
        const leniencyOffset = (Math.random() * 2 - 1) * lengthLeniency.value; // Random between -lengthLeniency and +lengthLeniency
        const travelDistance = Math.max(0, Math.min(distance, goalTravelDistance + leniencyOffset)); // Clamp between 0 and total distance

        // Calculate end point along the line
        const endX = startX + dirX * travelDistance;
        const endY = startY + dirY * travelDistance;

        // Create a quadrilateral shape: wider at start (maxWidth), narrower at end (minWidth)
        const startWidthHalf = maxWidth.value / 2;
        const endWidthHalf = minWidth.value / 2;

        // Four corners of the trapezoid
        const p1X = startX + perpX * startWidthHalf;
        const p1Y = startY + perpY * startWidthHalf;
        const p2X = startX - perpX * startWidthHalf;
        const p2Y = startY - perpY * startWidthHalf;
        const p3X = endX - perpX * endWidthHalf;
        const p3Y = endY - perpY * endWidthHalf;
        const p4X = endX + perpX * endWidthHalf;
        const p4Y = endY + perpY * endWidthHalf;

        ctx.beginPath();
        ctx.moveTo(p1X, p1Y);
        ctx.lineTo(p2X, p2Y);
        ctx.lineTo(p3X, p3Y);
        ctx.lineTo(p4X, p4Y);
        ctx.closePath();
        ctx.fill();
      }
    }
  }
}


</script>

<template>
  <div id="container">
    <div id="controls">
      <div class="vanishing-point">
        <label for="vanishingPointX">Vanishing Point X</label>
        <input type="range" min="0" max="100" v-model.number="vanishingPointX" /><span>{{ vanishingPointX }}</span>
        <label for="vanishingPointY">Vanishing Point Y</label>
        <input type="range" min="0" max="100" v-model.number="vanishingPointY" /><span>{{ vanishingPointY }}</span>
        <label for="radius">Radius</label>
        <input type="range" min="0" max="1000" v-model.number="radius" /><span>{{ radius }}</span>
        <label for="speedLineCount">Speed Line Count</label>
        <input type="range" min="10" max="1000" v-model.number="speedLineCount" /><span>{{ speedLineCount }}</span>
        <label for="minWidth">Min Width</label>
        <input type="range" min="1" max="10" v-model.number="minWidth" /><span>{{ minWidth }}</span>
        <label for="maxWidth">Max Width</label>
        <input type="range" min="1" max="50" v-model.number="maxWidth" /><span>{{ maxWidth }}</span>
        <label for="lengthLeniency">Length Leniency</label>
        <input type="range" min="1" max="100" v-model.number="lengthLeniency" /><span>{{ lengthLeniency }}</span>
      </div>
      <div>
        <button @click="drawSpeedLines">Draw Speed Lines</button>
      </div>
    </div>
    <div id="output">
      <svg ref="svgElement" :width="canvasWidth" :height="canvasHeight" :viewBox="`0 0 ${canvasWidth} ${canvasHeight}`"
        @mousemove="handleMouseMove" @mouseup="handleMouseUp" @mouseleave="handleMouseUp" @touchmove="handleMouseMove"
        @touchend="handleMouseUp">
        <!-- Outer variance circle (radius + lengthLeniency) -->
        <circle :cx="vanishingPointX * canvasWidth / 100" :cy="vanishingPointY * canvasHeight / 100"
          :r="radius + lengthLeniency" fill="none" stroke="#888" stroke-width="1" stroke-dasharray="4 4" />

        <!-- Leniency handle circle (on the edge of outer variance circle) -->
        <circle :cx="vanishingPointX * canvasWidth / 100 + radius + lengthLeniency"
          :cy="vanishingPointY * canvasHeight / 100" r="6" fill="#666" stroke="#333" stroke-width="2"
          @mousedown="startDragLeniency" @touchstart="startDragLeniency"
          :style="{ cursor: isDraggingLeniency ? 'grabbing' : 'grab' }" />
        <!-- Main radius circle -->
        <circle :cx="vanishingPointX * canvasWidth / 100" :cy="vanishingPointY * canvasHeight / 100" :r="radius"
          @mousedown="startDrag" @touchstart="startDrag" :style="{ cursor: isDragging ? 'grabbing' : 'grab' }" />
        <!-- Radius handle circle (on the edge of main radius circle) -->
        <circle :cx="vanishingPointX * canvasWidth / 100 + radius" :cy="vanishingPointY * canvasHeight / 100" r="6"
          fill="#999" stroke="#333" stroke-width="2" @mousedown="startDragRadius" @touchstart="startDragRadius"
          :style="{ cursor: isDraggingRadius ? 'grabbing' : 'grab' }" />
        <!-- Inner variance circle (radius - lengthLeniency) -->
        <circle class="pointer-ignore" :cx="vanishingPointX * canvasWidth / 100"
          :cy="vanishingPointY * canvasHeight / 100" :r="Math.max(0, radius - lengthLeniency)" fill="none" stroke="#888"
          stroke-width="1" stroke-dasharray="4 4" />
      </svg>
      <canvas ref="speedLineCanvas" id="speed-lines" :width="canvasWidth" :height="canvasHeight"></canvas>
    </div>
  </div>


</template>

<style scoped>
svg {
  width: 800px;
  height: 600px;
  border: 1px solid black;
  position: relative;
  user-select: none;
}

circle {
  pointer-events: all;
}

#speed-lines {
  border: 1px solid red;
}

.vanishing-point {
  display: flex;
  flex-direction: column;
  gap: 10px;
  width: 200px;
}

#container {
  display: flex;
  flex-direction: row;
  gap: 10px;
  width: 100%;
}

.pointer-ignore {
  pointer-events: none;
}
</style>
