[English](./README.md) | [中文](./README_CN.md)

# Simple Level Sequence Smooth

Level sequence data filtering plugin supporting multiple smoothing algorithms for animation data processing.

## Features

- **5 Smoothing Algorithms**:
  - **Moving Average**: Arithmetic mean over a window, quick noise preview
  - **Gaussian**: Weighted average with Gaussian distribution, smooth velocity and acceleration curves
  - **Exponential**: Exponentially weighted recursive filter, recent data weighted more
  - **Butterworth Low-pass**: Frequency-domain filter with zero-phase bidirectional processing, recommended for joint data
  - **Savitzky-Golay**: Local polynomial fitting, preserves peaks and inflection points
- **Content Browser Integration**: Right-click on LevelSequence assets to access smoothing
- **ControlRig Support**: Directly processes ControlRig parameter sections (Transform and Scalar)
- **Flexible Output**: Choose to overwrite the original sequence or create a new one
- **Batch Processing**: Process multiple sequences at once with consistent suffix naming
- **Auto Frame Rate Detection**: Automatically reads the sequence frame rate for Butterworth sampling rate

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

| Algorithm | Best For | Key Parameter |
|-----------|----------|---------------|
| Moving Average | Quick preview | Window Size |
| Gaussian | Large joints (arms, legs) | Window Size, Sigma |
| Exponential | Real-time preprocessing | Alpha (decay factor) |
| Butterworth | Joint data (recommended) | Cutoff Frequency |
| Savitzky-Golay | Fine joints (fingers) | Window Size, Poly Order |

## Dependencies

- ControlRig plugin (enabled by default)

## Supported Engine Versions

- Unreal Engine 5.2+

## Contact

For questions or feedback, please leave a comment on the Fab product page.
