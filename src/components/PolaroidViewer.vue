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
        <div class="polaroid-text">Create your sunshine photo</div>

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

    <!-- Filter controls - always present but conditionally visible -->
    <div class="filter-controls">
      <!-- Top row for all buttons -->
      <div
        class="filter-controls-top-row"
        :style="{
          opacity: hasImage ? 1 : 0,
          visibility: hasImage ? 'visible' : 'hidden',
        }"
      >
        <!-- Permission button for device orientation -->
        <button
          v-show="!orientationPermissionGranted && needsPermissionRequest"
          @click="requestOrientationPermission"
          class="action-button"
        >
          <span class="button-icon">✨</span>
          Enable Effects
        </button>

        <!-- Filter toggle button -->
        <button
          @click="toggleFilters"
          class="action-button"
          :class="{ active: filtersEnabled }"
        >
          <span class="button-icon">🎨</span>
          {{ filtersEnabled ? "Disable Filters" : "Enable Filters" }}
        </button>

        <!-- Download button -->
        <button
          class="action-button"
          @click.stop="downloadPolaroid"
          :disabled="!hasImage"
          :class="{ disabled: !hasImage }"
        >
          <span class="button-icon">💾</span>
          Download
        </button>
      </div>

      <!-- Horizontal scrollable filter selector -->
      <div
        class="filter-selector"
        :style="{
          opacity: hasImage && filtersEnabled ? 1 : 0,
          visibility: hasImage && filtersEnabled ? 'visible' : 'hidden',
        }"
      >
        <div class="filter-options-container">
          <div class="filter-options">
            <button
              v-for="filter in availableFilters"
              :key="filter.id"
              @click.stop="selectFilter(filter.id)"
              :class="{ active: currentFilter === filter.id }"
              class="filter-button"
            >
              {{ filter.name }}
            </button>
          </div>
        </div>

        <!-- Filter intensity slider container - always present but conditionally visible -->
        <div class="filter-intensity-container">
          <div
            :style="{
              opacity: showFilterIntensity ? 1 : 0,
              visibility: showFilterIntensity ? 'visible' : 'hidden',
            }"
            class="filter-intensity"
          >
            <label for="intensity-slider">Intensity:</label>
            <input
              type="range"
              id="intensity-slider"
              min="1"
              max="100"
              v-model="filterIntensity"
              @input="handleIntensityChange"
            />
          </div>
        </div>
      </div>
    </div>
  </div>

  <!-- Add this to ensure the Inter font is loaded -->
  <link
    href="https://fonts.googleapis.com/css2?family=Inter:wght@100;400&display=swap"
    rel="stylesheet"
  />
</template>

<script setup>
import { ref, onMounted, onUnmounted, watch, nextTick, computed } from "vue";

// Reactive state
const polaroid = ref(null);
const canvas = ref(null);
const imageInput = ref(null);
const messageInput = ref(null);
const hasImage = ref(false);
const isLoading = ref(false);
const userMessage = ref("");
const currentImageSrc = ref("");
const orientationPermissionGranted = ref(false);
const needsPermissionRequest = ref(false);
const currentFilter = ref("none"); // Default filter is none
const filterIntensity = ref(50); // Default intensity (1-100)
const filtersEnabled = ref(true); // New state for filter toggle
let backgroundCanvas = null;
let originalImageData = null; // Store original image data for filters
let processingCanvas = null; // For complex filter processing

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
const FIXED_DOWNLOAD_WIDTH = 500; // Fixed width for downloaded images
const MIN_POLAROID_WIDTH = 320; // Minimum width for polaroid
const REFERENCE_WIDTH = 500; // Reference width for scaling

// Available filters
const availableFilters = [
  { id: "none", name: "Normal" },
  { id: "grayscale", name: "B&W" },
  { id: "sepia", name: "Sepia" },
  { id: "vintage", name: "Vintage" },
  { id: "highContrast", name: "High Contrast" },
  { id: "blur", name: "Blur" },
  { id: "invert", name: "Invert" },
  { id: "duotone", name: "Duotone" },
  // Advanced filters
  { id: "oilPaint", name: "Oil Paint" },
  { id: "pixelate", name: "Pixelate" },
  { id: "glitch", name: "Glitch" },
  { id: "watercolor", name: "Watercolor" },
  { id: "emboss", name: "Emboss" },
  { id: "posterize", name: "Posterize" },
  { id: "halftone", name: "Halftone" },
];

// Filters that support intensity adjustment
const intensityFilters = [
  "blur",
  "pixelate",
  "oilPaint",
  "glitch",
  "watercolor",
  "emboss",
  "posterize",
  "halftone",
];

// Computed property to determine if we should show the intensity slider
const showFilterIntensity = computed(() => {
  return filtersEnabled.value && intensityFilters.includes(currentFilter.value);
});

