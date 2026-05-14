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

## Troubleshooting

### DataAsset Not Auto-Exported After Module Migration

**Symptom**: After moving a UDataAsset subclass from module A to module B (via CoreRedirects), the DA is not auto-exported to JSON after restarting the editor.

**Cause**: The Asset Registry uses cache files (`Intermediate/CachedAssetRegistry*.bin`) to speed up startup. After module migration, the .uasset file itself hasn't changed on disk, so the cache retains the old class path. During initial sync, the plugin cannot identify the asset as a DataAsset subclass.

**Solution**: Delete the Asset Registry cache files under the project's `Intermediate/` directory, then restart the editor:

```
del Intermediate\CachedAssetRegistry*.bin
```

After restart, the Asset Registry performs a full rescan, CoreRedirects take effect, and the plugin can discover and export the asset normally.

## Supported Engine Versions

- Unreal Engine 5.2+

## Contact

For questions or feedback, please leave a comment on the Fab product page.
