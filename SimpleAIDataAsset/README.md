[English](./README.md) | [中文](./README_CN.md)

# Simple AI DataAsset

Automatic bidirectional binding between DataAssets and JSON files, enabling AI to directly modify DataAssets.

## Features

- **Bidirectional Sync**: Changes in DataAssets automatically export to JSON; JSON file changes automatically import back to DataAssets
- **Initial Sync**: On editor startup, compares timestamps and syncs the latest version
- **Orphan Cleanup**: Automatically removes JSON files whose corresponding DataAssets no longer exist
- **Source Control Integration**: Automatically checks out files via Perforce/Git/SVN before saving
- **Circular Prevention**: Built-in mechanism to prevent infinite sync loops
- **High Precision**: JSON export uses full precision for float/double values
- **Property Name Restoration**: JSON field names match editor display names instead of UE's internal camelCase
- **Smart Write**: Only writes files when content actually changes, avoiding unnecessary version control noise
- **Editor Undo**: JSON imports create editor transactions, fully supporting Ctrl+Z undo

## Installation

1. Copy the `SimpleAIDataAsset` folder to your project's `Plugins/` directory
2. Restart the Unreal Editor
3. Enable the plugin in Edit > Plugins if not already enabled

## Configuration

Go to **Project Settings > Plugins > SimpleAIDataAsset**:

| Setting | Default | Description |
|---------|---------|-------------|
| JSON Output Directory | `Saved/SimpleAIDataAsset` | Directory where JSON files are stored (relative to project root) |
| Enable Auto Sync | `true` | Toggle automatic synchronization on/off |

## How It Works

1. When a DataAsset property changes in the editor, the plugin exports it to a JSON file
2. When a JSON file is modified externally (e.g., by an AI tool), the plugin imports changes back to the DataAsset
3. On startup, the plugin performs an initial sync based on file timestamps

## Supported Engine Versions

- Unreal Engine 5.2+

## Contact

For questions or feedback, please leave a comment on the Fab product page.
