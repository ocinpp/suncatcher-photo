<template>
  <div class="container">
    <div class="polaroid-text">Create your sunshine photo</div>

    <div class="polaroid-wrapper">
      <div
        class="polaroid"
        ref="polaroid"
        @click="triggerFileInput"
        :style="{ backgroundColor: currentColour.colour1 }"
      >
        <div class="image-container">
          <!-- Skeleton loader that shows while image is loading -->
          <div class="skeleton-loader" v-if="isLoading"></div>
          <canvas
            ref="canvas"
            class="canvas"
            :class="{ 'image-loaded': hasImage }"
          ></canvas>
        </div>

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
            :style="{
              '--placeholderTextColor': currentColour.colour3,
              color: currentColour.colour2,
            }"
          ></textarea>
        </div>
      </div>
    </div>

    <!-- Filter controls - always present but conditionally visible -->
    <div class="filter-controls">
      <!-- Horizontal scrollable colour selector -->
      <div
        class="colour-options-container"
        :style="{
          opacity: hasImage ? 1 : 0,
          visibility: hasImage ? 'visible' : 'hidden',
        }"
      >
        <div class="colour-options">
          <button
            v-for="colour in availableColours"
            :key="colour.id"
            @click.stop="selectColour(colour)"
            :class="{
              active: currentColour.id === colour.id,
            }"
            :style="{
              backgroundColor: colour.colour1,
            }"
            class="colour-button"
          >
            &nbsp;
          </button>
        </div>
      </div>

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

// Available colours for the polaroid
const availableColours = [
  {
    id: "original",
    name: "Original",
    colour1: "white",
    colour2: "black",
    colour3: "#aaa",
  },
  {
    id: "red",
    name: "Red",
    colour1: "red",
    colour2: "white",
    colour3: "white",
  },
  {
    id: "orange",
    name: "Orange",
    colour1: "orange",
    colour2: "blue",
    colour3: "blue",
  },
  {
    id: "yellow",
    name: "Yellow",
    colour1: "yellow",
    colour2: "violet",
    colour3: "violet",
  },
  {
    id: "green",
    name: "Green",
    colour1: "green",
    colour2: "orange",
    colour3: "orange",
  },
  {
    id: "blue",
    name: "Blue",
    colour1: "blue",
    colour2: "yellow",
    colour3: "yellow",
  },
  {
    id: "indigo",
    name: "Indigo",
    colour1: "indigo",
    colour2: "white",
    colour3: "white",
  },
  {
    id: "violet",
    name: "Violet",
    colour1: "violet",
    colour2: "indigo",
    colour3: "indigo",
  },
];

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
const currentColour = ref(availableColours[0]); // Default colour is the first one
let backgroundCanvas = null;
// let originalImageData = null; // Store original image data for filters
let processingCanvas = null; // For complex filter processing

// Canvas and effect variables
let ctx = null;
let image = new Image();
let noiseTexture = null;
let artifactTexture = null;
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

