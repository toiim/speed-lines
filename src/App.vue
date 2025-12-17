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
const outerEdgeShape = ref<'circle' | 'rectangle'>('circle'); // shape of the outer edge
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
const zoomLevel = ref(100); // Zoom percentage (100 = 100%)

onMounted(() => {
  drawSpeedLines();
});

watch([vanishingPointX, vanishingPointY, radius, speedLineCount, minWidth, maxWidth, outerLengthLeniency, innerLengthLeniency, threshold, isAntiAliasing, outputCanvasWidth, outputCanvasHeight, outputCanvasX, outputCanvasY, seed, outerEdge, outerEdgeShape], async () => {
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

  // Get the current viewBox
  const zoomFactor = zoomLevel.value / 100;
  const viewBoxWidth = canvasWidth / zoomFactor;
  const viewBoxHeight = canvasHeight / zoomFactor;
  const centerX = outputCanvasX.value + outputCanvasWidth.value / 2;
  const centerY = outputCanvasY.value + outputCanvasHeight.value / 2;
  const viewBoxX = centerX - viewBoxWidth / 2;
  const viewBoxY = centerY - viewBoxHeight / 2;

  // Convert screen coordinates to SVG coordinates accounting for zoom
  const scaleX = viewBoxWidth / rect.width;
  const scaleY = viewBoxHeight / rect.height;

  const x = (clientX - rect.left) * scaleX + viewBoxX;
  const y = (clientY - rect.top) * scaleY + viewBoxY;

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
  const newWidth = Math.max(50, distanceFromCenter * 2);

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
  const newHeight = Math.max(50, distanceFromCenter * 2);

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

  const centerX = outputCanvasX.value + outputCanvasWidth.value / 2;
  const centerY = outputCanvasY.value + outputCanvasHeight.value / 2;

  if (outerEdgeShape.value === 'circle') {
    // Calculate distance from circle center to mouse position
    const dx = point.x - centerX;
    const dy = point.y - centerY;
    const distanceFromCenter = Math.sqrt(dx * dx + dy * dy);

    // Calculate base radius (half diagonal of output canvas)
    const baseRadius = Math.sqrt((outputCanvasWidth.value / 2) ** 2 + (outputCanvasHeight.value / 2) ** 2);

    // Outer edge is the distance beyond the base radius
    const newOuterEdge = distanceFromCenter - baseRadius;

    // Clamp to reasonable range (0-200)
    outerEdge.value = Math.max(0, Math.min(200, newOuterEdge));
  } else {
    // Rectangle: calculate distance from rectangle edge
    const halfWidth = outputCanvasWidth.value / 2;
    const halfHeight = outputCanvasHeight.value / 2;

    // Calculate the distance from the rectangle edge (expanded by outerEdge)
    // Find the closest edge and calculate margin
    const dx = Math.abs(point.x - centerX);
    const dy = Math.abs(point.y - centerY);

    // For rectangle, outerEdge is the margin added to each side
    // Calculate how far beyond the rectangle the point is
    const marginX = dx - halfWidth;
    const marginY = dy - halfHeight;

    // Use the maximum margin (furthest from rectangle)
    const newOuterEdge = Math.max(0, Math.max(marginX, marginY));

    // Clamp to reasonable range (0-200)
    outerEdge.value = Math.max(0, Math.min(200, newOuterEdge));
  }
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

    // Calculate center (center of output canvas)
    const centerX = outputCanvasX.value + outputCanvasWidth.value / 2;
    const centerY = outputCanvasY.value + outputCanvasHeight.value / 2;

    // Create seeded random generator for this draw call
    // Combine seed with line count to ensure same seed + same line count = same pattern
    const rng = seededRandom(seed.value + speedLineCount.value * 1000);

    for (let i = 0; i < speedLineCount.value; i++) {
      let startXCanvas: number;
      let startYCanvas: number;

      if (outerEdgeShape.value === 'circle') {
        // Calculate circle radius (half diagonal of output canvas + outer edge margin)
        const baseRadius = Math.sqrt((outputCanvasWidth.value / 2) ** 2 + (outputCanvasHeight.value / 2) ** 2);
        const outerCircleRadius = baseRadius + outerEdge.value;

        // Calculate circumference for golden ratio distribution
        const circumference = 2 * Math.PI * outerCircleRadius;
        const goldenRatio = 1.618033988749895; // Golden ratio (phi)

        // Use golden ratio to distribute points evenly along the circle's circumference
        // This creates a sequence that naturally fills gaps (Fibonacci-style)
        // Multiply by golden ratio and take modulo 1 to get normalized position, then scale to circumference
        const normalizedPosition = (i * goldenRatio) % 1.0;
        const arcLength = normalizedPosition * circumference;

        // Convert arc length to angle (in radians)
        const angle = arcLength / outerCircleRadius;

        // Calculate point on circle edge
        startXCanvas = centerX + Math.cos(angle) * outerCircleRadius;
        startYCanvas = centerY + Math.sin(angle) * outerCircleRadius;
      } else {
        // Rectangle: distribute points along the rectangle perimeter
        const halfWidth = outputCanvasWidth.value / 2 + outerEdge.value;
        const halfHeight = outputCanvasHeight.value / 2 + outerEdge.value;

        // Calculate perimeter
        const perimeter = 2 * (halfWidth * 2 + halfHeight * 2);
        const goldenRatio = 1.618033988749895;

        // Use golden ratio to distribute points evenly along the rectangle's perimeter
        const normalizedPosition = (i * goldenRatio) % 1.0;
        const position = normalizedPosition * perimeter;

        // Determine which edge the point is on
        // Top edge: 0 to 2*halfWidth
        // Right edge: 2*halfWidth to 2*halfWidth + 2*halfHeight
        // Bottom edge: 2*halfWidth + 2*halfHeight to 4*halfWidth + 2*halfHeight
        // Left edge: 4*halfWidth + 2*halfHeight to perimeter

        if (position < 2 * halfWidth) {
          // Top edge (left to right)
          startXCanvas = centerX - halfWidth + position;
          startYCanvas = centerY - halfHeight;
        } else if (position < 2 * halfWidth + 2 * halfHeight) {
          // Right edge (top to bottom)
          startXCanvas = centerX + halfWidth;
          startYCanvas = centerY - halfHeight + (position - 2 * halfWidth);
        } else if (position < 4 * halfWidth + 2 * halfHeight) {
          // Bottom edge (right to left)
          startXCanvas = centerX + halfWidth - (position - 2 * halfWidth - 2 * halfHeight);
          startYCanvas = centerY + halfHeight;
        } else {
          // Left edge (bottom to top)
          startXCanvas = centerX - halfWidth;
          startYCanvas = centerY + halfHeight - (position - 4 * halfWidth - 2 * halfHeight);
        }
      }

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

        // Quick check: calculate bounding box and see if it intersects with canvas bounds
        const minX = Math.min(p1X, p2X, p3X, p4X);
        const maxX = Math.max(p1X, p2X, p3X, p4X);
        const minY = Math.min(p1Y, p2Y, p3Y, p4Y);
        const maxY = Math.max(p1Y, p2Y, p3Y, p4Y);

        // Check if bounding box intersects with canvas bounds (0, 0, outputCanvasWidth, outputCanvasHeight)
        if (maxX < 0 || minX > outputCanvasWidth.value || maxY < 0 || minY > outputCanvasHeight.value) {
          // Line is completely outside canvas bounds, skip drawing
          continue;
        }

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

function zoomIn() {
  zoomLevel.value = Math.min(500, zoomLevel.value + 10);
}

function zoomOut() {
  zoomLevel.value = Math.max(25, zoomLevel.value - 10);
}

// Calculate viewBox based on zoom level
// The SVG coordinate space is canvasWidth x canvasHeight
// The viewBox shows a portion of this space, centered on the output canvas area
// When zoom is 100%, viewBox shows full SVG space (canvasWidth x canvasHeight)
// When zoom is 200%, viewBox shows half the SVG space (zoomed in 2x)
function getViewBox() {
  const zoomFactor = zoomLevel.value / 100;
  // Calculate the visible area of the SVG coordinate space based on zoom
  const viewBoxWidth = canvasWidth / zoomFactor;
  const viewBoxHeight = canvasHeight / zoomFactor;
  // Center the viewBox on the output canvas area
  const centerX = outputCanvasX.value + outputCanvasWidth.value / 2;
  const centerY = outputCanvasY.value + outputCanvasHeight.value / 2;
  const viewBoxX = centerX - viewBoxWidth / 2;
  const viewBoxY = centerY - viewBoxHeight / 2;
  return `${viewBoxX} ${viewBoxY} ${viewBoxWidth} ${viewBoxHeight}`;
}



</script>

<template>
  <div id="container">
    <div id="controls">
      <!-- Vanishing Point Section -->
      <div class="control-group">
        <h3 class="group-header">
          <svg class="icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <circle cx="12" cy="12" r="3" />
            <path d="M12 1v6m0 6v6M23 12h-6m-6 0H1" />
          </svg>
          Vanishing Point
        </h3>
        <div class="control-item">
          <label for="vanishingPointX">
            <svg class="icon-small" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <line x1="3" y1="12" x2="21" y2="12" />
              <polyline points="12,3 21,12 12,21" />
            </svg>
            X Position
          </label>
          <div class="slider-container">
            <input type="range" id="vanishingPointX" min="0" max="100" v-model.number="vanishingPointX" />
            <span class="value">{{ vanishingPointX }}</span>
          </div>
        </div>
        <div class="control-item">
          <label for="vanishingPointY">
            <svg class="icon-small" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <line x1="12" y1="3" x2="12" y2="21" />
              <polyline points="3,12 12,3 21,12" />
            </svg>
            Y Position
          </label>
          <div class="slider-container">
            <input type="range" id="vanishingPointY" min="0" max="100" v-model.number="vanishingPointY" />
            <span class="value">{{ vanishingPointY }}</span>
          </div>
        </div>
      </div>

      <!-- Radius & Length Section -->
      <div class="control-group">
        <h3 class="group-header">
          <svg class="icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <circle cx="12" cy="12" r="10" />
          </svg>
          Radius & Length
        </h3>
        <div class="control-item">
          <label for="radius">
            <svg class="icon-small" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <circle cx="12" cy="12" r="10" />
            </svg>
            Radius
          </label>
          <div class="slider-container">
            <input type="range" id="radius" min="0" max="1000" v-model.number="radius" />
            <span class="value">{{ radius }}</span>
          </div>
        </div>
        <div class="control-item">
          <label for="outerLengthLeniency">
            <svg class="icon-small" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <circle cx="12" cy="12" r="10" />
              <circle cx="12" cy="12" r="7" stroke-dasharray="2 2" />
            </svg>
            Outer Variance
          </label>
          <div class="slider-container">
            <input type="range" id="outerLengthLeniency" min="1" max="500" v-model.number="outerLengthLeniency" />
            <span class="value">{{ outerLengthLeniency }}</span>
          </div>
        </div>
        <div class="control-item">
          <label for="innerLengthLeniency">
            <svg class="icon-small" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <circle cx="12" cy="12" r="10" />
              <circle cx="12" cy="12" r="5" stroke-dasharray="2 2" />
            </svg>
            Inner Variance
          </label>
          <div class="slider-container">
            <input type="range" id="innerLengthLeniency" min="1" max="500" v-model.number="innerLengthLeniency" />
            <span class="value">{{ innerLengthLeniency }}</span>
          </div>
        </div>
      </div>

      <!-- Line Properties Section -->
      <div class="control-group">
        <h3 class="group-header">
          <svg class="icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <line x1="2" y1="2" x2="22" y2="22" />
            <line x1="2" y1="22" x2="22" y2="2" />
          </svg>
          Line Properties
        </h3>
        <div class="control-item">
          <label for="speedLineCount">
            <svg class="icon-small" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <line x1="3" y1="12" x2="21" y2="12" />
              <line x1="3" y1="6" x2="21" y2="6" />
              <line x1="3" y1="18" x2="21" y2="18" />
            </svg>
            Line Count
          </label>
          <div class="slider-container">
            <input type="range" id="speedLineCount" min="10" max="1000" v-model.number="speedLineCount" />
            <span class="value">{{ speedLineCount }}</span>
          </div>
        </div>
        <div class="control-item">
          <label for="minWidth">
            <svg class="icon-small" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <line x1="12" y1="2" x2="12" y2="22" />
              <line x1="8" y1="6" x2="16" y2="6" />
            </svg>
            Min Width
          </label>
          <div class="slider-container">
            <input type="range" id="minWidth" step="0.01" min="0" max="10" v-model.number="minWidth" />
            <span class="value">{{ minWidth.toFixed(2) }}</span>
          </div>
        </div>
        <div class="control-item">
          <label for="maxWidth">
            <svg class="icon-small" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <line x1="12" y1="2" x2="12" y2="22" />
              <line x1="6" y1="6" x2="18" y2="6" />
            </svg>
            Max Width
          </label>
          <div class="slider-container">
            <input type="range" id="maxWidth" step="0.01" min="1" max="50" v-model.number="maxWidth" />
            <span class="value">{{ maxWidth.toFixed(2) }}</span>
          </div>
        </div>
        <div class="control-item">
          <label for="outerEdge">
            <svg class="icon-small" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <rect x="3" y="3" width="18" height="18" rx="2" />
              <rect x="1" y="1" width="22" height="22" rx="2" stroke-dasharray="2 2" />
            </svg>
            Line Origin
          </label>
          <div class="slider-container">
            <input type="range" id="outerEdge" min="0" max="200" v-model.number="outerEdge" />
            <span class="value">{{ outerEdge }}</span>
          </div>
        </div>
        <div class="control-item checkbox-item">
          <label for="outerEdgeShape" class="checkbox-label">
            <input type="checkbox" id="outerEdgeShape" :checked="outerEdgeShape === 'circle'"
              @change="outerEdgeShape = outerEdgeShape === 'circle' ? 'rectangle' : 'circle'" />
            <template v-if="outerEdgeShape === 'circle'">
              <svg class="icon-small" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <circle cx="12" cy="12" r="10" />
              </svg>
            </template>
            <template v-else>
              <svg class="icon-small" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <rect x="2" y="2" width="20" height="20" rx="2" />
              </svg>
            </template>
            {{ outerEdgeShape === 'circle' ? 'Circle' : 'Rectangle' }} Outer Edge
          </label>
        </div>
      </div>

      <!-- Output Canvas Section -->
      <div class="control-group">
        <h3 class="group-header">
          <svg class="icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <rect x="3" y="3" width="18" height="18" rx="2" />
          </svg>
          Output Canvas
        </h3>
        <div class="control-item">
          <label for="threshold">
            <svg class="icon-small" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <rect x="2" y="2" width="20" height="20" rx="2" />
              <line x1="6" y1="6" x2="18" y2="18" />
            </svg>
            Threshold
          </label>
          <div class="slider-container">
            <input :disabled="isAntiAliasing" type="range" id="threshold" min="1" max="255"
              v-model.number="threshold" />
            <span class="value">{{ threshold }}</span>
          </div>
        </div>
        <div class="control-item checkbox-item">
          <label for="isAntiAliasing" class="checkbox-label">
            <input type="checkbox" id="isAntiAliasing" v-model="isAntiAliasing" />
            <svg class="icon-small" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M12 2v4m0 12v4M2 12h4m12 0h4" />
            </svg>
            Anti-aliasing
          </label>
        </div>
      </div>

      <!-- Advanced Settings Section -->
      <div class="control-group">
        <h3 class="group-header">
          <svg class="icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <circle cx="12" cy="12" r="3" />
            <path d="M12 1v6m0 6v6M23 12h-6m-6 0H1" />
          </svg>
          Advanced
        </h3>
        <div class="control-item">
          <label for="seed">
            <svg class="icon-small" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M12 2v4m0 12v4M2 12h4m12 0h4" />
              <circle cx="12" cy="12" r="3" />
            </svg>
            Random Seed
          </label>
          <div class="input-container">
            <input type="number" id="seed" v-model.number="seed" />
            <span class="value">{{ seed }}</span>
          </div>
        </div>

      </div>
    </div>
    <div id="output">
      <div class="svg-container">
        <div class="zoom-controls">
          <button @click="zoomOut" class="zoom-button" title="Zoom Out">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <circle cx="12" cy="12" r="10" />
              <line x1="8" y1="12" x2="16" y2="12" />
            </svg>
          </button>
          <span class="zoom-percentage">{{ zoomLevel }}%</span>
          <button @click="zoomIn" class="zoom-button" title="Zoom In">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <circle cx="12" cy="12" r="10" />
              <line x1="12" y1="8" x2="12" y2="16" />
              <line x1="8" y1="12" x2="16" y2="12" />
            </svg>
          </button>
        </div>
        <svg ref="svgElement" :width="canvasWidth" :height="canvasHeight" :viewBox="getViewBox()"
          @mousemove="handleMouseMove" @mouseup="handleMouseUp" @mouseleave="handleMouseUp" @touchmove="handleMouseMove"
          @touchend="handleMouseUp">
          <!-- Main radius circle (interactive, no pointer-ignore) -->
          <circle :cx="vanishingPointX * canvasWidth / 100" :cy="vanishingPointY * canvasHeight / 100" :r="radius"
            @mousedown="startDrag" @touchstart="startDrag" :style="{ cursor: isDragging ? 'grabbing' : 'grab' }" />
          <circle :cx="vanishingPointX * canvasWidth / 100" :cy="vanishingPointY * canvasHeight / 100" :r="2" fill="red"
            class="pointer-ignore" />

          <!-- Background elements (with pointer-ignore) -->
          <!-- Outer edge shape (pink circle or rectangle) -->
          <circle v-if="outerEdgeShape === 'circle'" :cx="outputCanvasX + outputCanvasWidth / 2"
            :cy="outputCanvasY + outputCanvasHeight / 2"
            :r="Math.sqrt((outputCanvasWidth / 2) ** 2 + (outputCanvasHeight / 2) ** 2) + outerEdge" fill="none"
            stroke="pink" stroke-width="2" stroke-dasharray="4 4" class="pointer-ignore" />
          <rect v-else :x="outputCanvasX - outerEdge" :y="outputCanvasY - outerEdge"
            :width="outputCanvasWidth + outerEdge * 2" :height="outputCanvasHeight + outerEdge * 2" fill="none"
            stroke="pink" stroke-width="2" stroke-dasharray="4 4" class="pointer-ignore" />
          <!-- Output canvas box (red rectangle) -->
          <rect :x="outputCanvasX" :y="outputCanvasY" :width="outputCanvasWidth" :height="outputCanvasHeight"
            fill="none" stroke="red" stroke-width="2" stroke-dasharray="4 4" class="pointer-ignore" />
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
          <!-- Outer edge handle -->
          <circle v-if="outerEdgeShape === 'circle'"
            :cx="outputCanvasX + outputCanvasWidth / 2 + (Math.sqrt((outputCanvasWidth / 2) ** 2 + (outputCanvasHeight / 2) ** 2) + outerEdge) * Math.cos(-Math.PI / 4)"
            :cy="outputCanvasY + outputCanvasHeight / 2 + (Math.sqrt((outputCanvasWidth / 2) ** 2 + (outputCanvasHeight / 2) ** 2) + outerEdge) * Math.sin(-Math.PI / 4)"
            r="6" fill="pink" stroke="#333" stroke-width="2" @mousedown="startDragOuterEdge"
            @touchstart="startDragOuterEdge" :style="{ cursor: isDraggingOuterEdge ? 'grabbing' : 'grab' }" />
          <circle v-else :cx="outputCanvasX + outputCanvasWidth + outerEdge"
            :cy="outputCanvasY + outputCanvasHeight / 2" r="6" fill="pink" stroke="#333" stroke-width="2"
            @mousedown="startDragOuterEdge" @touchstart="startDragOuterEdge"
            :style="{ cursor: isDraggingOuterEdge ? 'grabbing' : 'grab' }" />
        </svg>
      </div>
      <canvas ref="speedLineCanvas" id="speed-lines" :width="outputCanvasWidth" :height="outputCanvasHeight"></canvas>
    </div>
  </div>


</template>

<style scoped>

*,
*::before,
*::after {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

.svg-container {
  position: relative;
  display: inline-block;
}

svg {
  width: 800px;
  height: 600px;
  border: 1px solid black;
  position: relative;
  user-select: none;
  display: block;
}

.zoom-controls {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
  padding: 8px;
  background: #f8f9fa;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  width: fit-content;
}

.zoom-button {
  background: #fff;
  border: 1px solid #ccc;
  border-radius: 4px;
  padding: 6px 10px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background-color 0.2s;
}

.zoom-button:hover {
  background: #e9ecef;
}

.zoom-button svg {
  width: 16px;
  height: 16px;
  border: none;
}

.zoom-percentage {
  font-size: 14px;
  font-weight: 600;
  color: #333;
  min-width: 50px;
  text-align: center;
}

circle {
  pointer-events: all;
}

#output {
  display: flex;
  flex: 1;
  overflow: scroll;
  flex-direction: column;
  gap: 10px;
  align-items: start;
}

#speed-lines {
  border: 1px solid red;
  display: block;
  width: auto;
  height: auto;
}

#container {
  font-family: 'Monaco', 'Menlo', 'Courier New', monospace;
  display: flex;
  flex-direction: row;
  gap: 20px;
  width: 100dvw;
  height: 100dvh;
  box-sizing: border-box;
}

#controls {
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  gap: 16px;
  width: 320px;
  overflow-y: auto;
  padding: 20px;
}

#controls::-webkit-scrollbar {
  width: 8px;
}

#controls::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 4px;
}

#controls::-webkit-scrollbar-thumb {
  background: #888;
  border-radius: 4px;
}

#controls::-webkit-scrollbar-thumb:hover {
  background: #555;
}

.control-group {
  background: #f8f9fa;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.group-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin: 0 0 4px 0;
  font-size: 14px;
  font-weight: 600;
  color: #333;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.icon {
  width: 18px;
  height: 18px;
  flex-shrink: 0;
}

.icon-small {
  width: 16px;
  height: 16px;
  flex-shrink: 0;
}


.pointer-ignore {
  pointer-events: none;
}
</style>
