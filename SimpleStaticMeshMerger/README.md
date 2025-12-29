# 📘 SimpleStaticMeshMerger User Guide

This guide provides detailed instructions on how to use the SimpleStaticMeshMerger plugin to merge static mesh instances in Unreal Engine and optimize scene performance.

---

## 🛠️ Opening the Tool Window

After enabling the plugin, find the tool entry in the editor menu:

### Steps:

1. Open Unreal Engine Editor
2. Click top menu `Window > Static Mesh Merger`
3. The tool window will open as a dockable panel

---

## 🎨 Interface Overview

The tool window consists of three main areas:

### 1. Toolbar (Top)

- **Break Level Instances**: Recursively breaks all LevelInstances in the scene (including nested ones)
- **Refresh**: Manually refresh the merge preview
- **Type Dropdown**: Select merge type (HISM or ISM, default HISM)
- **Merge Selected**: Execute merge operation on selected groups

### 2. Configuration Options (Below Toolbar)

#### Mobility Filter (Filter)
- ☑ **Static**: Include static objects
- ☑ **Stationary**: Include stationary objects
- ☐ **Movable**: Include movable objects
- Default: Static + Stationary checked

#### Min Instance Count
- Range: 0 - 1000
- Default: 2 (only show groups with at least 2 instances, at least 2 checkable instances required for merge)
- Purpose: Filter out groups with too few instances where optimization benefits are minimal

### 3. Dual-Panel Layout

#### Left Panel: Merge Groups Preview
Displays all qualifying merge groups, each containing:
- ☑ **Checkbox**: Select groups to participate in merge
- **Group Name**: Uses the first non-excluded instance's name (dynamically updated)
- **Instance Details**: Shows "Instance Count • Type • Mobility" (e.g., `5 Instances • HISM • Static`)

Top features:
- **Search Box**: Fuzzy search by instance name (tokenized + sequential matching)
- **Select All**: Batch check/uncheck currently displayed groups

#### Right Panel: Included Actors
Displays all instances in the selected left panel group:
- ☑ **Checkbox**: Check to include in merge, uncheck to exclude instance
- **Actor Name**: Instance label in the scene
- **[⊙] Locate Button**: Click to jump to the Actor in scene and select in Outliner

Top features:
- **Toggle All**: Batch check/uncheck all instances

---

## 🔍 Search Functionality

### Search Rules

Supports **tokenized fuzzy search** where keywords must appear in order:

**Example**: Search `SM 02`
- ✅ Match: `SM_Rock_02` ("SM" before "02")
- ✅ Match: `SM_000002` ("SM" before "02")
- ❌ No match: `02_SM_Rock` (wrong order)
- ❌ No match: `Rock_02` (missing "SM")

### Search Behavior

- **Display Only**: Doesn't modify data, clearing search box restores all groups
- **Group-Level Filter**: If any instance in a group matches, the entire group is shown (with all instances)
- **Merge Limitation**: Only merges currently displayed and checked groups

---

## ⚙️ Grouping Rules

The plugin automatically groups Actors with **identical properties** into the same group, including:

1. **Static Mesh**: The static mesh asset used
2. **Mobility**: Mobility type (Static/Stationary/Movable)
3. **Full Collision Settings**:
   - Collision enabled type (NoCollision/QueryOnly/PhysicsOnly, etc.)
   - Collision profile name
   - Collision object type
   - Responses for all 32 collision channels
4. **Materials**: Material list

**Only Actors with all identical properties will be merged into one group.**

---

## 🚀 Usage Workflow

### Basic Merge Workflow

1. **Open Tool**: `Window > Static Mesh Merger`
2. **(Optional) Break LevelInstances**: Click `Break Level Instances` button
3. **Configure Filter**:
   - Check desired mobility types
   - Set min instance count threshold
