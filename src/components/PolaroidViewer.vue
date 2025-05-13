<template>
  <div class="container">
    <div class="polaroid" ref="polaroid" @click="triggerFileInput">
      <div class="image-container">
        <canvas
          ref="canvas"
          class="canvas"
          :class="{ 'image-loaded': hasImage }"
        ></canvas>
      </div>
      <div class="polaroid-text">Rainbow</div>
      <div class="file-input-container" v-if="!hasImage">
        <input
          type="file"
          ref="imageInput"
          accept="image/*"
          @change="handleImageUpload"
          class="file-input"
        />
      </div>
      <div class="message-container">
        <textarea
          v-model="userMessage"
          placeholder="Type your message here..."
          class="message-input"
          @input="adjustTextareaHeight"
          ref="messageInput"
        ></textarea>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, watch } from "vue";

// Reactive state
const polaroid = ref(null);
const canvas = ref(null);
const imageInput = ref(null);
const messageInput = ref(null);
const hasImage = ref(false);
const userMessage = ref("");

// Canvas and effect variables
let ctx = null;
let image = new Image();
let noiseTexture = null;
let tiltX = 0,
  tiltY = 0,
  prevTiltX = 0,
  prevTiltY = 0;
let animationFrameId = null;

// Initialize canvas and event listeners
onMounted(() => {
  ctx = canvas.value.getContext("2d");
  resizeCanvas();
  window.addEventListener("resize", resizeCanvas);
  window.addEventListener("deviceorientation", handleDeviceOrientation);

  // Request permission for iOS devices
  if (typeof DeviceOrientationEvent.requestPermission === "function") {
    DeviceOrientationEvent.requestPermission()
      .then((permissionState) => {
        if (permissionState === "granted") {
          console.log("Device orientation permission granted");
        }
      })
      .catch(console.error);
  }

  // Start animation loop
  updateTilt();

  // Fix for first click issue - pre-initialize the file input
  imageInput.value.addEventListener("click", (e) => {
    e.stopPropagation();
  });
});

// Clean up event listeners
onUnmounted(() => {
  window.removeEventListener("resize", resizeCanvas);
  window.removeEventListener("deviceorientation", handleDeviceOrientation);
  cancelAnimationFrame(animationFrameId);
});

// Watch for image changes to update the background
watch(hasImage, (newValue) => {
  if (newValue && image.src) {
    document.body.style.setProperty("--bg-image", `url(${image.src})`);
  }
});

// Functions
function resizeCanvas() {
  if (!canvas.value || !polaroid.value) return;

  const polaroidWidth = polaroid.value.offsetWidth;
  const canvasSize = polaroidWidth - 40; // Account for 20px padding on each side
  canvas.value.width = canvasSize;
  canvas.value.height = canvasSize;

  if (image.src) drawImage();
  noiseTexture = createNoiseTexture(canvas.value.width, canvas.value.height);
}

function triggerFileInput() {
  // Only trigger file input if we're not clicking in the message area
  if (!hasImage.value || event.target !== messageInput.value) {
    imageInput.value.click();
  }
}

function handleImageUpload(e) {
  const file = e.target.files[0];
  if (file) {
    const reader = new FileReader();
    reader.onload = (event) => {
      image = new Image();
      image.onload = () => {
        hasImage.value = true;
        drawImage();
      };
      image.src = event.target.result;
    };
    reader.readAsDataURL(file);
  }
}

function handleDeviceOrientation(e) {
  tiltX = Math.min(Math.max(e.gamma / 30, -1), 1);
  tiltY = Math.min(Math.max(e.beta / 30, -1), 1);
}

function createNoiseTexture(width, height) {
  const noiseCanvas = document.createElement("canvas");
  noiseCanvas.width = width;
  noiseCanvas.height = height;
  const noiseCtx = noiseCanvas.getContext("2d");
  const imageData = noiseCtx.createImageData(width, height);

  for (let i = 0; i < imageData.data.length; i += 4) {
    const value = Math.random() * 255;
    imageData.data[i] = value;
    imageData.data[i + 1] = value;
    imageData.data[i + 2] = value;
    imageData.data[i + 3] = 20;
  }

  noiseCtx.putImageData(imageData, 0, 0);
  return noiseCanvas;
}

function drawImage() {
  if (!ctx || !canvas.value) return;

  ctx.clearRect(0, 0, canvas.value.width, canvas.value.height);

  // Draw image with preserved aspect ratio (object-fit: cover equivalent)
  const imgWidth = image.width;
  const imgHeight = image.height;
  const canvasWidth = canvas.value.width;
  const canvasHeight = canvas.value.height;

  // Calculate dimensions to maintain aspect ratio while covering the canvas
  let drawWidth,
    drawHeight,
    offsetX = 0,
    offsetY = 0;

  const imgRatio = imgWidth / imgHeight;
  const canvasRatio = canvasWidth / canvasHeight;

  if (imgRatio > canvasRatio) {
    // Image is wider than canvas (relative to height)
    drawHeight = canvasHeight;
    drawWidth = drawHeight * imgRatio;
    offsetX = (canvasWidth - drawWidth) / 2;
  } else {
    // Image is taller than canvas (relative to width)
    drawWidth = canvasWidth;
    drawHeight = drawWidth / imgRatio;
    offsetY = (canvasHeight - drawHeight) / 2;
  }

  ctx.drawImage(image, offsetX, offsetY, drawWidth, drawHeight);
  applySunCatcherEffect();
}

