[English](./README.md) | [中文](./README_CN.md)

# Simple Level Sequence Smooth

Level sequence data filtering plugin with 12 smoothing algorithms for animation data processing.

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

## Algorithm Guide

### Time-Domain Filters

| Algorithm | Best For | Key Parameter |
|-----------|----------|---------------|
| Moving Average | Quick preview | Window Size |
| Gaussian | Large joints (arms, legs) | Window Size, Sigma |
| Exponential | Real-time preprocessing | Alpha (decay factor) |
| Median | Impulse noise / frame jumps | Window Size |
| Bilateral | Edge-preserving smoothing | Spatial Sigma, Range Sigma |

### Frequency-Domain Filters

| Algorithm | Best For | Key Parameter |
|-----------|----------|---------------|
| Butterworth Low-pass | Joint data (recommended) | Cutoff Frequency |
| Wavelet Denoising | Multi-scale noise separation | Decomposition Level, Threshold |

### State Estimation

| Algorithm | Best For | Key Parameter |
|-----------|----------|---------------|
| Kalman | Trending motion data | Process Noise (Q), Measurement Noise (R) |

### Adaptive Filters

| Algorithm | Best For | Key Parameter |
|-----------|----------|---------------|
| One Euro | Real-time / VR / AR / mocap | Min Cutoff, Beta (speed coefficient) |
| Complementary | IMU mocap data | Complementary Alpha |

### AI-Specific

| Algorithm | Best For | Key Parameter |
|-----------|----------|---------------|
| Savitzky-Golay | Fine joints (fingers) | Window Size, Poly Order |
| Confidence-Weighted | AI pose estimation | Window Size, Sensitivity |

## Dependencies

- ControlRig plugin (enabled by default)

## Supported Engine Versions

- Unreal Engine 5.2+

## Contact

For questions or feedback, please leave a comment on the Fab product page.