// Select a colour
function selectColour(colour) {
  currentColour.value = colour;
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
  artifactTexture = createArtifactTexture(
    canvas.value.width,
    canvas.value.height
  );
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

// Artifact texture (coarser for scratches, stains)
function createArtifactTexture(width, height) {
  const artifactTexture = document.createElement("canvas");
  artifactTexture.width = 128;
  artifactTexture.height = 128;
  const artifactCtx = artifactTexture.getContext("2d", {
    willReadFrequently: true,
  });
  const artifactData = artifactCtx.createImageData(128, 128);
  for (let i = 0; i < artifactData.data.length; i += 4) {
    const value = Math.random() < 0.05 ? 255 : 0; // Sparse noise
    artifactData.data[i] = value;
    artifactData.data[i + 1] = value;
    artifactData.data[i + 2] = value;
    artifactData.data[i + 3] = value ? 128 : 0; // Semi-transparent
  }
  artifactCtx.putImageData(artifactData, 0, 0);
  return artifactTexture;
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
  const imageData = tempCtx.getImageData(0, 0, width, height, {
    willReadFrequently: true,
  });
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
      const blurredData = tempCtx.getImageData(0, 0, width, height, {
        willReadFrequently: true,
      }).data;

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

  // drawLightLeakEffect();
  drawExpiredFilmEffect();

  // Apply special effects after the filter
  // drawHologramEffect();
  applySunCatcherEffect();
}

function drawHolographicSheen() {
  if (!ctx || !canvas.value) return;

  ctx.save();
  const tiltIntensity = Math.min(Math.abs(tiltX) + Math.abs(tiltY), 1);
  ctx.globalAlpha = 0.2 * tiltIntensity;
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
  tempCtx.fillStyle = currentColour.value.colour1 || "white";
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
    tempCtx.fillStyle = currentColour.value.colour2 || "black";
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

// Add to existing code before drawImage
function drawHologramEffect() {
  if (!ctx || !canvas.value) return;

  ctx.save();
  const tiltIntensity = Math.min(Math.abs(tiltX) + Math.abs(tiltY), 1);
  const tiltAngle = Math.atan2(tiltY, tiltX); // Gradient angle from tilt
  const diagonal =
    Math.sqrt(
      canvas.value.width * canvas.value.width +
        canvas.value.height * canvas.value.height
    ) * 2; // Oversized for full coverage

  // Iridescent gradient (oversized, absolute coordinates)
  const gradX1 = canvas.value.width / 2 - Math.cos(tiltAngle) * diagonal;
  const gradY1 = canvas.value.height / 2 - Math.sin(tiltAngle) * diagonal;
  const gradX2 = canvas.value.width / 2 + Math.cos(tiltAngle) * diagonal;
  const gradY2 = canvas.value.height / 2 + Math.sin(tiltAngle) * diagonal;
  const gradient = ctx.createLinearGradient(gradX1, gradY1, gradX2, gradY2);
  gradient.addColorStop(0, `hsla(200, 50%, 50%, ${0.5 * tiltIntensity})`); // Blue
  gradient.addColorStop(0.25, `hsla(160, 50%, 50%, ${0.5 * tiltIntensity})`); // Green
  gradient.addColorStop(0.5, `hsla(0, 0%, 70%, ${0.5 * tiltIntensity})`); // Silver
  gradient.addColorStop(0.75, `hsla(40, 50%, 60%, ${0.5 * tiltIntensity})`); // Gold
  gradient.addColorStop(1, `hsla(200, 50%, 50%, ${0.5 * tiltIntensity})`); // Blue
  ctx.globalCompositeOperation = "source-over"; // Consistent blending
  ctx.fillStyle = gradient;
  ctx.fillRect(0, 0, canvas.value.width, canvas.value.height); // Full canvas rect

  // Glossy highlight (full-width, softer)
  const highlightX1 =
    canvas.value.width / 2 - Math.cos(tiltAngle + Math.PI / 4) * diagonal;
  const highlightY1 =
    canvas.value.height / 2 - Math.sin(tiltAngle + Math.PI / 4) * diagonal;
  const highlightX2 =
    canvas.value.width / 2 + Math.cos(tiltAngle + Math.PI / 4) * diagonal;
  const highlightY2 =
    canvas.value.height / 2 + Math.sin(tiltAngle + Math.PI / 4) * diagonal;
  const highlightGradient = ctx.createLinearGradient(
    highlightX1,
    highlightY1,
    highlightX2,
    highlightY2
  );
  highlightGradient.addColorStop(0, `hsla(0, 0%, 100%, 0)`);
  highlightGradient.addColorStop(
    0.3,
    `hsla(0, 0%, 100%, ${0.1 * tiltIntensity})`
  );
  highlightGradient.addColorStop(
    0.7,
    `hsla(0, 0%, 100%, ${0.1 * tiltIntensity})`
  );
  highlightGradient.addColorStop(1, `hsla(0, 0%, 100%, 0)`);
  ctx.globalCompositeOperation = "source-over";
  ctx.fillStyle = highlightGradient;
  ctx.fillRect(0, 0, canvas.value.width, canvas.value.height); // Full canvas rect

  ctx.restore();
}

function addDiffractionLines() {
  if (!ctx || !canvas.value) return;

  const tiltIntensity = Math.min(Math.abs(tiltX) + Math.abs(tiltY), 1);
  const diagonal =
    Math.sqrt(
      canvas.value.width * canvas.value.width +
        canvas.value.height * canvas.value.height
    ) * 2; // Oversized for full coverage

  // Micro-texture (diagonal lines, oversized)
  ctx.save();
  ctx.globalAlpha = 0.15 * tiltIntensity;
  ctx.translate(canvas.value.width / 2, canvas.value.height / 2);
  ctx.rotate(Math.PI / 4); // Fixed angle for diffraction lines
  for (let i = -diagonal; i < diagonal; i += 5) {
    ctx.beginPath();
    ctx.moveTo(i, -diagonal);
    ctx.lineTo(i, diagonal);
    ctx.strokeStyle = `hsla(0, 0%, 100%, 0.2)`;
    ctx.lineWidth = 1;
    ctx.stroke();
  }

  ctx.globalCompositeOperation = "source-over";
  ctx.globalAlpha = 1;
  ctx.restore();
}

// Revised Light Leak Effect
function drawLightLeakEffect() {
  if (!ctx || !canvas.value) return;

  ctx.save();
  const tiltIntensity = Math.min(Math.abs(tiltX) + Math.abs(tiltY), 1);

  // Generate streaks once (cached)
  if (!window.lightLeakCache) {
    window.lightLeakCache = [];
    const streakCount = 1 + Math.floor(Math.random() * 1); // 1–3 streaks
    for (let i = 0; i < streakCount; i++) {
      let x = Math.random() * canvas.value.width;
      let y = Math.random() * canvas.value.height;
      if (Math.random() < 0.3) {
        // 30% edge bias
        const edge = Math.floor(Math.random() * 4);
        if (edge === 0) x = Math.random() * 30; // Left
        else if (edge === 1)
          x = canvas.value.width - Math.random() * 30; // Right
        else if (edge === 2) y = Math.random() * 30; // Top
        else y = canvas.value.height - Math.random() * 30; // Bottom
      }
      const length = 500 + Math.random() * 150; // 100–250px
      const width = 500 + Math.random() * 60; // 20–80px
      const angle = Math.random() * Math.PI * 2;
      const colors = [
        {
          // Red → yellow → cyan
          start: { hue: 0, sat: 90, light: 60 },
          mid1: { hue: 60, sat: 95, light: 65 },
          mid2: { hue: 180, sat: 90, light: 60 },
        },
        {
          // Orange → cyan → magenta
          start: { hue: 30, sat: 90, light: 60 },
          mid1: { hue: 180, sat: 95, light: 65 },
          mid2: { hue: 300, sat: 90, light: 60 },
        },
        {
          // Blue → magenta → yellow
          start: { hue: 200, sat: 90, light: 60 },
          mid1: { hue: 300, sat: 95, light: 65 },
          mid2: { hue: 60, sat: 90, light: 60 },
        },
      ];
      const color = colors[Math.floor(Math.random() * colors.length)];
      window.lightLeakCache.push({ x, y, length, width, angle, color });
    }
  }

  // Draw static streaks, intensity only
  ctx.globalCompositeOperation = "screen";
  window.lightLeakCache.forEach((streak) => {
    const startX = streak.x;
    const startY = streak.y;
    const endX = streak.x + Math.cos(streak.angle) * streak.length;
    const endY = streak.y + Math.sin(streak.angle) * streak.length;
    const cpX = streak.x + Math.cos(streak.angle) * streak.length * 0.5;
    const cpY = streak.y + Math.sin(streak.angle) * streak.length * 0.5;
    const gradient = ctx.createLinearGradient(startX, startY, endX, endY);
    gradient.addColorStop(
      0,
      `hsla(${streak.color.start.hue}, ${streak.color.start.sat}%, ${
        streak.color.start.light
      }%, ${0.4 * tiltIntensity})`
    );
    gradient.addColorStop(
      0.33,
      `hsla(${streak.color.mid1.hue}, ${streak.color.mid1.sat}%, ${
        streak.color.mid1.light
      }%, ${0.35 * tiltIntensity})`
    );
    gradient.addColorStop(
      0.67,
      `hsla(${streak.color.mid2.hue}, ${streak.color.mid2.sat}%, ${
        streak.color.mid2.light
      }%, ${0.3 * tiltIntensity})`
    );
    gradient.addColorStop(1, "hsla(0, 0%, 0%, 0)");
    ctx.fillStyle = gradient;
    ctx.beginPath();
    ctx.moveTo(startX, startY - streak.width / 2);
    ctx.quadraticCurveTo(
      cpX,
      cpY - streak.width / 2,
      endX,
      endY - streak.width / 4
    );
    ctx.lineTo(endX, endY + streak.width / 4);
    ctx.quadraticCurveTo(
      cpX,
      cpY + streak.width / 2,
      startX,
      startY + streak.width / 2
    );
    ctx.closePath();
    ctx.fill();
  });

  ctx.globalCompositeOperation = "source-over";
  ctx.restore();
}

// Revised Expired Film Effect
function drawExpiredFilmEffect() {
  if (!ctx || !canvas.value) return;

  ctx.save();
  const tiltIntensity = Math.min(Math.abs(tiltX) + Math.abs(tiltY), 1);

  // Cache color patches
  if (!window.patchCache) {
    window.patchCache = document.createElement("canvas");
    window.patchCache.width = 32;
    window.patchCache.height = 32;
    const patchCtx = window.patchCache.getContext("2d", {
      willReadFrequently: true,
    });
    const colors = [
      "hsla(180, 30%, 50%, 0.2)", // Cyan
      "hsla(300, 30%, 50%, 0.2)", // Magenta
      "hsla(60, 30%, 50%, 0.2)", // Yellow
      "hsla(120, 30%, 50%, 0.15)", // Green
    ];
    for (let i = 0; i < 5; i++) {
      const x = Math.random() * 32;
      const y = Math.random() * 32;
      const radius = 5 + Math.random() * 10;
      const color = colors[Math.floor(Math.random() * colors.length)];
      const gradient = patchCtx.createRadialGradient(x, y, 0, x, y, radius);
      gradient.addColorStop(0, color);
      gradient.addColorStop(1, "hsla(0, 0%, 0%, 0)");
      patchCtx.fillStyle = gradient;
      patchCtx.beginPath();
      patchCtx.arc(x, y, radius, 0, Math.PI * 2);
      patchCtx.fill();
    }
  }

  // Color shifts (intensity only)
  ctx.globalCompositeOperation = "overlay";
  ctx.globalAlpha = 0.2 * tiltIntensity;
  ctx.drawImage(window.patchCache, 0, 0);
  ctx.globalAlpha = 1;

  // Static grain (intensity only)
  ctx.globalAlpha = 0.15 * tiltIntensity;
  ctx.drawImage(noiseTexture, 0, 0);
  ctx.globalAlpha = 1;

  // Irregular vignette (fixed, intensity only)
  if (!window.vignetteCache) {
    window.vignetteCache = [];
    const segments = 24; // Smoother path
    for (let i = 0; i <= segments; i++) {
      const angle = (i / segments) * Math.PI * 2;
      const radius =
        (canvas.value.width / 2) * (0.85 + Math.sin(angle * 4) * 0.15); // Sine-based waviness
      const x = canvas.value.width / 2 + Math.cos(angle) * radius;
      const y = canvas.value.height / 2 + Math.sin(angle) * radius;
      window.vignetteCache.push({ x, y });
    }
  }

  ctx.beginPath();
  ctx.moveTo(window.vignetteCache[0].x, window.vignetteCache[0].y);
  for (let i = 1; i < window.vignetteCache.length - 1; i++) {
    const cp1X = window.vignetteCache[i].x;
    const cp1Y = window.vignetteCache[i].y;
    const cp2X =
      (window.vignetteCache[i].x + window.vignetteCache[i + 1].x) / 2;
    const cp2Y =
      (window.vignetteCache[i].y + window.vignetteCache[i + 1].y) / 2;
    ctx.bezierCurveTo(cp1X, cp1Y, cp1X, cp1Y, cp2X, cp2Y);
  }
  ctx.closePath();
  const vignetteGradient = ctx.createRadialGradient(
    canvas.value.width / 2,
    canvas.value.height / 2,
    canvas.value.width / 3, // Softer edge
    canvas.value.width / 2,
    canvas.value.height / 2,
    canvas.value.width
  );
  vignetteGradient.addColorStop(0, "hsla(30, 50%, 20%, 0)"); // Darker brown
  vignetteGradient.addColorStop(
    1,
    `hsla(30, 50%, 20%, ${0.4 * tiltIntensity})`
  );
  ctx.globalCompositeOperation = "multiply";
  ctx.fillStyle = vignetteGradient;
  ctx.fillRect(0, 0, canvas.value.width, canvas.value.height); // Full canvas

  // Fading
  const tempCanvas = document.createElement("canvas");
  tempCanvas.width = canvas.value.width;
  tempCanvas.height = canvas.value.height;
  const tempCtx = tempCanvas.getContext("2d", { willReadFrequently: true });
  tempCtx.drawImage(canvas.value, 0, 0);
  const fadeData = tempCtx.getImageData(
    0,
    0,
    canvas.value.width,
    canvas.value.height
  );
  const pixels = fadeData.data;
  for (let i = 0; i < pixels.length; i += 4) {
    pixels[i] = Math.min(255, pixels[i] + 20);
    pixels[i + 1] = Math.min(255, pixels[i + 1] + 20);
    pixels[i + 2] = Math.min(255, pixels[i + 2] + 20);
    const luminance =
      (pixels[i] * 0.299 + pixels[i + 1] * 0.587 + pixels[i + 2] * 0.114) / 255;
    pixels[i] += (luminance * 255 - pixels[i]) * 0.3;
    pixels[i + 1] += (luminance * 255 - pixels[i + 1]) * 0.3;
    pixels[i + 2] += (luminance * 255 - pixels[i + 2]) * 0.3;
  }
  tempCtx.putImageData(fadeData, 0, 0);
  ctx.drawImage(tempCanvas, 0, 0);

  // Artifacts
  ctx.globalAlpha = 0.1;
  ctx.drawImage(artifactTexture, 0, 0);
  for (let i = 0; i < 2; i++) {
    const x1 = Math.random() * canvas.value.width;
    const y1 = Math.random() * canvas.value.height;
    const x2 = x1 + (Math.random() - 0.5) * 50;
    const y2 = y1 + (Math.random() - 0.5) * 50;
    ctx.beginPath();
    ctx.moveTo(x1, y1);
    ctx.quadraticCurveTo(x1 + (x2 - x1) / 2, y1 + (y2 - y1) / 2, x2, y2);
    ctx.strokeStyle = "hsla(0, 0%, 100%, 0.05)";
    ctx.lineWidth = 0.5;
    ctx.stroke();
  }
  for (let i = 0; i < 8; i++) {
    const x = Math.random() * canvas.value.width;
    const y = Math.random() * canvas.value.height;
    ctx.beginPath();
    ctx.arc(x, y, 1, 0, Math.PI * 2);
    ctx.fillStyle = "hsla(0, 0%, 50%, 0.1)";
    ctx.fill();
  }
  for (let i = 0; i < 1; i++) {
    const x = Math.random() * canvas.value.width;
    const y = Math.random() * canvas.value.height;
    const gradient = ctx.createRadialGradient(x, y, 0, x, y, 20);
    gradient.addColorStop(0, "hsla(60, 20%, 50%, 0.1)");
    gradient.addColorStop(1, "hsla(0, 0%, 0%, 0)");
    ctx.fillStyle = gradient;
    ctx.beginPath();
    ctx.arc(x, y, 20, 0, Math.PI * 2);
    ctx.fill();
  }

  // Creamy whites
  tempCtx.clearRect(0, 0, canvas.value.width, canvas.value.height);
  tempCtx.drawImage(canvas.value, 0, 0);
  const whiteData = tempCtx.getImageData(
    0,
    0,
    canvas.value.width,
    canvas.value.height
  );
  const whitePixels = whiteData.data;
  for (let i = 0; i < whitePixels.length; i += 4) {
    const luminance =
      (whitePixels[i] * 0.299 +
        whitePixels[i + 1] * 0.587 +
        whitePixels[i + 2] * 0.114) /
      255;
    if (luminance > 0.8) {
      whitePixels[i] = whitePixels[i] * 0.95 + 20;
      whitePixels[i + 1] = whitePixels[i + 1] * 0.95 + 15;
      whitePixels[i + 2] = whitePixels[i + 2] * 0.95 + 10;
    }
  }
  tempCtx.putImageData(whiteData, 0, 0);
  ctx.drawImage(tempCanvas, 0, 0);

  ctx.globalCompositeOperation = "source-over";
  ctx.globalAlpha = 1;
  ctx.restore();
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
  --placeholderTextColor: "#aaaaaa";
}

body {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  max-height: 100dvh;
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
  height: 100dvh;
  max-height: 100dvh;
  width: 100%;
  max-width: 100vw;
  box-sizing: border-box;
  gap: 8px;
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
  font-size: clamp(1.1rem, calc(1.2rem * var(--scale-ratio)), 1.4rem);
  text-align: left;
  text-transform: uppercase;
  letter-spacing: -0.3px;
  position: relative;
  left: 0;
  width: 100%;
  padding-top: 1.5rem;
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
  color: var(--placeholderTextColor);
}

/* Filter controls */
.filter-controls {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-bottom: 12px;
  width: 100%;
  max-width: var(--polaroid-width);
  min-height: 190px; /* Increased to account for all possible content */
}

/* Top row for all buttons */
.filter-controls-top-row {
  display: flex;
  justify-content: center;
  gap: 10px;
  margin-bottom: 12px;
  width: 100%;
  min-height: 36px; /* Minimum height to prevent layout shift */
  transition: opacity 0.3s ease, visibility 0.3s ease;
}

/* Common button style for all action buttons */
.action-button {
  background-color: rgba(0, 0, 0, 0.7);
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 25px;
  padding: 8px 16px;
  font-family: "Inter", sans-serif;
  font-size: clamp(0.8rem, calc(0.9rem * var(--scale-ratio)), 1rem);
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
  padding: 4px 0;
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

.colour-options-container {
  width: 100%;
  overflow-x: auto;
  -webkit-overflow-scrolling: touch;
  scrollbar-width: none; /* Firefox */
  -ms-overflow-style: none; /* IE and Edge */
  margin-bottom: 12px;
}

.colour-options-container::-webkit-scrollbar {
  display: none; /* Chrome, Safari, Opera */
}

.colour-options {
  display: flex;
  flex-wrap: nowrap;
  gap: 8px;
  padding: 2px 0;
  min-width: min-content;
  justify-content: space-between;
}

.colour-button {
  background-color: rgba(0, 0, 0, 0.7);
  border: 1px solid rgba(255, 255, 255, 0.2);
  /* border-radius: 20px; */
  padding: 8px 15px;
  font-family: "Inter", sans-serif;
  font-size: clamp(0.8rem, calc(0.9rem * var(--scale-ratio)), 1rem);
  color: white;
  cursor: pointer;
  transition: all 0.2s ease;
  white-space: nowrap;
  flex-shrink: 0;
}

.colour-button:hover {
  background-color: rgba(0, 0, 0, 0.8);
  border-color: rgba(255, 255, 255, 0.4);
}

.colour-button.active {
  background-color: rgba(0, 0, 0, 0.9);
  border-color: rgba(255, 255, 255, 0.7);
  font-weight: bold;
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
