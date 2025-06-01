# Suncatcher Photo

A Vue 3 application that creates nostalgic polaroid-style photos with realistic light leak and expired film effects.

## Features

- Upload your own photos to create polaroid-style images
- Add custom messages to your polaroids
- Realistic suncatcher effects that respond to device tilt
- Vintage expired film simulation with color shifts and artifacts
- Download your creations as PNG images

## Technologies Used

- Vue 3 with Composition API and `<script setup>` syntax
- Vite for fast development and optimized builds
- Canvas API for advanced image processing
- Responsive design that works on mobile and desktop

## Getting Started

### Prerequisites

- Node.js version 18.0 or higher

### Installation

1. Clone the repository:
   ```
   git clone https://github.com/yourusername/suncatcher-photo.git
   cd suncatcher-photo
   ```

2. Install dependencies:
   ```
   npm install
   ```

3. Start the development server:
   ```
   npm run dev
   ```

4. Open your browser and navigate to the URL shown in your terminal (typically http://localhost:5173)

## Building for Production

To create a production-ready build:

```
npm run build
```

This will generate optimized files in the `dist` directory that you can deploy to any static hosting service.

## How It Works

The application uses the HTML5 Canvas API to apply various effects to your images:

- **Suncatcher Effect**: Creates beautiful rainbow-like reflections that respond to device orientation
- **Expired Film Effect**: Recreates the look of expired film with color shifts, reduced contrast, film grain, and random artifacts
- **Polaroid Frame**: Adds a classic polaroid-style frame with space for a handwritten message

## Browser Compatibility

This application works best in modern browsers that support the Canvas API and device orientation events. For the full experience including tilt effects, use a mobile device with motion sensors.

## License

[MIT License](LICENSE)

## Acknowledgments

- Inspired by the nostalgic aesthetic of vintage polaroid photography
- Built with Vue 3 and Vite