// Toggle filters on/off
function toggleFilters() {
  filtersEnabled.value = !filtersEnabled.value;

  // If disabling filters, set to normal
  if (!filtersEnabled.value) {
    const previousFilter = currentFilter.value;
    currentFilter.value = "none";

    // Redraw the image without filters
    if (hasImage.value && image.src) {
      drawImage();
    }

    // Store the previous filter to restore when re-enabling
    currentFilter.value = previousFilter;
  } else {
    // Redraw with the current filter when re-enabling
    if (hasImage.value && image.src) {
      drawImage();
    }
  }
}

// Initialize canvas and event listeners
onMounted(() => {
  // Create canvas context with willReadFrequently attribute to fix performance warning
  ctx = canvas.value.getContext("2d", { willReadFrequently: true });

  // Set initial dimensions before any content loads
  setInitialDimensions();

  // Initialize canvas with proper dimensions
  resizeCanvas();

  // Create processing canvas for complex filters
  processingCanvas = document.createElement("canvas");

  // Set up event listeners
  window.addEventListener("resize", handleResize);
  window.addEventListener("deviceorientation", handleDeviceOrientation);

  // Check if device orientation permission is needed
  if (typeof DeviceOrientationEvent.requestPermission === "function") {
    needsPermissionRequest.value = true;
  } else {
    // No permission needed for this device/browser
    orientationPermissionGranted.value = true;
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

  // Watch for filter changes and redraw the image
  watch(currentFilter, () => {
    if (hasImage.value && image.src) {
      drawImage();
    }
  });

  // Initialize horizontal scrolling for filter options
  nextTick(() => {
    initializeHorizontalScroll();
  });
});

// Initialize horizontal scrolling for filter options
function initializeHorizontalScroll() {
  const filterOptions = document.querySelector(".filter-options");
  if (filterOptions) {
    // Enable mouse wheel horizontal scrolling
    filterOptions.addEventListener("wheel", (e) => {
      if (e.deltaY !== 0) {
        e.preventDefault();
        filterOptions.scrollLeft += e.deltaY;
      }
    });
  }
}

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

// Handle intensity change
function handleIntensityChange() {
  if (hasImage.value && image.src) {
    drawImage();
  }
}

// Request device orientation permission
function requestOrientationPermission() {
  if (typeof DeviceOrientationEvent.requestPermission === "function") {
    DeviceOrientationEvent.requestPermission()
      .then((permissionState) => {
        if (permissionState === "granted") {
          console.log("Device orientation permission granted");
          orientationPermissionGranted.value = true;
        } else {
          console.log("Device orientation permission denied");
        }
      })
      .catch(console.error);
  } else {
    // No permission needed for this device/browser
    orientationPermissionGranted.value = true;
  }
}

// Select a filter
function selectFilter(filterId) {
  currentFilter.value = filterId;

  // Reset intensity to default for new filter
  if (intensityFilters.includes(filterId)) {
    filterIntensity.value = 50;
  }
}

// Functions
function handleResize() {
  setInitialDimensions();
  resizeCanvas();

  // Re-initialize horizontal scrolling after resize
  nextTick(() => {
    initializeHorizontalScroll();
  });
}

function setInitialDimensions() {
  // Calculate and set dimensions based on viewport with minimum width
  const viewportWidth = Math.max(
    Math.min(window.innerWidth - 40, REFERENCE_WIDTH),
    MIN_POLAROID_WIDTH
  );
  document.documentElement.style.setProperty(
    "--polaroid-width",
    `${viewportWidth}px`
  );

  // Calculate the scale ratio compared to reference width
  const scaleRatio = viewportWidth / REFERENCE_WIDTH;
  document.documentElement.style.setProperty("--scale-ratio", scaleRatio);

  // Calculate canvas size based on polaroid width
  // Use the same proportions as when width is 500px
  const canvasSize = viewportWidth - 2 * (20 * scaleRatio); // 20px padding on each side at reference width
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

  // Set border size based on reference width
  const borderSize = 20 * scaleRatio;
  document.documentElement.style.setProperty(
    "--polaroid-padding",
    `${borderSize}px`
  );

  // Set bottom padding based on reference width
  const bottomPadding = 80 * scaleRatio;
  document.documentElement.style.setProperty(
    "--polaroid-bottom-padding",
    `${bottomPadding}px`
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

      // Draw resized image to canvas - use willReadFrequently for better performance
      const bgCtx = backgroundCanvas.getContext("2d", {
        willReadFrequently: true,
      });
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
  const noiseCtx = noiseCanvas.getContext("2d", { willReadFrequently: true });
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

// Apply filter to the canvas
function applyFilter(context, filterId, width, height) {
  // Skip filter application if filters are disabled
  if (!filtersEnabled.value || filterId === "none") return;

  // Create a temporary canvas to apply filters
  const tempCanvas = document.createElement("canvas");
  tempCanvas.width = width;
  tempCanvas.height = height;
  const tempCtx = tempCanvas.getContext("2d", { willReadFrequently: true });

  // Copy the current canvas content to the temp canvas
  tempCtx.drawImage(context.canvas, 0, 0);

  // Get image data for pixel manipulation
  const imageData = tempCtx.getImageData(0, 0, width, height);
  const data = imageData.data;

  // Get intensity value (1-100)
  const intensity = parseInt(filterIntensity.value, 10);

  switch (filterId) {
    case "grayscale":
      // Black and White filter
      for (let i = 0; i < data.length; i += 4) {
        const avg = (data[i] + data[i + 1] + data[i + 2]) / 3;
        data[i] = avg; // red
        data[i + 1] = avg; // green
        data[i + 2] = avg; // blue
      }
      break;

    case "sepia":
      // Sepia filter
      for (let i = 0; i < data.length; i += 4) {
        const r = data[i];
        const g = data[i + 1];
        const b = data[i + 2];

        data[i] = Math.min(255, r * 0.393 + g * 0.769 + b * 0.189);
        data[i + 1] = Math.min(255, r * 0.349 + g * 0.686 + b * 0.168);
        data[i + 2] = Math.min(255, r * 0.272 + g * 0.534 + b * 0.131);
      }
      break;

    case "vintage":
      // Vintage filter (sepia + reduced saturation + vignette)
      for (let i = 0; i < data.length; i += 4) {
        const r = data[i];
        const g = data[i + 1];
        const b = data[i + 2];

        // Sepia-like effect
        data[i] = Math.min(255, r * 0.393 + g * 0.769 + b * 0.189);
        data[i + 1] = Math.min(255, r * 0.349 + g * 0.686 + b * 0.168);
        data[i + 2] = Math.min(255, r * 0.272 + g * 0.534 + b * 0.131);

        // Reduce saturation
        const avg = (data[i] + data[i + 1] + data[i + 2]) / 3;
        data[i] = data[i] * 0.9 + avg * 0.1;
        data[i + 1] = data[i + 1] * 0.9 + avg * 0.1;
        data[i + 2] = data[i + 2] * 0.9 + avg * 0.1;
      }

      // Apply vignette after putting image data back
      tempCtx.putImageData(imageData, 0, 0);

      // Create vignette effect
      const gradient = tempCtx.createRadialGradient(
        width / 2,
        height / 2,
        0,
        width / 2,
        height / 2,
        width * 0.7
      );
      gradient.addColorStop(0, "rgba(0,0,0,0)");
      gradient.addColorStop(1, "rgba(0,0,0,0.3)");

      tempCtx.fillStyle = gradient;
      tempCtx.globalCompositeOperation = "multiply";
      tempCtx.fillRect(0, 0, width, height);
      tempCtx.globalCompositeOperation = "source-over";

      // Draw the result back to the original context
      context.drawImage(tempCanvas, 0, 0);
      return; // Skip the imageData.putData below since we've already drawn

    case "highContrast":
      // High contrast filter
      for (let i = 0; i < data.length; i += 4) {
        data[i] = data[i] < 127 ? data[i] * 0.8 : data[i] * 1.2;
        data[i + 1] = data[i + 1] < 127 ? data[i + 1] * 0.8 : data[i + 1] * 1.2;
        data[i + 2] = data[i + 2] < 127 ? data[i + 2] * 0.8 : data[i + 2] * 1.2;
      }
      break;

    case "blur":
      // Blur with adjustable intensity
      const blurAmount = Math.max(1, Math.floor(intensity / 10)); // 1-10px blur
      tempCtx.putImageData(imageData, 0, 0);
      tempCtx.filter = `blur(${blurAmount}px)`;
      tempCtx.drawImage(tempCanvas, 0, 0);
      context.drawImage(tempCanvas, 0, 0);
      return; // Skip the imageData.putData below

    case "invert":
      // Invert colors
      for (let i = 0; i < data.length; i += 4) {
        data[i] = 255 - data[i];
        data[i + 1] = 255 - data[i + 1];
        data[i + 2] = 255 - data[i + 2];
      }
      break;

    case "duotone":
      // Duotone (blue and pink)
      for (let i = 0; i < data.length; i += 4) {
        const avg = (data[i] + data[i + 1] + data[i + 2]) / 3;
        if (avg < 128) {
          // Shadows to blue
          data[i] = avg * 0.4;
          data[i + 1] = avg * 0.4;
          data[i + 2] = avg * 1.2;
        } else {
          // Highlights to pink
          data[i] = avg * 1.2;
          data[i + 1] = avg * 0.4;
          data[i + 2] = avg * 1.0;
        }
      }
      break;

    case "pixelate":
      // Pixelate effect with adjustable block size
      const blockSize = Math.max(2, Math.floor(intensity / 5)); // 2-20px blocks

      // Put the original image data back
      tempCtx.putImageData(imageData, 0, 0);

      // Create pixelation by drawing the image at a smaller size and scaling back up
      const smallCanvas = document.createElement("canvas");
      const smallCtx = smallCanvas.getContext("2d", {
        willReadFrequently: true,
      });

      // Calculate the scaled-down dimensions
      const scaledWidth = Math.ceil(width / blockSize);
      const scaledHeight = Math.ceil(height / blockSize);

      smallCanvas.width = scaledWidth;
      smallCanvas.height = scaledHeight;

      // Draw the image at a smaller size
      smallCtx.drawImage(tempCanvas, 0, 0, scaledWidth, scaledHeight);

      // Clear the temp canvas
      tempCtx.clearRect(0, 0, width, height);

      // Draw the small image back to the temp canvas, scaled up to create pixelation
      tempCtx.imageSmoothingEnabled = false; // Disable smoothing for pixelated look
      tempCtx.drawImage(smallCanvas, 0, 0, width, height);

      // Draw the result back to the original context
      context.drawImage(tempCanvas, 0, 0);
      return;

    case "oilPaint":
      // Oil paint effect with adjustable intensity
      const radius = Math.max(1, Math.floor(intensity / 10)); // 1-10px radius
      const numLevels = 16; // Number of color levels

      // Create a new array to store the oil paint data
      const oilData = new Uint8ClampedArray(data.length);

      // For each pixel in the image
      for (let y = 0; y < height; y++) {
        for (let x = 0; x < width; x++) {
          // Initialize arrays to store intensity counts and color sums
          const intensityCounts = new Array(numLevels).fill(0);
          const redSums = new Array(numLevels).fill(0);
          const greenSums = new Array(numLevels).fill(0);
          const blueSums = new Array(numLevels).fill(0);

          // Sample the surrounding pixels
          for (let dy = -radius; dy <= radius; dy++) {
            for (let dx = -radius; dx <= radius; dx++) {
              const nx = Math.min(Math.max(x + dx, 0), width - 1);
              const ny = Math.min(Math.max(y + dy, 0), height - 1);

              const i = (ny * width + nx) * 4;

              // Calculate intensity level
              const r = data[i];
              const g = data[i + 1];
              const b = data[i + 2];
              const intensity = Math.floor(
                (((r + g + b) / 3) * numLevels) / 255
              );

              // Accumulate colors at this intensity
              intensityCounts[intensity]++;
              redSums[intensity] += r;
              greenSums[intensity] += g;
              blueSums[intensity] += b;
            }
          }

          // Find the most common intensity
          let maxCount = 0;
          let maxIndex = 0;
          for (let i = 0; i < numLevels; i++) {
            if (intensityCounts[i] > maxCount) {
              maxCount = intensityCounts[i];
              maxIndex = i;
            }
          }

          // Calculate the average color at the most common intensity
          const outputIndex = (y * width + x) * 4;
          if (maxCount > 0) {
            oilData[outputIndex] = Math.floor(redSums[maxIndex] / maxCount);
            oilData[outputIndex + 1] = Math.floor(
              greenSums[maxIndex] / maxCount
            );
            oilData[outputIndex + 2] = Math.floor(
              blueSums[maxIndex] / maxCount
            );
            oilData[outputIndex + 3] = 255; // Alpha
          } else {
            // Fallback if no samples were found (shouldn't happen)
            oilData[outputIndex] = data[(y * width + x) * 4];
            oilData[outputIndex + 1] = data[(y * width + x) * 4 + 1];
            oilData[outputIndex + 2] = data[(y * width + x) * 4 + 2];
            oilData[outputIndex + 3] = 255;
          }
        }
      }

      // Create a new ImageData object with the oil paint data
      const oilImageData = new ImageData(oilData, width, height);

      // Put the oil paint data back to the temp canvas
      tempCtx.putImageData(oilImageData, 0, 0);

      // Draw the result back to the original context
      context.drawImage(tempCanvas, 0, 0);
      return;

    case "glitch":
      // Glitch effect with adjustable intensity
      const glitchAmount = intensity / 100; // 0-1 range

      // Put the original image data back
      tempCtx.putImageData(imageData, 0, 0);

      // Create RGB shift
      tempCtx.globalCompositeOperation = "lighten";

      // Red channel shift
      tempCtx.fillStyle = "rgba(255,0,0,0.5)";
      tempCtx.fillRect(
        -glitchAmount * 10,
        0,
        width + glitchAmount * 20,
        height
      );

      // Blue channel shift
      tempCtx.fillStyle = "rgba(0,0,255,0.5)";
      tempCtx.fillRect(glitchAmount * 10, 0, width - glitchAmount * 20, height);

      // Create random glitch blocks
      tempCtx.globalCompositeOperation = "source-over";

      const numGlitches = Math.floor(glitchAmount * 10);
      for (let i = 0; i < numGlitches; i++) {
        // Random position and size
        const glitchX = Math.random() * width;
        const glitchY = Math.random() * height;
        const glitchW = Math.random() * 100 * glitchAmount + 20;
        const glitchH = Math.random() * 50 * glitchAmount + 10;

        // Random source position
        const srcX = Math.random() * width;
        const srcY = Math.random() * height;

        // Copy a random block to a random position
        tempCtx.drawImage(
          tempCanvas,
          srcX,
          srcY,
          glitchW,
          glitchH,
          glitchX,
          glitchY,
          glitchW,
          glitchH
        );
      }

      // Add some noise
      const noiseOpacity = glitchAmount * 0.2;
      for (let i = 0; i < (width * height) / 50; i++) {
        const noiseX = Math.random() * width;
        const noiseY = Math.random() * height;
        const noiseW = Math.random() * 20 + 1;
        const noiseH = Math.random() * 2 + 1;

        tempCtx.fillStyle = `rgba(${Math.random() * 255},${
          Math.random() * 255
        },${Math.random() * 255},${noiseOpacity})`;
        tempCtx.fillRect(noiseX, noiseY, noiseW, noiseH);
      }

      // Draw the result back to the original context
      context.drawImage(tempCanvas, 0, 0);
      return;

    case "watercolor":
      // Watercolor effect with adjustable intensity
      const blurRadius = Math.max(1, Math.floor(intensity / 20)); // 1-5px blur
      const edgeStrength = intensity / 100; // 0-1 range

      // First apply a slight blur
      tempCtx.putImageData(imageData, 0, 0);
      tempCtx.filter = `blur(${blurRadius}px)`;
      tempCtx.drawImage(tempCanvas, 0, 0);

      // Get the blurred image data
      const blurredData = tempCtx.getImageData(0, 0, width, height).data;

      // Create a new array for the watercolor effect
      const watercolorData = new Uint8ClampedArray(data.length);

      // Apply edge detection and color bleeding
      for (let y = 1; y < height - 1; y++) {
        for (let x = 1; x < width - 1; x++) {
          const i = (y * width + x) * 4;

          // Get surrounding pixels
          const topIdx = ((y - 1) * width + x) * 4;
          const bottomIdx = ((y + 1) * width + x) * 4;
          const leftIdx = (y * width + (x - 1)) * 4;
          const rightIdx = (y * width + (x + 1)) * 4;

          // Calculate edge strength (simple Sobel-like)
          const edgeX =
            (blurredData[rightIdx] -
              blurredData[leftIdx] +
              blurredData[rightIdx + 1] -
              blurredData[leftIdx + 1] +
              blurredData[rightIdx + 2] -
              blurredData[leftIdx + 2]) /
            3;

          const edgeY =
            (blurredData[bottomIdx] -
              blurredData[topIdx] +
              blurredData[bottomIdx + 1] -
              blurredData[topIdx + 1] +
              blurredData[bottomIdx + 2] -
              blurredData[topIdx + 2]) /
            3;

          const edgeMagnitude =
            Math.sqrt(edgeX * edgeX + edgeY * edgeY) * edgeStrength;

          // Apply watercolor effect - enhance edges and blend colors
          watercolorData[i] = Math.min(255, blurredData[i] + edgeMagnitude);
          watercolorData[i + 1] = Math.min(
            255,
            blurredData[i + 1] + edgeMagnitude
          );
          watercolorData[i + 2] = Math.min(
            255,
            blurredData[i + 2] + edgeMagnitude
          );
          watercolorData[i + 3] = 255;
        }
      }

      // Create a new ImageData object with the watercolor data
      const watercolorImageData = new ImageData(watercolorData, width, height);

      // Put the watercolor data back to the temp canvas
      tempCtx.putImageData(watercolorImageData, 0, 0);

      // Add a paper texture effect
      tempCtx.globalCompositeOperation = "multiply";
      tempCtx.fillStyle = "rgba(240, 240, 230, 0.5)";
      tempCtx.fillRect(0, 0, width, height);

      // Draw the result back to the original context
      context.drawImage(tempCanvas, 0, 0);
      return;

    case "emboss":
      // Emboss effect with adjustable intensity
      const embossStrength = intensity / 50; // 0-2 range

      // Create a new array for the emboss effect
      const embossData = new Uint8ClampedArray(data.length);

      // Apply emboss filter
      for (let y = 1; y < height - 1; y++) {
        for (let x = 1; x < width - 1; x++) {
          const i = (y * width + x) * 4;
          const topLeftIdx = ((y - 1) * width + (x - 1)) * 4;
          const bottomRightIdx = ((y + 1) * width + (x + 1)) * 4;

          // Calculate emboss value for each channel
          for (let c = 0; c < 3; c++) {
            const embossValue =
              (data[topLeftIdx + c] - data[bottomRightIdx + c]) *
                embossStrength +
              128;
            embossData[i + c] = Math.min(255, Math.max(0, embossValue));
          }

          embossData[i + 3] = 255; // Alpha
        }
      }

      // Create a new ImageData object with the emboss data
      const embossImageData = new ImageData(embossData, width, height);

      // Put the emboss data back to the temp canvas
      tempCtx.putImageData(embossImageData, 0, 0);

      // Draw the result back to the original context
      context.drawImage(tempCanvas, 0, 0);
      return;

    case "posterize":
      // Posterize effect with adjustable levels
      const posterizeLevels = Math.max(2, Math.floor(intensity / 10)); // 2-10 levels
      const step = 255 / (posterizeLevels - 1);

      for (let i = 0; i < data.length; i += 4) {
        // Posterize each channel
        data[i] = Math.round(data[i] / step) * step;
        data[i + 1] = Math.round(data[i + 1] / step) * step;
        data[i + 2] = Math.round(data[i + 2] / step) * step;
      }
      break;

    case "halftone":
      // Halftone effect with adjustable dot size
      const dotSize = Math.max(3, Math.floor(intensity / 5)); // 3-20px dots
      const spacing = dotSize;

      // Put the original image data back
      tempCtx.putImageData(imageData, 0, 0);

      // Create a new canvas for the halftone effect
      const dotCanvas = document.createElement("canvas");
      dotCanvas.width = width;
      dotCanvas.height = height;
      const dotCtx = dotCanvas.getContext("2d", { willReadFrequently: true });

      // Fill with white background
      dotCtx.fillStyle = "white";
      dotCtx.fillRect(0, 0, width, height);

      // Draw dots
      dotCtx.fillStyle = "black";

      for (let y = spacing / 2; y < height; y += spacing) {
        for (let x = spacing / 2; x < width; x += spacing) {
          // Get the color at this position
          const i = (Math.floor(y) * width + Math.floor(x)) * 4;
          const r = data[i];
          const g = data[i + 1];
          const b = data[i + 2];

          // Calculate brightness (0-1)
          const brightness = (r + g + b) / (3 * 255);

          // Calculate dot radius based on brightness
          const radius = ((1 - brightness) * dotSize) / 2;

          // Draw the dot
          dotCtx.beginPath();
          dotCtx.arc(x, y, radius, 0, Math.PI * 2);
          dotCtx.fill();
        }
      }

      // Draw the halftone canvas back to the temp canvas
      tempCtx.clearRect(0, 0, width, height);
      tempCtx.drawImage(dotCanvas, 0, 0);

      // Draw the result back to the original context
      context.drawImage(tempCanvas, 0, 0);
      return;
  }

  // Put the modified image data back to the temp canvas
  tempCtx.putImageData(imageData, 0, 0);

  // Draw the result back to the original context
  context.drawImage(tempCanvas, 0, 0);
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

  // Draw the original image
  ctx.drawImage(image, offsetX, offsetY, drawWidth, drawHeight);

  // Apply selected filter if filters are enabled
  if (filtersEnabled.value) {
    applyFilter(ctx, currentFilter.value, canvasWidth, canvasHeight);
  }

  // Apply special effects after the filter
  applySunCatcherEffect();
}

function drawHolographicSheen() {
  if (!ctx || !canvas.value) return;

  ctx.save();
  const tiltIntensity = Math.min(Math.abs(tiltX) + Math.abs(tiltY), 1);
  ctx.globalAlpha = 0.3 * tiltIntensity;
  const gradient = ctx.createLinearGradient(
    canvas.value.width * (0.5 + tiltX),
    canvas.value.height * (0.5 + tiltY),
    canvas.value.width * (0.5 - tiltX),
    canvas.value.height * (0.5 - tiltY)
  );
  gradient.addColorStop(0, "rgba(0, 255, 255, 0)");
  gradient.addColorStop(0.5, "rgba(255, 0, 255, 0.5)");
  gradient.addColorStop(1, "rgba(255, 255, 0, 0)");
  ctx.fillStyle = gradient;
  ctx.fillRect(0, 0, canvas.value.width, canvas.value.height);
  ctx.globalAlpha = 1;
  ctx.restore();
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

  // Holographic Sheen
  drawHolographicSheen();

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
  // Removed polaroid tilting as requested

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

// Improved download function with fixed width of 500px
function downloadPolaroid(e) {
  e.preventDefault();
  e.stopPropagation();

  if (!polaroid.value || !hasImage.value || !canvas.value) return;

  // Create a temporary canvas with fixed width of 500px
  const tempCanvas = document.createElement("canvas");
  const fixedWidth = FIXED_DOWNLOAD_WIDTH;

  // Fixed proportions based on the reference design (500px width)
  const fixedPaddingTop = 20; // 20px top padding at 500px width
  const fixedPaddingSide = 20; // 20px side padding at 500px width
  const fixedBottomPadding = 80; // 80px bottom padding at 500px width

  // Calculate the image area dimensions
  const imageWidth = fixedWidth - fixedPaddingSide * 2;
  const imageHeight = imageWidth; // Square image

  // Calculate total height of the polaroid
  const totalHeight = imageHeight + fixedPaddingTop + fixedBottomPadding;

  // Set canvas dimensions with fixed proportions
  tempCanvas.width = fixedWidth;
  tempCanvas.height = totalHeight;

  const tempCtx = tempCanvas.getContext("2d", { willReadFrequently: true });

  // Draw white background for the entire polaroid
  tempCtx.fillStyle = "white";
  tempCtx.fillRect(0, 0, tempCanvas.width, tempCanvas.height);

  // Draw directly from the original canvas which already has all filters and effects applied
  tempCtx.drawImage(
    canvas.value,
    fixedPaddingSide,
    fixedPaddingTop,
    imageWidth,
    imageHeight
  );

  // Add the message text at the bottom of the polaroid
  if (userMessage.value) {
    // Use consistent font size based on reference width
    const fontSize = 19.2; // Fixed font size for 500px width

    tempCtx.font = `200 ${fontSize}px Inter, sans-serif`;
    tempCtx.fillStyle = "black";
    tempCtx.textAlign = "left";

    // Position text in the bottom white area - always at the same position
    const textX = fixedPaddingSide;
    const textY = fixedPaddingTop + imageHeight + 44; // Fixed position from the bottom of the image

    // Handle text wrapping for long messages
    const maxWidth = tempCanvas.width - fixedPaddingSide * 2;
    const words = userMessage.value.split(" ");

    let line = "";
    let y = textY;
    const lineHeight = fontSize * 1.2;

    for (let i = 0; i < words.length; i++) {
      const testLine = line + words[i] + " ";
      const metrics = tempCtx.measureText(testLine);

      if (metrics.width > maxWidth && i > 0) {
        tempCtx.fillText(line, textX, y);
        line = words[i] + " ";
        y += lineHeight;
      } else {
        line = testLine;
      }
    }

    tempCtx.fillText(line, textX, y);
  }

  // Convert to data URL and trigger download
  try {
    // Ensure the Inter font is loaded before generating the image
    document.fonts.ready.then(() => {
      const dataUrl = tempCanvas.toDataURL("image/png");
      const link = document.createElement("a");
      link.download = "my-polaroid.png";
      link.href = dataUrl;
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);
    });
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
  --polaroid-bottom-padding: 80px;
  --canvas-size: calc(var(--polaroid-width) - (var(--polaroid-padding) * 2));
  --message-input-height: 40px;
  --download-button-height: 44px;
  --download-button-width: 180px;
  --bg-image: url("/bg.jpeg");
  --scale-ratio: 1;
}

body {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100dvh;
  margin: 0;
  background-color: #000000;
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
  filter: grayscale(100%) blur(0.2rem) brightness(50%);
  z-index: -1;
  transition: background-image 0.3s;
  will-change: background-image;
  transform: scale(1.1);
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
  filter: grayscale(100%) blur(0.2rem) brightness(50%);
  z-index: -1; /* Same z-index as body::before */
  transition: background-image 0.3s;
  pointer-events: none;
  transform: scale(1.1);
}

.container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  width: 100%;
  max-width: 100vw;
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
  padding: var(--polaroid-padding) var(--polaroid-padding)
    var(--polaroid-bottom-padding) var(--polaroid-padding);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  width: var(--polaroid-width);
  min-width: 320px;
  max-width: 100%;
  position: relative;
  text-align: center;
  filter: none;
  z-index: 1;
  cursor: pointer;
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
  font-size: clamp(1.2rem, calc(1.6rem * var(--scale-ratio)), 1.8rem);
  text-align: left;
  text-transform: uppercase;
  letter-spacing: -0.3px;
  position: absolute;
  top: calc(-42px * var(--scale-ratio)); /* Position above Polaroid */
  left: 0;
  width: 100%;
  margin: 0;
  background: linear-gradient(to right, red, orange, yellow, green, violet);
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
  bottom: var(--polaroid-bottom-padding);
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
  font-size: clamp(1.5rem, calc(2rem * var(--scale-ratio)), 2.5rem);
  margin-bottom: 10px;
}

.prompt-text {
  font-size: clamp(0.9rem, calc(1.2rem * var(--scale-ratio)), 1.4rem);
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
  height: calc(
    var(--polaroid-bottom-padding) - 20px
  ); /* Fixed height to prevent layout shift */
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
  font-size: clamp(0.9rem, calc(1.2rem * var(--scale-ratio)), 1.4rem);
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

/* Filter controls */
.filter-controls {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-top: 10px;
  margin-bottom: 10px;
  width: 100%;
  max-width: var(--polaroid-width);
  min-height: 134px; /* Increased to account for all possible content */
}

/* Top row for all buttons */
.filter-controls-top-row {
  display: flex;
  justify-content: center;
  gap: 10px;
  margin-bottom: 10px;
  width: 100%;
  min-height: 44px; /* Minimum height to prevent layout shift */
  transition: opacity 0.3s ease, visibility 0.3s ease;
}

/* Common button style for all action buttons */
.action-button {
  background-color: rgba(0, 0, 0, 0.7);
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 25px;
  padding: 8px 16px;
  font-family: "Inter", sans-serif;
  font-size: clamp(0.9rem, calc(1rem * var(--scale-ratio)), 1.2rem);
  color: white;
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
}

.action-button:hover:not(.disabled) {
  background-color: rgba(0, 0, 0, 0.8);
  border-color: rgba(255, 255, 255, 0.5);
}

.action-button.active {
  background-color: rgba(0, 0, 0, 0.9);
  border-color: rgba(255, 255, 255, 0.7);
}

.action-button.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Filter selector with horizontal scrolling */
.filter-selector {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  min-height: 80px; /* Height for filter options + intensity slider */
  transition: opacity 0.3s ease, visibility 0.3s ease;
}

.filter-options-container {
  width: 100%;
  max-width: var(--polaroid-width);
  overflow-x: auto;
  overflow-y: hidden;
  -webkit-overflow-scrolling: touch;
  scrollbar-width: none; /* Firefox */
  -ms-overflow-style: none; /* IE and Edge */
  margin-bottom: 10px;
}

.filter-options-container::-webkit-scrollbar {
  display: none; /* Chrome, Safari, Opera */
}

.filter-options {
  display: flex;
  flex-wrap: nowrap;
  gap: 8px;
  padding: 4px;
  min-width: min-content;
}

.filter-button {
  background-color: rgba(0, 0, 0, 0.7);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 20px;
  padding: 8px 15px;
  font-family: "Inter", sans-serif;
  font-size: clamp(0.8rem, calc(0.9rem * var(--scale-ratio)), 1rem);
  color: white;
  cursor: pointer;
  transition: all 0.2s ease;
  white-space: nowrap;
  flex-shrink: 0;
}

.filter-button:hover {
  background-color: rgba(0, 0, 0, 0.8);
  border-color: rgba(255, 255, 255, 0.4);
}

.filter-button.active {
  background-color: rgba(0, 0, 0, 0.9);
  border-color: rgba(255, 255, 255, 0.7);
  font-weight: bold;
}

/* Filter intensity slider container and slider */
.filter-intensity-container {
  width: 100%;
  min-height: 30px; /* Reserve space for the intensity slider */
}

.filter-intensity {
  display: flex;
  align-items: center;
  width: 100%;
  max-width: 300px;
  padding: 0 10px;
  margin: 0 auto;
  transition: opacity 0.3s ease, visibility 0.3s ease;
}

.filter-intensity label {
  font-family: "Inter", sans-serif;
  font-size: clamp(0.8rem, calc(0.9rem * var(--scale-ratio)), 1rem);
  color: white;
  margin-right: 10px;
  white-space: nowrap;
}

.filter-intensity input[type="range"] {
  flex: 1;
  height: 6px;
  -webkit-appearance: none;
  appearance: none;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 3px;
  outline: none;
}

.filter-intensity input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background: white;
  cursor: pointer;
}

.filter-intensity input[type="range"]::-moz-range-thumb {
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background: white;
  cursor: pointer;
  border: none;
}

.button-icon {
  margin-right: 8px;
  font-size: clamp(0.9rem, calc(1.2rem * var(--scale-ratio)), 1.4rem);
}

/* Responsive adjustments */
@media (max-width: 600px) {
  .filter-intensity {
    max-width: 90%;
  }
}

@media (max-width: 400px) {
  .filter-button {
    padding: 4px 8px;
    font-size: 0.7rem;
  }
}
</style>
