# Technical Implementation Details

This document covers the technical implementation details of the rPPG Vital Signs application.

## Architecture Overview

The application is a single-page web application built with vanilla HTML, CSS, and JavaScript. It uses no external dependencies or frameworks for the core functionality.

```
┌─────────────────────────────────────────────────────────────┐
│                      Browser                                │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │   HTML5     │  │   Canvas    │  │    JavaScript       │ │
│  │   Video     │  │   2D API    │  │    Processing       │ │
│  │  Element    │  │             │  │                     │ │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘ │
│         │                │                     │            │
│         └────────────────┼─────────────────────┘            │
│                          │                                  │
│                          ▼                                  │
│              ┌───────────────────────┐                      │
│              │   WebRTC getUserMedia │                      │
│              │        API            │                      │
│              └───────────┬───────────┘                      │
│                          │                                  │
└──────────────────────────┼──────────────────────────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │    Camera    │
                    └──────────────┘
```

## File Structure

```
public/
├── index.html      # Main application (beautiful UI)
└── baseline.html   # Original baseline version

docs/
├── ALGORITHM.md    # Algorithm documentation
├── TECHNICAL.md    # This file
└── screenshot.png  # Demo screenshot
```

## Core Components

### 1. Video Capture

```javascript
async function initCamera() {
  const stream = await navigator.mediaDevices.getUserMedia({
    video: { width: WIDTH, height: HEIGHT }
  });
  video.srcObject = stream;
}
```

**Key Points:**
- Uses WebRTC `getUserMedia` API
- Requests specific resolution (640x480)
- Video element is mirrored via CSS for natural UX
- Canvas receives un-mirrored raw data

### 2. Canvas Processing

```javascript
ctx.drawImage(video, 0, 0, WIDTH, HEIGHT);
const frame = ctx.getImageData(0, 0, WIDTH, HEIGHT).data;
```

**Frame Data Format:**
- `ImageData.data` is a `Uint8ClampedArray`
- RGBA format: 4 bytes per pixel
- Total size: 640 × 480 × 4 = 1,228,800 bytes
- Pixel at (x, y): index = (y × WIDTH + x) × 4

### 3. Timing System

```javascript
measurementInterval = setInterval(() => {
  processFrame();
}, 1000 / FPS);  // ~33.3ms interval
```

**Timing Considerations:**
- `setInterval` provides ~30 FPS processing
- Actual frame timing may vary slightly
- Real-time elapsed tracked via `Date.now()`

### 4. State Management

All state is managed via simple JavaScript variables:

```javascript
let signal = [];           // Time-series buffer
let roiMask = [];          // Precomputed ROI pixels
let bpmHistory = [];       // HR measurements
let respHistory = [];      // RR measurements
let measurementInterval;   // Timer reference
let startTime;             // Measurement start timestamp
let currentBPM = 0;        // Current display value
let currentRPM = 0;        // Current display value
```

## CSS Architecture

### Design System

The beautiful version uses a custom design system:

```css
/* Color Palette */
--bg-dark: #0f172a;
--bg-card: rgba(30, 41, 59, 0.5);
--text-primary: #ffffff;
--text-secondary: #64748b;
--accent-purple: #6366f1;
--accent-pink: #ec4899;
--accent-cyan: #06b6d4;
--accent-green: #10b981;
--accent-red: #ff6b6b;
```

### Glassmorphism Effect

```css
.app {
  background: rgba(30, 41, 59, 0.5);
  backdrop-filter: blur(20px);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 24px;
}
```

### Animations

**Heartbeat Logo:**
```css
@keyframes heartbeat {
  0%, 100% { transform: scale(1); }
  15% { transform: scale(1.15); }
  30% { transform: scale(1); }
  45% { transform: scale(1.1); }
  60% { transform: scale(1); }
}
```

**Pulsing Indicators:**
```javascript
function createPulsingDot(dotElem, getBPM) {
  function animate(timestamp) {
    const bpm = getBPM();
    const freq = bpm / 60;
    phase += 2 * Math.PI * freq * dt;

    const scale = 1 + 0.3 * Math.sin(phase);
    const opacity = 0.7 + 0.3 * Math.sin(phase);

    dotElem.style.transform = `scale(${scale})`;
    dotElem.style.opacity = `${opacity}`;

    requestAnimationFrame(animate);
  }
}
```

## Performance Optimizations

