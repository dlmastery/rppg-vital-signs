# rPPG Algorithm Documentation

This document provides a detailed explanation of the remote photoplethysmography (rPPG) algorithm used in this application.

## Table of Contents

1. [Overview](#overview)
2. [Signal Acquisition](#signal-acquisition)
3. [Preprocessing](#preprocessing)
4. [Frequency Analysis](#frequency-analysis)
5. [Peak Detection](#peak-detection)
6. [Statistical Processing](#statistical-processing)
7. [Complete Algorithm Flow](#complete-algorithm-flow)

## Overview

The rPPG algorithm extracts physiological signals (heart rate and respiration rate) from video by analyzing subtle color changes in facial skin caused by blood flow.

### Physical Basis

When the heart beats, it pumps blood through the body, causing periodic changes in blood volume in the capillaries near the skin surface. These blood volume changes affect how light is absorbed and reflected by the skin:

- **Systole** (heart contraction): More blood in capillaries → more light absorption → darker skin
- **Diastole** (heart relaxation): Less blood in capillaries → less light absorption → lighter skin

These variations are tiny (typically <1% change in pixel intensity) but can be detected with proper signal processing.

### Why Green Channel?

The green wavelength (~520-560nm) is optimal for detecting blood volume changes because:
- Hemoglobin has strong absorption in this range
- Good penetration depth into skin tissue
- Less affected by melanin variations than blue light
- More sensitive than red light to blood volume changes

## Signal Acquisition

### Region of Interest (ROI)

```javascript
const ROI = { x: 200, y: 100, w: 240, h: 280 };
```

The ROI is an oval region positioned to capture the forehead and cheeks, which have good capillary density and relatively flat surfaces.

### Oval Mask with Gaussian Weighting

```javascript
function createOvalMask() {
  const pixels = [];
  for (let y = ROI.y; y < ROI.y + ROI.h; y++) {
    for (let x = ROI.x; x < ROI.x + ROI.w; x++) {
      // Normalize coordinates to [-1, 1] range
      const dx = (x - (ROI.x + ROI.w / 2)) / (ROI.w / 2);
      const dy = (y - (ROI.y + ROI.h / 2)) / (ROI.h / 2);
      const r2 = dx * dx + dy * dy;

      if (r2 <= 1) {  // Inside ellipse
        // Gaussian weight: highest at center, tapers to 0.5 at edge
        const w = 0.5 + 0.5 * Math.exp(-4 * r2);
        pixels.push([x, y, w]);
      }
    }
  }
  return pixels;
}
```

**Weight Distribution:**
- Center (r=0): weight = 1.0
- Edge (r=1): weight ≈ 0.509
- This reduces influence of edge pixels which are more prone to noise

### Frame Processing

For each video frame:

```javascript
ctx.drawImage(video, 0, 0, WIDTH, HEIGHT);
const frame = ctx.getImageData(0, 0, WIDTH, HEIGHT).data;

let greenSum = 0;
let weightSum = 0;
for (const [x, y, w] of roiMask) {
  const i = (y * WIDTH + x) * 4;  // RGBA index
  greenSum += w * frame[i + 1];    // Green channel (index 1)
  weightSum += w;
}
const meanGreen = greenSum / weightSum;
```

The weighted mean green intensity becomes one sample in our time-series signal.

## Preprocessing

### Signal Buffer

```javascript
signal.push(meanGreen);
if (signal.length > BUFFER_SIZE) signal.shift();  // BUFFER_SIZE = 600
```

We maintain a rolling buffer of the last 600 samples (~20 seconds at 30 FPS).

### Detrending (DC Removal)

Before FFT analysis, we remove the DC component (mean value):

```javascript
const segment = signal.slice(-FFT_SIZE);  // Last 512 samples
const mean = segment.reduce((a, b) => a + b, 0) / FFT_SIZE;
const detrended = segment.map(v => v - mean);
```

This centers the signal around zero, which is required for proper frequency analysis.

## Frequency Analysis

### Fast Fourier Transform (FFT)

We use the Cooley-Tukey radix-2 decimation-in-time FFT algorithm:

```javascript
function fftMagReIm(re) {
  const N = re.length;  // Must be power of 2 (512)
  const im = new Array(N).fill(0);  // Imaginary part starts at 0

  // Step 1: Bit-reversal permutation
  function bitReverse(arr) {
    const n = arr.length;
    const bits = Math.log2(n);  // 9 bits for N=512
    for (let i = 0; i < n; i++) {
      const j = parseInt(
        i.toString(2).padStart(bits, '0').split('').reverse().join(''),
        2
      );
      if (j > i) {
        [arr[i], arr[j]] = [arr[j], arr[i]];
        [im[i], im[j]] = [im[j], im[i]];
      }
    }
  }

  bitReverse(re);

  // Step 2: Butterfly operations
  for (let len = 2; len <= N; len *= 2) {
    const angle = -2 * Math.PI / len;
    for (let i = 0; i < N; i += len) {
      for (let j = 0; j < len / 2; j++) {
        const k = i + j;
        const l = i + j + len / 2;

        // Twiddle factor: W = e^(-2πi*j/len) = cos(θ) - i*sin(θ)
        const tRe = Math.cos(angle * j) * re[l] - Math.sin(angle * j) * im[l];
        const tIm = Math.sin(angle * j) * re[l] + Math.cos(angle * j) * im[l];

        // Butterfly
        re[l] = re[k] - tRe;
        im[l] = im[k] - tIm;
        re[k] += tRe;
        im[k] += tIm;
      }
    }
  }

  // Return magnitude spectrum
  return re.map((r, i) => Math.sqrt(r * r + im[i] * im[i]));
}
```

### Frequency Resolution

With FFT_SIZE = 512 and FPS = 30:

- **Frequency resolution**: Δf = FPS / FFT_SIZE = 30/512 ≈ 0.0586 Hz
- **Bin k corresponds to frequency**: f = k × (FPS / FFT_SIZE)
- **Maximum detectable frequency**: f_max = FPS / 2 = 15 Hz (Nyquist)

## Peak Detection

### Heart Rate Band

```javascript
const hrStart = Math.floor(0.8 / (FPS / FFT_SIZE));   // Bin 13
const hrEnd = Math.ceil(3.5 / (FPS / FFT_SIZE));      // Bin 60
```

This covers 0.8 - 3.5 Hz, corresponding to 48 - 210 BPM.

### Respiration Rate Band

```javascript
const rrStart = Math.floor(0.15 / (FPS / FFT_SIZE));  // Bin 2
const rrEnd = Math.ceil(0.4 / (FPS / FFT_SIZE));      // Bin 7
```

This covers 0.15 - 0.4 Hz, corresponding to 9 - 24 breaths per minute.

### Gaussian-Weighted Peak Selection

```javascript
function gaussianWeight(bin, center, spread) {
  return Math.exp(-0.5 * Math.pow((bin - center) / spread, 2));
}

// For heart rate:
let hrMax = 0, hrBin = 0;
for (let i = hrStart; i <= hrEnd; i++) {
  const w = gaussianWeight(i, (hrStart + hrEnd) / 2, (hrEnd - hrStart) * 1);
  if (w * mag[i] > hrMax) {
    hrMax = w * mag[i];
    hrBin = i;
  }
}
```

The Gaussian weighting:
- Centers preference on the middle of the physiological range
- Reduces likelihood of selecting edge frequencies that may be artifacts
- Spread parameter controls how strongly to prefer center frequencies

### Converting Bin to BPM/RPM

```javascript
const hrFreq = hrBin * FPS / FFT_SIZE;  // Convert bin to Hz
const bpm = hrFreq * 60;                 // Convert Hz to beats per minute
```

## Statistical Processing

### Running History

```javascript
bpmHistory.push(bpm);
respHistory.push(rpm);
```

All measurements are stored for statistical analysis.

### Mean Calculation

```javascript
const avgBPM = bpmHistory.reduce((a, b) => a + b, 0) / bpmHistory.length;
```

### Standard Deviation

```javascript
// Using the computational formula: σ = √(E[X²] - E[X]²)
const sdevBPM = Math.sqrt(
  bpmHistory.reduce((a, b) => a + b**2, 0) / bpmHistory.length
  - avgBPM**2
);
```

## Complete Algorithm Flow

```
┌─────────────────────────────────────────────────────────────┐
│                      VIDEO FRAME                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  1. EXTRACT GREEN CHANNEL from ROI with Gaussian weights    │
│     meanGreen = Σ(weight × green) / Σ(weight)              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  2. ADD TO SIGNAL BUFFER (rolling 600 samples)              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  3. TAKE LAST 512 SAMPLES for analysis                      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  4. DETREND by subtracting mean                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  5. COMPUTE FFT to get frequency spectrum                   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  6. FIND PEAKS in HR band (0.8-3.5 Hz) and RR band         │
│     (0.15-0.4 Hz) using Gaussian-weighted selection         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  7. CONVERT to BPM/RPM and update statistics                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  8. DISPLAY RESULTS                                         │
└─────────────────────────────────────────────────────────────┘
```

## Parameters Summary

| Parameter | Value | Purpose |
|-----------|-------|---------|
| WIDTH | 640 | Video width |
| HEIGHT | 480 | Video height |
| FPS | 30 | Frame rate |
| FFT_SIZE | 512 | FFT window (~17s of data) |
| BUFFER_SIZE | 600 | Signal buffer (~20s) |
| HR_LOW | 0.8 Hz | Min heart rate (48 BPM) |
| HR_HIGH | 3.5 Hz | Max heart rate (210 BPM) |
| RR_LOW | 0.15 Hz | Min respiration (9/min) |
| RR_HIGH | 0.4 Hz | Max respiration (24/min) |

## References

1. Cooley, J. W., & Tukey, J. W. (1965). An algorithm for the machine calculation of complex Fourier series. *Mathematics of Computation*, 19(90), 297-301.

2. Verkruysse, W., Svaasand, L. O., & Nelson, J. S. (2008). Remote plethysmographic imaging using ambient light. *Optics Express*, 16(26), 21434-21445.

3. Poh, M. Z., McDuff, D. J., & Picard, R. W. (2010). Non-contact, automated cardiac pulse measurements using video imaging and blind source separation. *Optics Express*, 18(10), 10762-10774.
