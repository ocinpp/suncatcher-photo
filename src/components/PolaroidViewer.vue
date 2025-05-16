<template>
  <div class="container">
    <div class="polaroid-wrapper">
      <div class="polaroid" ref="polaroid" @click="triggerFileInput">
        <div class="image-container">
          <!-- Skeleton loader that shows while image is loading -->
          <div class="skeleton-loader" v-if="isLoading"></div>
          <canvas
            ref="canvas"
            class="canvas"
            :class="{ 'image-loaded': hasImage }"
          ></canvas>
        </div>
        <div class="polaroid-text">Sunshine</div>

        <!-- Custom file input with message and icon -->
        <div
          class="file-input-container"
          :class="{ 'visually-hidden': hasImage }"
        >
          <div class="file-input-prompt">
            <div class="photo-icon">📷</div>
            <div class="prompt-text">Choose a photo to start</div>
          </div>
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
            @click.stop
          ></textarea>
        </div>
      </div>
    </div>

    <!-- Download button always present but disabled when no image -->
    <button
      class="download-button"
      @click.stop="downloadPolaroid"
      :disabled="!hasImage"
      :class="{ disabled: !hasImage }"
    >
      <span class="button-icon">💾</span>
      Download
    </button>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, watch, nextTick } from "vue";

// Reactive state
const polaroid = ref(null);
const canvas = ref(null);
const imageInput = ref(null);
const messageInput = ref(null);
const hasImage = ref(false);
const isLoading = ref(false);
const userMessage = ref("");
const currentImageSrc = ref("");
let backgroundCanvas = null;

// Canvas and effect variables
let ctx = null;
let image = new Image();
let noiseTexture = null;
let tiltX = 0,
  tiltY = 0,
  prevTiltX = 0,
  prevTiltY = 0;
let animationFrameId = null;
let observer = null;
let bgElement = null;

// Constants
const MAX_BACKGROUND_SIZE = 1200; // Maximum size for background image

// Initialize canvas and event listeners
onMounted(() => {
  ctx = canvas.value.getContext("2d");

  // Set initial dimensions before any content loads
  setInitialDimensions();

  // Initialize canvas with proper dimensions
  resizeCanvas();

  // Set up event listeners
  window.addEventListener("resize", handleResize);
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

  // Set initial height for message input to prevent layout shift
  if (messageInput.value) {
    messageInput.value.style.height = "var(--message-input-height)";
  }

  // Use Intersection Observer to only animate when visible
  observer = new IntersectionObserver(
    (entries) => {
      if (entries[0].isIntersecting) {
        // Start animation loop when visible
        if (!animationFrameId) {
          updateTilt();
        }
      } else {
        // Pause animations when not visible
        if (animationFrameId) {
          cancelAnimationFrame(animationFrameId);
          animationFrameId = null;
        }
      }
    },
    { threshold: 0.1 }
  );

  if (polaroid.value) {
    observer.observe(polaroid.value);
  }

  // Fix for first click issue - pre-initialize the file input
  imageInput.value.addEventListener("click", (e) => {
    e.stopPropagation();
  });

  // Create background element for mobile compatibility
  bgElement = document.createElement("div");
  bgElement.id = "bg-fallback";
  bgElement.className = "background-fallback";
  document.body.appendChild(bgElement);

  // Create a background canvas for resizing large images
  backgroundCanvas = document.createElement("canvas");

  // Apply background image when it changes
  watch(currentImageSrc, (newSrc) => {
    if (newSrc) {
      // Create a resized version of the image for the background
      createResizedBackground(newSrc);
    }
  });
});

// Clean up event listeners
onUnmounted(() => {
  window.removeEventListener("resize", handleResize);
  window.removeEventListener("deviceorientation", handleDeviceOrientation);
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId);
  }
  if (observer) {
    observer.disconnect();
  }

  // Remove fallback background if it exists
  if (bgElement && bgElement.parentNode) {
    bgElement.parentNode.removeChild(bgElement);
  }
});

// Functions
function handleResize() {
  setInitialDimensions();
  resizeCanvas();
}