4. **View Preview**: Left panel displays all merge groups
5. **(Optional) Filter with Search**: Enter keywords in search box
6. **Check Groups**:
   - Click `Select All` to batch check
   - Or manually check groups to merge
7. **(Optional) Exclude Instances**:
   - Click group in left panel to view instance list on right
   - Uncheck instances you don't want to merge
8. **Execute Merge**: Click `Merge Selected` button
9. **View Results**: After merge completes, automatically selects newly created ISM/HISM Actors

### Advanced Tips

#### Precisely Control Merge Objects
1. Use search box to narrow scope (e.g., search `Tree` to only show tree-related groups)
2. Uncheck unwanted groups
3. Enter right panel to exclude individual instances
4. Click merge to only process currently displayed and checked groups

#### Locate Scene Objects
- Click `[⊙]` button in right instance list
- Editor viewport jumps to that Actor
- Outliner synchronizes selection

#### Auto Uncheck
When excluding instances causes checkable count < min instance count, the group automatically unchecks

---

## 📊 Instance Count Rules

### Display Rule
- Total Actor count >= Min instance count

### Checkable Rule
- Checkable Actor count >= Min instance count

### Merge Execution
- Actual participating Actor count >= Min instance count

**Example** (Min instance count = 2):
- Group has 5 instances
- Uncheck 3 instances
- 2 checkable instances remain
- **Can merge** (2 >= 2)

---

## 🏗️ Breaking LevelInstances

### Functionality

Clicking `Break Level Instances` recursively breaks all LevelInstances in the scene:

- ✅ Automatically handles nested LevelInstances
- ✅ Smart deduplication (duplicate instances with same path are deleted)
- ✅ Real-time progress notifications (flowing display of current item)
- ✅ UI auto-refreshes after each iteration

### Processing Strategy

| Scenario | Processing | Result |
|----------|------------|--------|
| First encounter of path | Add to cache → Break | Content moved to main scene |
| Same-level duplicate | Detect → Delete | Avoids duplicate content |
| Nested duplicate | Detect → Delete | Avoids circular references |

---

## ✅ Notes

### Merge Rules
- Only merges Actors in **main scene (PersistentLevel)**
- Does not include Actors inside LevelInstances (must break first)
- All properties must be identical to be grouped together

### Collision Preservation
- Completely copies collision profiles
- Accurately copies object type and all 32 channel responses
- Collision behavior after merge is identical to original Actors

### Min Instance Count
- Default 2: At least 2 instances required to merge
- Filters out groups with too few instances (minimal optimization benefit)
- Can adjust as needed (range 0-1000)

### Scene Monitoring
- Auto-monitors Actor additions/deletions (refreshes after 500ms delay)
- Map switches clear list (requires manual Refresh click)

### State Caching Mechanism
- **Excluded List Persistence**: Unchecked instances are recorded in the exclusion list
- **State Maintained on Refresh**: Clicking Refresh button or auto-refresh on scene changes **preserves** exclusion state
- **Window-Level Lifecycle**: All instances reset to default checked state only when closing and reopening the window
- **Use Case**: Can gradually adjust excluded instances, refresh multiple times to verify results without repeating operations

### After Merge
- Automatically selects all newly created ISM/HISM Actors
- Viewport jumps to first newly created Actor
- Original Actors are destroyed

---

## 🎯 Performance Optimization Tips

### Merge Strategy

1. **Prioritize Large Duplicates**: More instances = better optimization
2. **Batch Merge**: Use search to merge by category (e.g., merge trees first, then rocks)
3. **Preserve Necessary Grouping**: Don't merge objects that need individual control

### HISM vs ISM

- **HISM** (Default Recommended):
  - Supports LOD
  - Supports occlusion culling
  - Better performance for large-scale instances

- **ISM**:
  - Simpler and lighter
  - Lower overhead for small-scale instances

---

## 📮 Contact

For questions, suggestions, or technical support, please contact:
**mengzhishanghun@outlook.com**


 
