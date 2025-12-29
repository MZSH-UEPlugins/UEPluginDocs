# SimpleConfigExclude Tutorial

Visually manage which config files (.ini) are included or excluded when packaging your Unreal Engine project.

---

## Configuration

**Location**: Editor > Project Settings > Plugins > Simple Config Exclude

### Settings

#### Hide Plugin Config
- **Default**: Enabled
- **Function**: Hides DefaultSimpleConfigExclude.ini from the config list
- **Purpose**: Prevents accidental exclusion of plugin's own settings

#### Clean Config Before Copy
- **Default**: Enabled
- **Function**: Deletes target Config folder before copying excluded files
- **Purpose**: Ensures target directory only contains latest excluded files

### Copy

#### Copy Target Directory
- **Default**: /Windows
- **Function**: Target directory for copying excluded config files
- **Path Format**: Relative to project directory (e.g., `/Windows`, `/Build/Output`)
- **Cross-Platform**: Supports Windows, Mac, Linux
- **Directory Structure**: Preserves original structure (`TargetDir/ProjectName/Config/xxx.ini`)
- **Copy Button**: Copies all Disallowed config files to target directory

### Config Exclude Rules

Tri-state selector for all .ini files in Config folder:

| Policy | Description |
|--------|-------------|
| **Default** | Use UE default behavior (usually based on file type) |
| **Allowed** | Force include in package (even if UE would exclude) |
| **Disallowed** | Force exclude from package |

#### Refresh Config List Button

Click to rescan Config folder. Use when:
- New config files added
- Config files deleted
- List out of sync with actual files

---

## Examples

### Example 1: Exclude Debug Config

If you have `DefaultDebugSettings.ini` with debug hotkeys that shouldn't be in release:

1. Open Project Settings > Plugins > Simple Config Exclude
2. Find `Config/DefaultDebugSettings.ini` in the list
3. Select "Disallowed"
4. Package project - file will be excluded

### Example 2: Force Include Custom Config

If you have `DefaultCustomSettings.ini` that must be packaged:

1. Open plugin settings
2. Find `Config/DefaultCustomSettings.ini`
3. Select "Allowed"
4. Package project - file will be force included

### Example 3: Copy Excluded Files to Packaged Build

After packaging, you may want to manually add excluded configs:

1. Set "Copy Target Directory" to package output path (e.g., `/Windows`)
2. Set configs to exclude as "Disallowed"
3. Package project
4. Click "Copy" button
5. Files copy to `Windows/YourProject/Config/xxx.ini`

---

## Important Notes

### Default Behavior

All config files default to "Default" state, using UE's default packaging behavior. UE automatically decides based on file type (engine.ini, game.ini, input.ini get packaged).

### Plugin Config Protection

Plugin hides `DefaultSimpleConfigExclude.ini` by default because:
- It contains plugin's own settings
- Excluding it may cause plugin malfunction
- Uncheck "Hide Plugin Config" to show it

### Settings Persistence

- Plugin settings saved in `DefaultSimpleConfigExclude.ini`
- Exclusion rules saved in `DefaultGame.ini` `[Staging]` section
- Both files should be committed to version control

### Clean Before Copy

When "Clean Config Before Copy" is enabled:
- Deletes `TargetDir/ProjectName/Config/` folder before copying
- Only copies currently Disallowed files
- Ensures target directory reflects current state

---

## Troubleshooting

### Files Not Showing in List

1. Click "Refresh Config List" button
2. Ensure file is in `Config` folder or subfolders
3. Ensure file has `.ini` extension

### Rules Not Applied After Packaging

1. Check if `DefaultGame.ini` contains `[Staging]` section
2. Try clicking "Refresh Config List"
3. Restart editor and repackage

### Plugin Config Showing in List

Check "Hide Plugin Config" setting, enable it to hide DefaultSimpleConfigExclude.ini

### Extra Files in Target Directory After Copy

Enable "Clean Config Before Copy" option to clean target Config folder before copying

---

## FAQ

**Q: Does this affect game runtime?**
A: No, this is an editor-only plugin that only affects the packaging process.

**Q: Are subfolders supported?**
A: Yes, the plugin recursively scans Config folder and all subfolders, preserving directory structure when copying.

**Q: Will copied excluded configs work?**
A: Depends on config type. Runtime configs (like Input) work, compile-time baked configs (like rendering settings) don't.

---

## Support

**Email**: mengzhishanghun@outlook.com
 
