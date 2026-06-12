[English](./README.md) | [中文](./README_CN.md)

# SimpleAssetCleaner Plugin Tutorial

**SimpleAssetCleaner** automatically identifies and removes unreferenced assets from your Unreal Engine project, keeping your repository clean.

> **Important**: This plugin can delete assets from source control. Read carefully before use.

**Compatibility**: UE 5.2 ~ 5.7

---

## What Gets Cleaned

### Only Project Content
- **ONLY** assets in your project's `/Content/` folder (`/Game/` path)

### Never Touched
- Plugin assets (e.g., `/PluginName/Content/`)
- Engine assets
- Code files (C++ headers/source)
- External references outside `/Content/`

---

## How It Works

The plugin uses **reference chain analysis**:

1. **Root Assets** - Always considered "in use" (Maps, Blueprints, DataAssets, etc.)
2. **Reference Chain** - Any asset referenced by a root (directly or indirectly) is "in use"
3. **Unreferenced Assets** - Assets not reachable from any root are cleaned

### Auto-Clean on Submit (Recommended)
- Triggers when opening source control submit dialog
- Only checks newly added files
- Reverts unreferenced files from source control
- **Local copies stay safe** - never deletes from disk

---

## Auto-Add Dependencies on Submit

When submitting a changelist, the plugin automatically scans all assets in the changelist and checks their dependencies.

- If any dependency files are **not under version control**, a confirmation dialog pops up listing them
- You can check/uncheck individual files, then confirm to add them to the changelist
- This prevents broken references caused by forgetting to submit dependency assets
- Can be toggled via the **Enable Auto Add Dependencies** setting in Configuration

---

## Context Menu

Right-click any folder or asset(s) in the Content Browser to access the **Simple Asset Cleaner** submenu with two actions:

### Clean Unreferenced Assets
- Scans the selected scope (folder or selected assets) for unreferenced assets
- Shows a dialog with checkboxes listing all unreferenced assets found
- Confirm to clean (revert or delete) the selected assets
- Behavior matches the same rules as auto-clean: newly added files are reverted (local copies kept), already tracked files are deleted from server and local

### Add Uncontrolled Dependencies
- Checks all dependencies of the selected assets or folder
- Finds dependency files that are not under version control
- Shows a confirmation dialog to add them to source control
- Useful for batch-adding missing dependencies before submission

Both actions support **multi-selection** for files and folders.

---

## Configuration

**Location**: Editor > Project Settings > General > Simple Asset Cleaner

![Settings Screenshot](./Images/ShowSettings.png)

### Source Control Provider (Important!)

The plugin automatically wraps your source control provider as a proxy to monitor submissions.

![Proxy Provider](./Images/ShowProxy.png)

**DO NOT select the proxy provider manually!**
- Always select your actual source control provider (Perforce, Git, SVN, etc.)
- The plugin automatically wraps it in the background
- Selecting the proxy directly may cause issues

**Correct Setup**:
- Use: `Perforce`, `Git`, `Subversion`, etc.
- Avoid: `Perforce_Proxy`, `Git_Proxy`, or any provider ending with `_Proxy`

### General Settings

#### Enable Auto Clean
- **Default**: ON
- **What it does**: Checks references before submission
- **Behavior**: Reverts unreferenced newly added files (local files safe)
- **Recommendation**: Keep enabled for daily use

#### Enable Auto Add Dependencies
- **Default**: ON
- **What it does**: Scans dependencies of changelist assets during submission and detects files not under version control
- **Behavior**: Shows a confirmation dialog listing uncontrolled dependencies; checked files are added to the changelist
- **Recommendation**: Keep enabled to avoid missing dependency submissions

#### Language
- **Options**: Chinese / English
- **Default**: English
- **What it does**: Switches the language for all UI text, tooltips, and log output
- **Behavior**: Takes effect immediately (real-time switching, no restart required)

#### Enable Verbose Logging
- **Default**: OFF
- **What it does**: Detailed logging for debugging
- **Recommendation**: Enable only when troubleshooting

### Root Asset Configuration

#### Root Asset Classes
- **Default**: `UWorld` (Map files)
- **What it does**: Assets of these types (including parent/derived classes) are treated as roots
- **Examples**:
  - `UWorld` - Maps (.umap) and derived types
  - `UDataAsset` - Data assets and derived types
- **Note**: Automatically includes parent classes and all derived classes

#### Custom Root Assets
- **What it does**: Manually specify individual assets as roots
- **Example**: `/Game/Core/GameMode.GameMode`
- **Use case**: Critical assets not covered by Root Asset Classes

#### Additional Root Paths
- **What it does**: All assets under these paths are treated as roots
- **Example**: `/Game/Essential/`, `/Game/Core/`
- **Use case**: Protect entire system folders from cleanup

---

## Important Notes

### What Gets Deleted vs Reverted

| Asset Status | Auto Clean on Submit | Context Menu Clean |
|-------------|---------------------|-------------------|
| Newly added | Reverted, **local kept** | Reverted, **local kept** |
| Already tracked | Not touched | **Deleted (server + local)** |

### Auto-Add Dependencies

| Dependency Status | Behavior |
|------------------|----------|
| Under version control | No action needed |
| Not under version control | Listed in confirmation dialog, user selects which to add |

---

## Support

For questions or feedback, please leave a comment on the Fab product page.

**When reporting issues include**:
- Unreal Engine version
- Source control system (Perforce/Git/SVN)
- Plugin settings screenshot
- Log output (with verbose logging enabled)

---

**Platform Support**: Tested on Windows, theoretically supports all platforms

**Remember**: Start conservative. You can always delete assets later, but recovery requires source control history!
