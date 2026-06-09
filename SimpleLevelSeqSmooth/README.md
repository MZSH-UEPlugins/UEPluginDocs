[English](./README.md) | [中文](./README_CN.md)

# Simple Level Sequence Smooth

Level sequence data filtering plugin with 12 smoothing algorithms covering mocap, post-processing, and AI scenarios.

## Features

- **12 Smoothing Algorithms** covering time-domain, frequency-domain, state estimation, adaptive, and AI-specific approaches
- **Content Browser Integration**: Right-click on LevelSequence assets to access smoothing
- **ControlRig Support**: Directly processes ControlRig parameter sections (Transform and Scalar)
- **Flexible Output**: Choose to overwrite the original sequence or create a new one
- **Batch Processing**: Process multiple sequences at once with consistent suffix naming
- **Auto Frame Rate Detection**: Automatically reads the sequence frame rate

## Installation

1. Copy the `SimpleLevelSeqSmooth` folder to your project's `Plugins/` directory
2. Restart the Unreal Editor
3. Enable the plugin in Edit > Plugins if not already enabled

## Usage

1. In the Content Browser, right-click on one or more LevelSequence assets
2. Select **Smooth** from the context menu
3. Choose a smoothing algorithm and adjust parameters
4. Select output mode (overwrite or create new)
5. Click **OK** to apply

## Algorithm Overview

| # | Algorithm | Category | Key Advantage | Best For |
|---|-----------|----------|---------------|----------|
| 1 | Moving Average | Time-domain | Simplest, zero configuration | Quick preview |
| 2 | Gaussian | Time-domain | Velocity/acceleration curves also smooth, no abrupt changes | Large joints (arms, legs) |
| 3 | Exponential | Time-domain | Recent data weighted more, fast response | Real-time preprocessing |
| 4 | Median | Time-domain | Only filter that cleanly removes impulse noise / frame jumps | Mocap data first pass |
| 5 | Bilateral | Time-domain | Smooths noise while preserving sudden stops / motion edges | Actions with clear start/end |
| 6 | Butterworth Low-pass | Frequency-domain | Precise cutoff frequency, zero-phase no delay | Joint data (recommended) |
| 7 | Wavelet Denoising | Frequency-domain | Multi-scale separation, automatic noise level estimation | Fine joints / compound noise |
| 8 | Kalman | State Estimation | Combines smoothing with velocity prediction, optimal estimate | Trending motion data |
| 9 | One Euro | Adaptive | Strong smoothing at rest, weak at fast motion, minimal latency | VR / AR / real-time interaction |
| 10 | Complementary | Adaptive | Fuses integration path + direct measurement, zero-phase | IMU inertial mocap |
| 11 | Savitzky-Golay | Polynomial | Precisely preserves peaks and inflection points | Fine joints (fingers) |
| 12 | Confidence-Weighted | AI-specific | Auto-detects outlier frames and reduces their weight | AI pose estimation |

## Algorithm Parameters

### 1. Moving Average

Arithmetic mean over a sliding window.

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| Window Size | 51 | 3–501 (odd) | Number of frames to average, larger = smoother |

### 2. Gaussian

Gaussian distribution weighted average, center-weighted with symmetric falloff.

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| Window Size | 51 | 3–501 (odd) | Reference frame range |
| Sigma | 12.0 | 0.1–100 | Gaussian width, recommended 1/6–1/3 of window size |

### 3. Exponential

Exponentially weighted recursive filter, recent data weighted more.

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| Alpha | 0.15 | 0.01–1.0 | Smaller = smoother: 0.01 = strong, 0.15 = recommended, 0.5 = weak |

### 4. Median

Selects the middle value in a sliding window, best for impulse noise.

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| Window Size | 51 | 3–501 (odd) | Larger window removes more noise but loses detail |

### 5. Bilateral

Weights by both spatial distance and intensity difference, edge-preserving.

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| Window Size | 51 | 3–501 (odd) | Spatial reference range |
| Spatial Sigma | 12.0 | 0.1–100 | Controls spatial smoothing range, larger = smoother |
| Range Sigma | 0.5 | 0.001–100 | Controls intensity tolerance, larger = ignores edges, smaller = preserves edges |

### 6. Butterworth Low-pass

Second-order frequency-domain filter with zero-phase bidirectional processing.

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| Cutoff Frequency (Hz) | 10.0 | 0.1–60 | Motion below this frequency is preserved. Human body < 6 Hz, fingers < 10 Hz |
| Sample Rate (Hz) | Auto | 1–1000 | Auto-filled from sequence frame rate |

### 7. Wavelet Denoising

Haar wavelet transform + MAD noise estimation + soft thresholding.

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| Decomposition Level | 4 | 1–10 | More levels = wider frequency range processed, typically 3–5 |
| Threshold Coefficient | 0.5 | 0.01–5.0 | < 1 preserves more detail, 1 = standard, > 1 = stronger denoising |

### 8. Kalman

Optimal state estimation based on constant-velocity motion model (position + velocity).

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| Process Noise (Q) | 0.01 | 0.0001–10 | Motion model uncertainty, larger = trusts measurement more (tighter tracking) |
| Measurement Noise (R) | 0.1 | 0.0001–100 | Sensor/mocap noise level, larger = stronger smoothing |

### 9. One Euro

Adaptive cutoff frequency low-pass filter (CHI 2012), strong smoothing at rest, weak at fast motion.

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| Min Cutoff | 1.0 | 0.001–30 | Cutoff frequency at rest, smaller = smoother when still |
| Beta (Speed Coefficient) | 0.007 | 0–1.0 | Speed influence on cutoff: 0 = fixed low-pass, > 0 = reduces smoothing at fast motion |
| Derivative Cutoff | 1.0 | 0.001–30 | Smoothing of the speed estimate itself, usually keep default |
| Sample Rate (Hz) | Auto | 1–1000 | Auto-filled from sequence frame rate |

### 10. Complementary

Forward-backward complementary filter, fuses integration path and direct measurement, zero-phase.

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| Complementary Alpha | 0.98 | 0.5–0.999 | Larger = trusts integration (preserves detail), smaller = trusts measurement (suppresses drift). Recommended 0.95–0.99 |

### 11. Savitzky-Golay

Local polynomial fitting that preserves peaks and inflection points while smoothing.

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| Window Size | 21 | 5–501 (odd) | Local fitting range |
| Polynomial Order | 3 | 2–6 | Fit degree: 2 = parabola, 3 = cubic (recommended), 4 = quartic |

### 12. Confidence-Weighted

Auto-detects outlier frames using MAD robust statistics and reduces their weight.

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| Window Size | 51 | 3–501 (odd) | Local analysis and smoothing range |
| Sensitivity | 2.0 | 0.1–10 | Frames beyond this many standard deviations are flagged. Smaller = more sensitive: 1.5 = sensitive, 2.0 = standard, 3.0 = lenient |

## Dependencies

- ControlRig plugin (enabled by default)

## Supported Engine Versions

- Unreal Engine 5.2+

## Contact

For questions or feedback, please leave a comment on the Fab product page.