function setInitialDimensions() {
  // Calculate and set dimensions based on viewport
  const viewportWidth = Math.min(window.innerWidth - 40, 500); // 40px for container padding
  document.documentElement.style.setProperty(
    "--polaroid-width",
    `${viewportWidth}px`
  );

  // Calculate canvas size based on polaroid width
  const canvasSize = viewportWidth - 2 * 20; // 20px padding on each side
  document.documentElement.style.setProperty(
    "--canvas-size",
    `${canvasSize}px`
  );

  // Set polaroid height based on aspect ratio
  const polaroidHeight = viewportWidth * 1.2; // 1.2 aspect ratio
  document.documentElement.style.setProperty(
    "--polaroid-height",
    `${polaroidHeight}px`
  );
}

function resizeCanvas() {
  if (!canvas.value) return;

  // Get computed canvas size from CSS variable
  const computedStyle = getComputedStyle(document.documentElement);
  const canvasSize = parseInt(
    computedStyle.getPropertyValue("--canvas-size"),
    10
  );

  // Set canvas dimensions
  canvas.value.width = canvasSize;
  canvas.value.height = canvasSize;

  // Redraw if we have an image
  if (image.src) {
    drawImage();
  }

  // Recreate noise texture at new size
  noiseTexture = createNoiseTexture(canvas.value.width, canvas.value.height);
}

function triggerFileInput(e) {
  // Prevent default if it's a button click
  if (e) e.preventDefault();

  // Only trigger file input if we're not clicking in the message area
  if (e && e.target === messageInput.value) return;

  imageInput.value.click();
}

function handleImageUpload(e) {
  const file = e.target.files[0];
  if (file) {
    // Show loading state
    isLoading.value = true;

    // Draw loading placeholder
    if (ctx && canvas.value) {
      ctx.fillStyle = "#f0f0f0";
      ctx.fillRect(0, 0, canvas.value.width, canvas.value.height);

      ctx.fillStyle = "#999";
      ctx.textAlign = "center";
      ctx.font = ".12rem Inter, sans-serif";
      ctx.fillText(
        "Loading...",
        canvas.value.width / 2,
        canvas.value.height / 2
      );
    }

    const reader = new FileReader();
    reader.onload = (event) => {
      image = new Image();
      image.onload = () => {
        // Image loaded successfully
        hasImage.value = true;
        currentImageSrc.value = image.src; // Update reactive source
        isLoading.value = false;
        drawImage();
      };
      image.onerror = () => {
        // Handle image load error
        isLoading.value = false;
        alert("Failed to load image. Please try another file.");
      };
      image.src = event.target.result;
    };
    reader.readAsDataURL(file);
  }
}

// Function to create a resized version of the image for background
function createResizedBackground(imageSrc) {
  const bgImage = new Image();

  bgImage.onload = () => {
    try {
      // Calculate new dimensions while maintaining aspect ratio
      let newWidth, newHeight;

      if (bgImage.width > bgImage.height) {
        newWidth = Math.min(bgImage.width, MAX_BACKGROUND_SIZE);
        newHeight = (bgImage.height / bgImage.width) * newWidth;
      } else {
        newHeight = Math.min(bgImage.height, MAX_BACKGROUND_SIZE);
        newWidth = (bgImage.width / bgImage.height) * newHeight;
      }

      // Set canvas dimensions
      backgroundCanvas.width = newWidth;
      backgroundCanvas.height = newHeight;

      // Draw resized image to canvas
      const bgCtx = backgroundCanvas.getContext("2d");
      bgCtx.drawImage(bgImage, 0, 0, newWidth, newHeight);

      // Get data URL of resized image
      const resizedImageUrl = backgroundCanvas.toDataURL("image/jpeg", 0.7); // Use JPEG with 70% quality for smaller size

      // Detect if we're on a mobile device
      const isMobile =
        /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(
          navigator.userAgent
        );

      if (isMobile) {
        // Use the fallback element for mobile
        bgElement.style.backgroundImage = `url(${resizedImageUrl})`;
        // Clear the CSS variable to avoid duplicate backgrounds
        document.body.style.setProperty("--bg-image", "none");
      } else {
        // Use the CSS variable for desktop
        document.body.style.setProperty(
          "--bg-image",
          `url(${resizedImageUrl})`
        );
        // Clear the fallback background
        bgElement.style.backgroundImage = "none";
      }
    } catch (error) {
      console.error("Error creating background:", error);
      // Fallback to solid color background if there's an error
      document.body.style.setProperty("--bg-image", "none");
      bgElement.style.backgroundImage = "none";
      document.body.style.backgroundColor = "#f0f0f0";
    }
  };

  bgImage.onerror = () => {
    console.error("Error loading background image");
    // Fallback to solid color background
    document.body.style.setProperty("--bg-image", "none");
    bgElement.style.backgroundImage = "none";
    document.body.style.backgroundColor = "#f0f0f0";
  };

  // Load the image
  bgImage.src = imageSrc;
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
  const newHeight = Math.min(
    Math.max(
      messageInput.value.scrollHeight,
      parseInt(
        getComputedStyle(document.documentElement).getPropertyValue(
          "--message-input-height"
        ),
        10
      )
    ),
    100
  );
  messageInput.value.style.height = `${newHeight}px`;
}

