<script setup lang="ts">
import { ref, watch, onMounted, nextTick } from 'vue';

const vanishingPointX = ref(50);
const vanishingPointY = ref(50);
const radius = ref(150);
const speedLineCount = ref(100);
const minWidth = ref(1);
const maxWidth = ref(5);
const outerLengthLeniency = ref(50); // number of pixels that can extend beyond the radius
const innerLengthLeniency = ref(50); // number of pixels that can be shorter than the radius
const outerEdge = ref(50); // margin/padding outside the output canvas for line starting points
const seed = ref(12345); // seed for deterministic random variance

const canvasWidth = 800;
const canvasHeight = 600;

// Output canvas box dimensions (initially matches canvas size)
const outputCanvasWidth = ref(800);
const outputCanvasHeight = ref(600);
// Center the box in the SVG
const outputCanvasX = ref((canvasWidth - 800) / 2);
const outputCanvasY = ref((canvasHeight - 600) / 2);

const speedLineCanvas = ref<HTMLCanvasElement | null>(null);
const svgElement = ref<SVGSVGElement | null>(null);
const isDragging = ref(false);
const isDraggingOuterLeniency = ref(false);
const isDraggingInnerLeniency = ref(false);
const isDraggingRadius = ref(false);
const isDraggingOutputWidth = ref(false);
const isDraggingOutputHeight = ref(false);
const isDraggingOuterEdge = ref(false);
const dragOffset = ref<{ x: number; y: number } | null>(null);
const threshold = ref(50);
const isAntiAliasing = ref(true);

onMounted(() => {
  drawSpeedLines();
});