function applySunCatcherEffect() {
  if (!ctx || !canvas.value) return;

  ctx.save();
  const centerX = canvas.value.width / 2 + tiltX * 50;
  const centerY = canvas.value.height / 2 + tiltY * 50;

  const gradient = ctx.createRadialGradient(
    centerX,
    centerY,
    10,
    centerX,
    centerY,
    canvas.value.width / 1.5
  );

  const hueShift = (Math.abs(tiltX) + Math.abs(tiltY)) * 50;
  gradient.addColorStop(0, `hsla(${hueShift}, 80%, 70%, 0.4)`);
  gradient.addColorStop(0.3, `hsla(${hueShift + 60}, 80%, 70%, 0.3)`);
  gradient.addColorStop(0.6, `hsla(${hueShift + 120}, 80%, 70%, 0.3)`);
  gradient.addColorStop(1, `hsla(${hueShift + 180}, 80%, 70%, 0.2)`);

  ctx.fillStyle = gradient;
  ctx.fillRect(0, 0, canvas.value.width, canvas.value.height);

  // Apply noise texture
  if (noiseTexture) {
    ctx.globalAlpha = 0.1;
    ctx.drawImage(noiseTexture, 0, 0);
    ctx.globalAlpha = 1;
  }

  // Sparkles only when tilting
  const tiltChange = Math.abs(tiltX - prevTiltX) + Math.abs(tiltY - prevTiltY);
  if (tiltChange > 0.02 && Math.random() < 0.1) {
    const sparkleX = centerX + (Math.random() - 0.5) * 50;
    const sparkleY = centerY + (Math.random() - 0.5) * 50;
    const sparkleGradient = ctx.createRadialGradient(
      sparkleX,
      sparkleY,
      0,
      sparkleX,
      sparkleY,
      10
    );
    sparkleGradient.addColorStop(0, "rgba(255, 255, 255, 0.8)");
    sparkleGradient.addColorStop(1, "rgba(255, 255, 255, 0)");
    ctx.fillStyle = sparkleGradient;
    ctx.beginPath();
    ctx.arc(sparkleX, sparkleY, 10, 0, Math.PI * 2);
    ctx.fill();
  }

  ctx.restore();
}

function updateTilt() {
  if (polaroid.value) {
    polaroid.value.style.transform = `perspective(1000px) rotateX(${
      tiltY * 10
    }deg) rotateY(${tiltX * 10}deg)`;
  }

  if (hasImage.value && image.src) {
    drawImage();
  }

  prevTiltX = tiltX;
  prevTiltY = tiltY;
  animationFrameId = requestAnimationFrame(updateTilt);
}

function adjustTextareaHeight() {
  if (!messageInput.value) return;

  // Reset height to auto to get the correct scrollHeight
  messageInput.value.style.height = "auto";

  // Set the height to match content (with a max height)
  const newHeight = Math.min(messageInput.value.scrollHeight, 100);
  messageInput.value.style.height = `${newHeight}px`;
}
</script>

<style>
body {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
  margin: 0;
  background-color: #f0f0f0;
  font-family: Arial, sans-serif;
  position: relative;
  --bg-image: none; /* Default background image */
}

/* Pseudo-element for grayscale background */
body::before {
  content: "";
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-image: var(--bg-image);
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  filter: grayscale(100%);
  z-index: -1;
  transition: background-image 0.3s;
}

.container {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 95vw;
  box-sizing: border-box;
}

.polaroid {
  background: white;
  padding: 20px 20px 80px 20px; /* Extra bottom padding for Polaroid effect */
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  transition: transform 0.1s;
  width: 90vw;
  max-width: 500px;
  position: relative;
  text-align: center;
  filter: none;
  z-index: 1;
  cursor: pointer; /* Indicates clickable */
}

.image-container {
  width: 100%;
  position: relative;
}

.canvas {
  display: block;
  width: 100%;
}

.polaroid-text {
  font-family: "Inter", sans-serif;
  font-weight: 200;
  font-size: 40px;
  position: absolute;
  top: -48px; /* Position above Polaroid */
  left: 0;
  margin: 0;
  text-transform: uppercase;
  background: linear-gradient(
    to right,
    red,
    orange,
    yellow,
    green,
    blue,
    indigo,
    violet
  );
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  color: transparent;
}

.file-input-container {
  position: absolute;
  top: 20px; /* Account for top padding */
  left: 20px; /* Account for left padding */
  right: 20px;
  bottom: 100px; /* Account for bottom padding */
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 2;
}

.file-input {
  opacity: 1;
  cursor: pointer;
}

.message-container {
  position: absolute;
  bottom: 10px;
  left: 20px;
  right: 20px;
  padding: 5px;
  z-index: 3;
}

.message-input {
  width: 100%;
  min-height: 40px;
  max-height: 100px;
  border: none;
  background: transparent;
  font-family: "Inter", sans-serif;
  font-size: 20px;
  font-weight: 200;
  resize: none;
  outline: none;
  text-align: left;
  overflow-y: auto;
}

.message-input::placeholder {
  color: #aaa;
}
</style>