// Improved download function to capture the full polaroid with proper text placement
function downloadPolaroid(e) {
  e.preventDefault();
  e.stopPropagation();

  if (!polaroid.value || !hasImage.value) return;

  // Create a temporary canvas to draw the entire polaroid
  const tempCanvas = document.createElement("canvas");
  const polaroidRect = polaroid.value.getBoundingClientRect();

  // Set canvas size to match polaroid dimensions
  tempCanvas.width = polaroidRect.width;
  tempCanvas.height = polaroidRect.height;

  const tempCtx = tempCanvas.getContext("2d");

  // Draw white background for the entire polaroid
  tempCtx.fillStyle = "white";
  tempCtx.fillRect(0, 0, tempCanvas.width, tempCanvas.height);

  // Get the position of the canvas within the polaroid
  const canvasRect = canvas.value.getBoundingClientRect();
  const canvasOffsetX = canvasRect.left - polaroidRect.left;
  const canvasOffsetY = canvasRect.top - polaroidRect.top;

  // Draw the image canvas at the correct position
  tempCtx.drawImage(
    canvas.value,
    canvasOffsetX,
    canvasOffsetY,
    canvasRect.width,
    canvasRect.height
  );

  // Add the message text at the bottom of the polaroid
  if (userMessage.value) {
    tempCtx.font = "200 1.2rem Inter, sans-serif";
    tempCtx.fillStyle = "black";
    tempCtx.textAlign = "left";

    // Position text in the bottom white area
    const textX = canvasOffsetX;
    const textY = canvasOffsetY + canvasRect.height + 40;

    // Handle text wrapping for long messages
    const maxWidth = tempCanvas.width - 40;
    const words = userMessage.value.split(" ");

    let line = "";
    let y = textY;

    for (let i = 0; i < words.length; i++) {
      const testLine = line + words[i] + " ";
      const metrics = tempCtx.measureText(testLine);

      if (metrics.width > maxWidth && i > 0) {
        tempCtx.fillText(line, textX, y);
        line = words[i] + " ";
        y += 20;
      } else {
        line = testLine;
      }
    }

    tempCtx.fillText(line, textX, y);
  }

  // Convert to data URL and trigger download
  try {
    const dataUrl = tempCanvas.toDataURL("image/png");
    const link = document.createElement("a");
    link.download = "my-polaroid.png";
    link.href = dataUrl;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  } catch (err) {
    console.error("Error creating download:", err);
    alert("Could not download the image. Please try again.");
  }
}
</script>

<style>
:root {
  /* CSS Variables for consistent sizing */
  --polaroid-width: 500px;
  --polaroid-height: 600px;
  --polaroid-padding: 20px;
  --canvas-size: calc(var(--polaroid-width) - (var(--polaroid-padding) * 2));
  --message-input-height: 40px;
  --download-button-height: 44px;
  --download-button-width: 180px;
  --bg-image: none;
}

body {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
  margin: 0;
  background-color: #f0f0f0;
  position: relative;
  overflow-x: hidden;
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
  filter: grayscale(100%) blur(0.5rem);
  z-index: -1;
  transition: background-image 0.3s;
  will-change: background-image;
}