### 1. Precomputed ROI Mask

The oval mask with Gaussian weights is computed once at initialization:

```javascript
// Computed once
roiMask = createOvalMask();  // ~52,000 pixels

// Used every frame
for (const [x, y, w] of roiMask) {
  // Direct array access, no recalculation
}
```

### 2. In-Place FFT

The FFT algorithm modifies arrays in place to minimize memory allocation:

```javascript
// Bit-reversal in place
[arr[i], arr[j]] = [arr[j], arr[i]];

// Butterfly in place
re[l] = re[k] - tRe;
re[k] += tRe;
```

### 3. Rolling Buffer

Using `shift()` for the signal buffer maintains constant memory:

```javascript
signal.push(meanGreen);
if (signal.length > BUFFER_SIZE) signal.shift();
```

## Browser Compatibility

### Required APIs

| API | Purpose | Fallback |
|-----|---------|----------|
| `getUserMedia` | Camera access | None (required) |
| `Canvas 2D` | Image processing | None (required) |
| `requestAnimationFrame` | Smooth animations | `setTimeout` |
| `Array.prototype.fill` | Array initialization | Polyfill available |

### CSS Features

| Feature | Browser Support |
|---------|----------------|
| CSS Grid | All modern browsers |
| Flexbox | All modern browsers |
| `backdrop-filter` | Chrome 76+, Firefox 103+, Safari 9+ |
| CSS Custom Properties | All modern browsers |
| `conic-gradient` | Chrome 69+, Firefox 83+, Safari 12.1+ |

### Graceful Degradation

The baseline version uses simpler CSS that works in older browsers while maintaining full functionality.

## Security Considerations

### Camera Permissions

- Camera access requires explicit user permission
- Permission prompt is browser-controlled
- HTTPS required for production deployment (except localhost)

### Data Privacy

- No data leaves the browser
- No external API calls
- No cookies or local storage used
- All processing is client-side

### Content Security

The application could be enhanced with CSP headers:

```html
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self';
               script-src 'self' 'unsafe-inline';
               style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
               font-src https://fonts.gstatic.com;">
```

## Error Handling

### Camera Errors

```javascript
try {
  const stream = await navigator.mediaDevices.getUserMedia({...});
} catch (err) {
  if (err.name === 'NotAllowedError') {
    // User denied permission
  } else if (err.name === 'NotFoundError') {
    // No camera available
  } else if (err.name === 'NotReadableError') {
    // Camera in use by another app
  }
}
```

### Video Ready State

```javascript
if (video.readyState < 2) return;  // HAVE_CURRENT_DATA
```

Video ready states:
- 0: HAVE_NOTHING
- 1: HAVE_METADATA
- 2: HAVE_CURRENT_DATA
- 3: HAVE_FUTURE_DATA
- 4: HAVE_ENOUGH_DATA

## Testing

### Manual Testing Checklist

- [ ] Camera permission prompt appears
- [ ] Video displays with mirrored orientation
- [ ] Green oval overlay is visible and centered
- [ ] Start button enables after camera loads
- [ ] Timer counts up correctly
- [ ] HR and RR values update after ~17 seconds
- [ ] Values are within physiological range
- [ ] Measurement completes at 60 seconds
- [ ] Final averages and std dev are displayed
- [ ] "New Measurement" button works

### Performance Testing

Monitor these metrics during measurement:
- Frame processing time: should be < 33ms
- Memory usage: should be stable ~50MB
- CPU usage: should be < 10% on modern hardware

## Deployment

### Static Hosting

The `public/` folder can be deployed to any static host:

```bash
# Netlify
netlify deploy --dir=public

# Vercel
vercel public

# GitHub Pages
# Push public/ contents to gh-pages branch
```

### HTTPS Requirement

`getUserMedia` requires a secure context:
- `https://` in production
- `http://localhost` for development
- `file://` may work in some browsers

## Future Improvements

Potential enhancements:

1. **Face Detection**: Auto-position ROI using face detection API
2. **Motion Compensation**: Detect and compensate for head movement
3. **Multi-channel Analysis**: Use RGB channels for better noise rejection
4. **Adaptive Filtering**: Bandpass filter tuned to detected heart rate
5. **Quality Metrics**: Signal quality indicator for user feedback
6. **Data Export**: Allow users to save measurement history
7. **PWA Support**: Offline capability and installability