watch([vanishingPointX, vanishingPointY, radius, speedLineCount, minWidth, maxWidth, outerLengthLeniency, innerLengthLeniency, threshold, isAntiAliasing, outputCanvasWidth, outputCanvasHeight, outputCanvasX, outputCanvasY, seed, outerEdge], async () => {
  await nextTick();
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
  if (isDraggingOuterLeniency.value || isDraggingInnerLeniency.value || isDraggingRadius.value || isDraggingOutputWidth.value || isDraggingOutputHeight.value || isDraggingOuterEdge.value) return;
  event.preventDefault();

  const point = getSVGPoint(event);
  if (!point) return;

  // Calculate offset from click position to vanishing point center
  const vpX = vanishingPointX.value * canvasWidth / 100;
  const vpY = vanishingPointY.value * canvasHeight / 100;
  dragOffset.value = {
    x: point.x - vpX,
    y: point.y - vpY
  };

  isDragging.value = true;
}

function drag(event: MouseEvent | TouchEvent) {
  if (!isDragging.value || isDraggingOuterLeniency.value || isDraggingInnerLeniency.value || isDraggingRadius.value || isDraggingOutputWidth.value || isDraggingOutputHeight.value || isDraggingOuterEdge.value || !dragOffset.value) return;
  event.preventDefault();

  const point = getSVGPoint(event);
  if (!point) return;

  // Calculate new vanishing point position by subtracting the stored offset
  const newVpX = point.x - dragOffset.value.x;
  const newVpY = point.y - dragOffset.value.y;

  // Convert SVG coordinates to percentage values
  const xPercent = (newVpX / canvasWidth) * 100;
  const yPercent = (newVpY / canvasHeight) * 100;

  // Clamp values to valid range (0-100)
  vanishingPointX.value = Math.max(0, Math.min(100, xPercent));
  vanishingPointY.value = Math.max(0, Math.min(100, yPercent));
}

function stopDrag(event: MouseEvent | TouchEvent) {
  if (isDraggingOuterLeniency.value || isDraggingInnerLeniency.value || isDraggingRadius.value || isDraggingOutputWidth.value || isDraggingOutputHeight.value || isDraggingOuterEdge.value) return;
  event.preventDefault();
  isDragging.value = false;
  dragOffset.value = null;
}

function startDragOuterLeniency(event: MouseEvent | TouchEvent) {
  event.preventDefault();
  event.stopPropagation();
  isDraggingOuterLeniency.value = true;
}

function startDragInnerLeniency(event: MouseEvent | TouchEvent) {
  event.preventDefault();
  event.stopPropagation();
  isDraggingInnerLeniency.value = true;
}

function startDragRadius(event: MouseEvent | TouchEvent) {
  event.preventDefault();
  event.stopPropagation();
  isDraggingRadius.value = true;
}

function dragOuterLeniency(event: MouseEvent | TouchEvent) {
  if (!isDraggingOuterLeniency.value) return;
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

  // Update outerLengthLeniency based on distance from vanishing point
  // outerLengthLeniency = distance - radius
  const newLeniency = distance - radius.value;

  // Clamp to valid range (1-500 based on input range)
  outerLengthLeniency.value = Math.max(1, Math.min(500, newLeniency));
}

function dragInnerLeniency(event: MouseEvent | TouchEvent) {
  if (!isDraggingInnerLeniency.value) return;
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

  // Update innerLengthLeniency based on distance from vanishing point
  // innerLengthLeniency = radius - distance (when distance < radius)
  const newLeniency = radius.value - distance;

  // Clamp to valid range (1-500 based on input range)
  // Also ensure it doesn't exceed the radius
  innerLengthLeniency.value = Math.max(1, Math.min(500, Math.max(0, newLeniency)));
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

function stopDragOuterLeniency(event: MouseEvent | TouchEvent) {
  event.preventDefault();
  event.stopPropagation();
  isDraggingOuterLeniency.value = false;
}

function stopDragInnerLeniency(event: MouseEvent | TouchEvent) {
  event.preventDefault();
  event.stopPropagation();
  isDraggingInnerLeniency.value = false;
}

function stopDragRadius(event: MouseEvent | TouchEvent) {
  event.preventDefault();
  event.stopPropagation();
  isDraggingRadius.value = false;
}

function startDragOutputWidth(event: MouseEvent | TouchEvent) {
  event.preventDefault();
  event.stopPropagation();
  isDraggingOutputWidth.value = true;
}

function startDragOutputHeight(event: MouseEvent | TouchEvent) {
  event.preventDefault();
  event.stopPropagation();
  isDraggingOutputHeight.value = true;
}

function dragOutputWidth(event: MouseEvent | TouchEvent) {
  if (!isDraggingOutputWidth.value) return;
  event.preventDefault();
  event.stopPropagation();

  const point = getSVGPoint(event);
  if (!point) return;

  // Calculate center of the box
  const centerX = outputCanvasX.value + outputCanvasWidth.value / 2;

  // Calculate distance from center to mouse (doubled to get full width)
  const distanceFromCenter = Math.abs(point.x - centerX);
  const newWidth = Math.max(50, Math.min(canvasWidth, distanceFromCenter * 2));

  // Update width and x position to keep box centered
  outputCanvasWidth.value = newWidth;
  outputCanvasX.value = (canvasWidth - newWidth) / 2;
}

function dragOutputHeight(event: MouseEvent | TouchEvent) {
  if (!isDraggingOutputHeight.value) return;
  event.preventDefault();
  event.stopPropagation();

  const point = getSVGPoint(event);
  if (!point) return;

  // Calculate center of the box
  const centerY = outputCanvasY.value + outputCanvasHeight.value / 2;

  // Calculate distance from center to mouse (doubled to get full height)
  const distanceFromCenter = Math.abs(point.y - centerY);
  const newHeight = Math.max(50, Math.min(canvasHeight, distanceFromCenter * 2));

  // Update height and y position to keep box centered
  outputCanvasHeight.value = newHeight;
  outputCanvasY.value = (canvasHeight - newHeight) / 2;
}

function stopDragOutputWidth(event: MouseEvent | TouchEvent) {
  event.preventDefault();
  event.stopPropagation();
  isDraggingOutputWidth.value = false;
}

function stopDragOutputHeight(event: MouseEvent | TouchEvent) {
  event.preventDefault();
  event.stopPropagation();
  isDraggingOutputHeight.value = false;
}

function startDragOuterEdge(event: MouseEvent | TouchEvent) {
  event.preventDefault();
  event.stopPropagation();
  isDraggingOuterEdge.value = true;
}

function dragOuterEdge(event: MouseEvent | TouchEvent) {
  if (!isDraggingOuterEdge.value) return;
  event.preventDefault();
  event.stopPropagation();

  const point = getSVGPoint(event);
  if (!point) return;

  // Calculate distances from output canvas edges
  const distFromLeft = outputCanvasX.value - point.x;
  const distFromRight = point.x - (outputCanvasX.value + outputCanvasWidth.value);
  const distFromTop = outputCanvasY.value - point.y;
  const distFromBottom = point.y - (outputCanvasY.value + outputCanvasHeight.value);

  // Use the maximum distance (furthest outside edge) to determine outer edge
  // This ensures the outer box always encompasses the mouse position
  const newOuterEdge = Math.max(
    Math.max(0, distFromLeft),
    Math.max(0, distFromRight),
    Math.max(0, distFromTop),
    Math.max(0, distFromBottom)
  );

  // Clamp to reasonable range (0-200)
  outerEdge.value = Math.max(0, Math.min(200, newOuterEdge));
}

function stopDragOuterEdge(event: MouseEvent | TouchEvent) {
  event.preventDefault();
  event.stopPropagation();
  isDraggingOuterEdge.value = false;
}

function handleMouseMove(event: MouseEvent | TouchEvent) {
  if (isDraggingOuterLeniency.value) {
    dragOuterLeniency(event);
  } else if (isDraggingInnerLeniency.value) {
    dragInnerLeniency(event);
  } else if (isDraggingRadius.value) {
    dragRadius(event);
  } else if (isDraggingOutputWidth.value) {
    dragOutputWidth(event);
  } else if (isDraggingOutputHeight.value) {
    dragOutputHeight(event);
  } else if (isDraggingOuterEdge.value) {
    dragOuterEdge(event);
  } else if (isDragging.value) {
    drag(event);
  }
}

function handleMouseUp(event: MouseEvent | TouchEvent) {
  if (isDraggingOuterLeniency.value) {
    stopDragOuterLeniency(event);
  } else if (isDraggingInnerLeniency.value) {
    stopDragInnerLeniency(event);
  } else if (isDraggingRadius.value) {
    stopDragRadius(event);
  } else if (isDraggingOutputWidth.value) {
    stopDragOutputWidth(event);
  } else if (isDraggingOutputHeight.value) {
    stopDragOutputHeight(event);
  } else if (isDraggingOuterEdge.value) {
    stopDragOuterEdge(event);
  } else if (isDragging.value) {
    stopDrag(event);
  }
}

// Seeded random number generator using Linear Congruential Generator
function seededRandom(seed: number): () => number {
  let state = seed;
  return function () {
    // LCG parameters (same as used in many programming languages)
    state = (state * 1664525 + 1013904223) % Math.pow(2, 32);
    return state / Math.pow(2, 32);
  };
}

function drawSpeedLines() {
  const canvas = speedLineCanvas.value;
  if (!canvas) return;

  // Ensure canvas dimensions match the reactive values
  if (canvas.width !== outputCanvasWidth.value || canvas.height !== outputCanvasHeight.value) {
    canvas.width = outputCanvasWidth.value;
    canvas.height = outputCanvasHeight.value;
  }

  const ctx = canvas.getContext('2d');
  if (ctx) {

    ctx.fillStyle = '#fff';
    ctx.fillRect(0, 0, outputCanvasWidth.value, outputCanvasHeight.value);

    // Convert vanishing point from SVG coordinates to canvas coordinates
    // The vanishing point is in SVG space, but we need it in canvas space
    const vpXSVG = vanishingPointX.value * canvasWidth / 100;
    const vpYSVG = vanishingPointY.value * canvasHeight / 100;
    // Convert to canvas coordinates by subtracting the canvas offset
    const vpX = vpXSVG - outputCanvasX.value;
    const vpY = vpYSVG - outputCanvasY.value;
    const r = radius.value;

    ctx.fillStyle = '#000';

    // Use outer box dimensions for drawing (output canvas + outer edge margin)
    const outerBoxWidth = outputCanvasWidth.value + 2 * outerEdge.value;
    const outerBoxHeight = outputCanvasHeight.value + 2 * outerEdge.value;

    // Calculate perimeter for golden ratio distribution (using outer box)
    const perimeter = 2 * outerBoxWidth + 2 * outerBoxHeight;
    const goldenRatio = 1.618033988749895; // Golden ratio (phi)

    // Create seeded random generator for this draw call
    // Combine seed with line count to ensure same seed + same line count = same pattern
    const rng = seededRandom(seed.value + speedLineCount.value * 1000);

    for (let i = 0; i < speedLineCount.value; i++) {
      // Use golden ratio to distribute points evenly along the perimeter
      // This creates a sequence that naturally fills gaps (Fibonacci-style)
      // Multiply by golden ratio and take modulo 1 to get normalized position, then scale to perimeter
      const normalizedPosition = (i * goldenRatio) % 1.0;
      const perimeterPosition = normalizedPosition * perimeter;

      // Convert perimeter position to x,y coordinates on the outer box edge
      // Note: These coordinates are relative to the canvas, not the output canvas
      // We need to account for the outer box position
      let startXRelative: number, startYRelative: number;

      if (perimeterPosition < outerBoxWidth) {
        // Top edge (left to right)
        startXRelative = perimeterPosition;
        startYRelative = 0;
      } else if (perimeterPosition < outerBoxWidth + outerBoxHeight) {
        // Right edge (top to bottom)
        startXRelative = outerBoxWidth;
        startYRelative = perimeterPosition - outerBoxWidth;
      } else if (perimeterPosition < 2 * outerBoxWidth + outerBoxHeight) {
        // Bottom edge (right to left)
        startXRelative = outerBoxWidth - (perimeterPosition - outerBoxWidth - outerBoxHeight);
        startYRelative = outerBoxHeight;
      } else {
        // Left edge (bottom to top)
        startXRelative = 0;
        startYRelative = outerBoxHeight - (perimeterPosition - 2 * outerBoxWidth - outerBoxHeight);
      }

      // Convert from outer box coordinates to canvas coordinates
      const outerBoxX = outputCanvasX.value - outerEdge.value;
      const outerBoxY = outputCanvasY.value - outerEdge.value;
      const startXCanvas = outerBoxX + startXRelative;
      const startYCanvas = outerBoxY + startYRelative;

      // Convert to canvas coordinates (relative to output canvas)
      const startX = startXCanvas - outputCanvasX.value;
      const startY = startYCanvas - outputCanvasY.value;

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

        // Apply separate inner and outer leniency boundaries
        // Lines should never extend beyond (radius + outerLengthLeniency) - furthest from VP
        // Lines should never be shorter than (radius - innerLengthLeniency) - closest to VP
        // Smaller travelDistance = ends further from VP (outer boundary)
        // Larger travelDistance = ends closer to VP (inner boundary)
        const minTravelDistance = Math.max(0, goalTravelDistance - outerLengthLeniency.value);
        const maxTravelDistance = Math.min(distance, goalTravelDistance + innerLengthLeniency.value);

        // Use seeded random to choose a travel distance within the allowed range
        // This ensures the same seed + line index produces the same variance
        const randomValue = rng();
        const travelDistance = minTravelDistance + randomValue * (maxTravelDistance - minTravelDistance);

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
    if (!isAntiAliasing.value) {
      // Apply threshold filter to remove anti-aliasing
      // Convert pixels to black or white based on threshold value
      const imageData = ctx.getImageData(0, 0, outputCanvasWidth.value, outputCanvasHeight.value);
      const data = imageData.data;
      const thresholdValue = threshold.value;

      // Process pixels in a single pass for optimal performance
      // Each pixel has 4 values: R, G, B, A
      for (let i = 0; i < data.length; i += 4) {
        // Calculate grayscale value using standard luminance formula
        // Using integer arithmetic for better performance
        const r = data[i] ?? 0;
        const g = data[i + 1] ?? 0;
        const b = data[i + 2] ?? 0;
        const gray = Math.round(r * 0.299 + g * 0.587 + b * 0.114);

        // Apply threshold: >= threshold becomes white (255), < threshold becomes black (0)
        const value = gray >= thresholdValue ? 255 : 0;

        // Set RGB to the thresholded value, keep alpha channel unchanged
        data[i] = value;     // R
        data[i + 1] = value; // G
        data[i + 2] = value; // B
        // data[i + 3] remains unchanged (alpha)
      }

      // Put the processed image data back to the canvas
      ctx.putImageData(imageData, 0, 0);
    }
  }
}

function toggleAntiAliasing() {
  isAntiAliasing.value = !isAntiAliasing.value;
}



</script>

<template>
  <div id="container">
    <div id="controls">
      <div class="vanishing-point">
        <div>
          <label for="vanishingPointX">Vanishing Point X</label>
          <input type="range" min="0" max="100" v-model.number="vanishingPointX" /><span>{{ vanishingPointX }}</span>
        </div>
        <div>
          <label for="vanishingPointY">Vanishing Point Y</label>
          <input type="range" min="0" max="100" v-model.number="vanishingPointY" /><span>{{ vanishingPointY }}</span>
        </div>
        <div>
          <label for="radius">Radius</label>
          <input type="range" min="0" max="1000" v-model.number="radius" /><span>{{ radius }}</span>
        </div>
        <div>
          <label for="speedLineCount">Speed Line Count</label>
          <input type="range" min="10" max="1000" v-model.number="speedLineCount" /><span>{{ speedLineCount }}</span>
        </div>
        <div>
          <label for="minWidth">Min Width</label>
          <input type="range" min="0" max="10" v-model.number="minWidth" /><span>{{ minWidth }}</span>
        </div>
        <div>
          <label for="maxWidth">Max Width</label>
          <input type="range" min="1" max="50" v-model.number="maxWidth" /><span>{{ maxWidth }}</span>
        </div>
        <div>
          <label for="outerLengthLeniency">Outer Length Leniency</label>
          <input type="range" min="1" max="500" v-model.number="outerLengthLeniency" /><span>{{ outerLengthLeniency
            }}</span>
        </div>
        <div>
          <label for="innerLengthLeniency">Inner Length Leniency</label>
          <input type="range" min="1" max="500" v-model.number="innerLengthLeniency" /><span>{{ innerLengthLeniency
            }}</span>
        </div>
        <div>
          <label for="outerEdge">Outer Edge</label>
          <input type="range" min="0" max="200" v-model.number="outerEdge" /><span>{{ outerEdge }}</span>
        </div>
        <div>
          <label for="seed">Seed</label>
          <input type="number" v-model.number="seed" /><span>{{ seed }}</span>
        </div>
        <div>
          <label for="threshold">Threshold</label>
          <input type="range" min="1" max="255" v-model.number="threshold" /><span>{{ threshold }}</span>
        </div>
      </div>
      <div>
        <button @click="drawSpeedLines">Draw Speed Lines</button>
      </div>
      <div>
        <button @click="toggleAntiAliasing">Toggle Anti Aliasing</button>
      </div>
    </div>
    <div id="output">
      <svg ref="svgElement" :width="canvasWidth" :height="canvasHeight" :viewBox="`0 0 ${canvasWidth} ${canvasHeight}`"
        @mousemove="handleMouseMove" @mouseup="handleMouseUp" @mouseleave="handleMouseUp" @touchmove="handleMouseMove"
        @touchend="handleMouseUp">
        <!-- Main radius circle (interactive, no pointer-ignore) -->
        <circle :cx="vanishingPointX * canvasWidth / 100" :cy="vanishingPointY * canvasHeight / 100" :r="radius"
          @mousedown="startDrag" @touchstart="startDrag" :style="{ cursor: isDragging ? 'grabbing' : 'grab' }" />
        <!-- Background elements (with pointer-ignore) -->
        <!-- Outer edge box (pink rectangle) -->
        <rect :x="outputCanvasX - outerEdge" :y="outputCanvasY - outerEdge" 
          :width="outputCanvasWidth + 2 * outerEdge" :height="outputCanvasHeight + 2 * outerEdge" 
          fill="none" stroke="pink" stroke-width="2" stroke-dasharray="4 4" class="pointer-ignore" />
        <!-- Output canvas box (red rectangle) -->
        <rect :x="outputCanvasX" :y="outputCanvasY" :width="outputCanvasWidth" :height="outputCanvasHeight" fill="none"
          stroke="red" stroke-width="2" stroke-dasharray="4 4" class="pointer-ignore" />
        <!-- Outer variance circle (radius + outerLengthLeniency) -->
        <circle class="pointer-ignore" :cx="vanishingPointX * canvasWidth / 100"
          :cy="vanishingPointY * canvasHeight / 100" :r="radius + outerLengthLeniency" fill="none" stroke="#888"
          stroke-width="1" stroke-dasharray="4 4" />
        <!-- Inner variance circle (radius - innerLengthLeniency) -->
        <circle class="pointer-ignore" :cx="vanishingPointX * canvasWidth / 100"
          :cy="vanishingPointY * canvasHeight / 100" :r="Math.max(0, radius - innerLengthLeniency)" fill="none"
          stroke="#888" stroke-width="1" stroke-dasharray="4 4" />


        <!-- Handles (rendered last, on top) -->
        <!-- Width resize handle (right edge, center) -->
        <circle :cx="outputCanvasX + outputCanvasWidth" :cy="outputCanvasY + outputCanvasHeight / 2" r="6" fill="red"
          stroke="#333" stroke-width="2" @mousedown="startDragOutputWidth" @touchstart="startDragOutputWidth"
          :style="{ cursor: isDraggingOutputWidth ? 'ew-resize' : 'ew-resize' }" />
        <!-- Height resize handle (bottom edge, center) -->
        <circle :cx="outputCanvasX + outputCanvasWidth / 2" :cy="outputCanvasY + outputCanvasHeight" r="6" fill="red"
          stroke="#333" stroke-width="2" @mousedown="startDragOutputHeight" @touchstart="startDragOutputHeight"
          :style="{ cursor: isDraggingOutputHeight ? 'ns-resize' : 'ns-resize' }" />

        <!-- Radius handle circle (on the edge of main radius circle) -->
        <circle :cx="vanishingPointX * canvasWidth / 100 + radius" :cy="vanishingPointY * canvasHeight / 100" r="6"
          fill="#999" stroke="#333" stroke-width="2" @mousedown="startDragRadius" @touchstart="startDragRadius"
          :style="{ cursor: isDraggingRadius ? 'grabbing' : 'grab' }" />
        <!-- Outer leniency handle circle (on the edge of outer variance circle) -->
        <circle :cx="vanishingPointX * canvasWidth / 100 + radius + outerLengthLeniency"
          :cy="vanishingPointY * canvasHeight / 100" r="6" fill="#666" stroke="#333" stroke-width="2"
          @mousedown="startDragOuterLeniency" @touchstart="startDragOuterLeniency"
          :style="{ cursor: isDraggingOuterLeniency ? 'grabbing' : 'grab' }" />
        <!-- Inner leniency handle circle (on the edge of inner variance circle) -->
        <circle :cx="vanishingPointX * canvasWidth / 100 + Math.max(0, radius - innerLengthLeniency)"
          :cy="vanishingPointY * canvasHeight / 100" r="6" fill="#666" stroke="#333" stroke-width="2"
          @mousedown="startDragInnerLeniency" @touchstart="startDragInnerLeniency"
          :style="{ cursor: isDraggingInnerLeniency ? 'grabbing' : 'grab' }" />
        <!-- Outer edge handle (top-left corner of outer box) -->
        <circle :cx="outputCanvasX - outerEdge" :cy="outputCanvasY - outerEdge" r="6" fill="pink" stroke="#333" stroke-width="2"
          @mousedown="startDragOuterEdge" @touchstart="startDragOuterEdge"
          :style="{ cursor: isDraggingOuterEdge ? 'grabbing' : 'grab' }" />
      </svg>
      <canvas ref="speedLineCanvas" id="speed-lines" :width="outputCanvasWidth" :height="outputCanvasHeight"></canvas>
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
  display: block;
  width: auto;
  height: auto;
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