/* Fallback background for mobile compatibility */
.background-fallback {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  filter: grayscale(100%) blur(0.5rem);
  z-index: -1; /* Same z-index as body::before */
  transition: background-image 0.3s;
  pointer-events: none;
}

.container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  width: 100%;
  max-width: 100%;
  padding: 20px;
  box-sizing: border-box;
  gap: 20px;
  position: relative;
  z-index: 1;
}

.polaroid-wrapper {
  display: flex;
  justify-content: center;
  width: 100%;
}

.polaroid {
  background: white;
  padding: var(--polaroid-padding) var(--polaroid-padding) 80px
    var(--polaroid-padding);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  width: var(--polaroid-width);
  max-width: 100%;
  position: relative;
  text-align: center;
  filter: none;
  z-index: 1;
  cursor: pointer;
  will-change: transform;
  contain: layout style;
  box-sizing: border-box;
  margin: 0 auto;
}

.image-container {
  width: 100%;
  aspect-ratio: 1 / 1;
  position: relative;
  margin: 0 auto;
  contain: layout size style;
}

.canvas {
  display: block;
  width: 100%;
  height: 100%;
  contain: strict;
}

/* Skeleton loader */
.skeleton-loader {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
  background-size: 200% 100%;
  animation: loading 1.5s infinite;
  z-index: 2;
}

@keyframes loading {
  0% {
    background-position: 200% 0;
  }
  100% {
    background-position: -200% 0;
  }
}

.polaroid-text {
  font-family: "Inter", sans-serif;
  font-weight: 200;
  font-size: 1.8rem;
  text-align: left;
  text-transform: uppercase;
  letter-spacing: -0.3px;
  position: absolute;
  top: -42px; /* Position above Polaroid */
  left: 0;
  width: 100%;
  margin: 0;
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
  contain: content;
}

.file-input-container {
  position: absolute;
  top: var(--polaroid-padding);
  left: var(--polaroid-padding);
  right: var(--polaroid-padding);
  bottom: 100px;
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 2;
  contain: layout style;
}

.file-input {
  opacity: 0;
  position: absolute;
  width: 100%;
  height: 100%;
  cursor: pointer;
}

.file-input-prompt {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background-color: rgba(245, 245, 245, 0.7);
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
  transition: all 0.2s ease;
  contain: content;
}

.file-input-prompt:hover {
  background-color: rgba(245, 245, 245, 0.9);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15);
}

.photo-icon {
  font-size: 2rem;
  margin-bottom: 10px;
}

.prompt-text {
  font-size: 1.2rem;
  letter-spacing: -0.3px;
  color: #555;
}

.visually-hidden {
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.3s ease;
}

.message-container {
  position: absolute;
  bottom: 0px;
  left: var(--polaroid-padding);
  right: var(--polaroid-padding);
  z-index: 3;
  height: 60px; /* Fixed height to prevent layout shift */
  contain: layout size style;
}

.message-input {
  width: 100%;
  height: var(--message-input-height);
  min-height: var(--message-input-height);
  max-height: 100px;
  border: none;
  background: transparent;
  font-family: "Inter", sans-serif;
  font-size: 1.2rem;
  letter-spacing: -0.3px;
  line-height: 1.5;
  resize: none;
  outline: none;
  text-align: left;
  overflow-y: auto;
  box-sizing: border-box;
}

.message-input::placeholder {
  color: #aaa;
}

/* Download button below the polaroid */
.download-button {
  background-color: #f0f8ff;
  border: 1px solid #ddd;
  border-radius: 25px;
  padding: 10px 20px;
  font-size: 1.2rem;
  letter-spacing: -0.3px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.2s ease;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  margin-top: 10px;
  height: var(--download-button-height);
  min-width: var(--download-button-width);
  contain: layout size style;
}

.download-button:hover:not(.disabled) {
  background-color: #e6f2ff;
  box-shadow: 0 3px 6px rgba(0, 0, 0, 0.15);
}

.download-button.disabled {
  opacity: 0.5;
  cursor: not-allowed;
  background-color: #f5f5f5;
}

.button-icon {
  margin-right: 8px;
  font-size: 1.2rem;
}

/* Responsive adjustments */
@media (max-width: 600px) {
  :root {
    --polaroid-width: calc(100vw - 40px);
  }
}
</style>
