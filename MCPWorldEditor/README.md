[English](./README.md) | [中文](./README_CN.md)

# MCPWorldEditor

AI-driven level/world editing plugin for Unreal Engine, powered by MCP (Model Context Protocol).

## Overview

MCPWorldEditor exposes level editing capabilities to AI assistants via an HTTP server running inside the Unreal Editor. AI tools can spawn actors, modify transforms, edit properties, manage lighting, landscape, and capture viewport — all through a standardized protocol.

> **Note**: This plugin is currently under development. The server infrastructure is ready, and tool capabilities continue to improve.

## Configuration

Project Settings > Plugins > MCP World Editor:

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| Port | int32 | 8764 | HTTP port (auto-increment if occupied, up to 10 retries) |
| bAutoStart | bool | true | Auto-start server on editor launch |
| RequestTimeoutSeconds | int32 | 30 | Tool execution timeout |
| MaxRequestBodyBytes | int32 | 1048576 | Maximum HTTP request body accepted before UTF-8/JSON parsing |
| MaxResponseBodyBytes | int32 | 8388608 | Maximum UTF-8 JSON response; screenshot Base64 content shares this budget |
| MaxCapturePixels | int32 | 8294400 | Maximum pixels read by `CaptureViewport` (approximately 4K by default) |

`CaptureViewport` normalizes viewport alpha to fully opaque before PNG encoding, so saved files and embedded screenshots display correctly in standard image viewers and Fab workflows.

## MCP Connection

The plugin runs an HTTP server inside the Unreal Editor. AI clients connect via MCP protocol.

**MCP URL**: `http://127.0.0.1:8764/mcp`

The server starts automatically when the editor opens (configurable via `bAutoStart`). The toolbar shows the actual URL (port auto-increments if 8764 is occupied).

### Claude Code

Add to `~/.claude/.mcp.json` (global) or project root `.mcp.json`:

```json
{
  "mcpServers": {
    "ue-world-editor": {
      "url": "http://127.0.0.1:8764/mcp"
    }
  }
}
```

### Other AI Clients

MCP client configuration varies by tool (Cursor, Windsurf, VS Code Copilot, etc.). The server URL is the same — consult your AI tool's documentation for how to add an MCP server.

A tool request that times out before the Game Thread claims it is cancelled and may be retried after the editor becomes available. If the Game Thread already claimed it, the error content reports `code=timeout_execution_in_progress`, `retryable=false`, and `stateMayHaveChanged=true`; the operation may still finish, so clients must inspect editor state instead of retrying blindly.

The response-size check runs after tool execution. If a `tools/call` result exceeds the budget, JSON-RPC error data reports `code=response_too_large`, `retryable=false`, and `stateMayHaveChanged=true`; this does not mean the tool was rolled back, so clients must inspect editor state before any retry.

Before a request reaches the Game Thread, the server validates top-level required fields, exact JSON types, array item types, array `maxItems`, and unknown fields against the selected tool's published `inputSchema`; every tool schema defaults to `additionalProperties=false`, and current batch arrays publish their 100-item limit. Tool-specific business rules such as asset paths, numeric ranges, and object existence remain a second validation layer.

The editor becomes busy as soon as a PIE start request is queued. `PIEControl/GetState` and `GetEditorState` distinguish `Idle`, `StartQueued`, and `Playing`, while `Stop` can cancel a queued start. Viewport type and render-mode tools use only a known level-editor viewport instead of down-casting an asset-preview viewport; when both ViewMode and ShowFlag are supplied, both are validated before either change is applied.

`ListActors` and `GetSelectedActors` return `ActorPath`, `FName`, `Label`, and `LevelPath`. Every tool that uses `ActorName/ActorNames` to address existing actors accepts the full `ActorPath` as the preferred exact identifier. Legacy labels and FNames execute only when they uniquely identify one actor in the current world; ambiguous input is rejected with sorted candidate paths instead of selecting, moving, editing, or deleting the wrong actor. Refresh discovery after an actor is renamed or moved to another level because its path changes.

## Requirements

- Unreal Engine 5.2+
- Editor only

## Tool and Editor State Contracts

### Saving, map checks, and builds

`GetDirtyPackages` lists loaded dirty packages in `CurrentLevel` or `AllLoaded` scope. `CurrentLevel` includes loaded external object packages and a dirty separate Map BuiltData package for the current level. `SavePackages` takes exact `PackageNames` and defaults to `DryRun=true`: preview the selection, then pass its `PlanHash` with `DryRun=false`. Saving an empty package may delete its existing file, so it requires `AllowDeleteEmptyPackages`. A local-only save rejects actual writes while a source-control provider is enabled. Results report each package and any remaining dirty packages. The parameterless `SaveLevel` still saves the current level and the external object packages included by the engine, and now reports the actual level package path and remaining dirty packages. Disk saves are outside editor Undo.

`MapCheck` runs the engine map check and returns error and warning counts plus visible messages with tokens. If the Message Log UI filter hides messages or `MaxMessages` truncates them, `Complete=false`, so the result cannot be reported as passing. The check creates a new page for the current map while retaining older pages and the user's UI filter.

`BuildNavigation` and `BuildLighting` default to previews; execution requires the corresponding `PlanHash` and `DryRun=false`. In regular and World Partition maps, navigation uses the engine's public synchronous `UNavigationSystemV1::Build` for registered, loaded visible editor levels (`Scope=LoadedVisibleEditorWorld`). Loaded hidden and unloaded levels are excluded; the preview lists loaded hidden paths in `ExcludedLoadedHiddenLevelPaths`. Changing level visibility invalidates the preview hash, and a level in a visibility transition must settle before preview or execution. Navigation preserves editor Undo and does not implicitly run `MapCheck` or save packages. In World Partition it may discard loaded navigation chunks; `bCompleteWorldPartitionBuild=false` and `bSaved=false` do not claim a full offline partitioned-world build. Lighting uses Preview quality and returns an `OperationId`; query `GetBuildStatus` for its terminal state. A lighting build succeeds only after all active lighting scenarios have applied their results. Neither build invents a progress percentage or saves the map. Switching or unloading the map invalidates running operation IDs.

`FindNavigationPath` takes world-space `Start` and `End` positions and synchronously queries the currently loaded default navigation data. It returns whether a path was found, whether it is partial, its length, and a bounded list of points without starting a build. `MaxPoints` accepts 2–1000; longer paths set `bTruncated=true`.

Use MCP `tools/list` to discover the tools and their current input schemas; the available set can change with the plugin version.

### Foliage and instanced meshes

Use `ListFoliageTypes` to find loaded foliage types and their exact paths. `AddFoliageInstances` adds a static mesh with `MeshPath` and either world-space `Transforms` or legacy `Locations` (identity rotation and scale). For existing loaded foliage, call `FoliageInstances` with `Operation=Query`, exact `IFAPath` and `FoliageTypePath`; page with `Offset` and `Limit` (up to 100). To `Update`, `Remove`, or `Move` selected `Indices`, first preview with the default `bDryRun=true`, then repeat the same request with `bDryRun=false`, `ExpectedCurrentSnapshotHash`, and `ExpectedPlanHash`. `Update` requires one transform per index. `Move` also needs a distinct loaded `TargetIFAPath` and `ExpectedTargetSnapshotHash`; in World Partition, both the source and destination foliage partitions must be loaded and the locations must belong to the selected partitions. Indices only identify instances in their matching snapshot.

`ScatterFoliage` previews deterministic candidates for an existing loaded foliage type on traced WorldStatic ground. Supply the exact IFA/type paths, XY rectangle (`MinX`, `MinY`, `MaxX`, `MaxY`), accepted Z range, `Seed`, `MaxSlopeDegrees`, and `MinSpacing`, plus either `Count` (at most 100) or `Density`; `MaxSamples` bounds the search. Review the preview and pass both returned hashes to execute. For non-foliage ISM/HISM components, `InstancedMeshes` provides paged `Query` and previewed `Add`, `Update`, and `Remove` using exact `ComponentPath`, world `Transforms`, and snapshot indices. It rejects foliage components; use `FoliageInstances` for those.

`ConvertInstancedMeshes` converts simple `StaticMeshActor` objects to grouped ISM/HISM instances (`Direction=ActorsToInstances`, exact `ActorPaths`, optional `bUseHISM`) or selected non-foliage component instances back to actors (`Direction=InstancesToActors`, exact `ComponentPath` and `Indices`). Preview first, then apply with `ExpectedCurrentSnapshotHash` and `ExpectedPlanHash`. Sources remain by default. With `bDeleteSource=true`, known external loaded actor references, component referencers, and level-script references refuse actor deletion, including when `bAllowLossyConversion=true`. The preview and result report `ReferenceCoverageComplete=false`, `ReferenceCoverage`, and `UninspectableReferenceSources`: native `FGCObject` owners behind `GCObjectReferencer` cannot be fully inspected, and unloaded partitions and project-wide soft references are unscanned. Native hard references are not migrated; deleting sources may break their semantics. Actor deletion additionally requires `bAllowLossyConversion=true` and fresh preview hashes. The plan hash includes the reference policy and observed coverage, but cannot prove that opaque native references stayed unchanged. Conversion creates and verifies targets before deleting sources, and cannot preserve actor identity and all actor-specific or custom instance data.

### Actor placement and splines

`PlaceActors` uses `Mode=Ground`, `Surface`, `Grid`, `Bounds`, or `Vertex` with `ActorNames` (up to 100 exact paths or unique names) to reposition existing actors. Ground and Surface trace downward and place the transformed bounds bottom on the hit; `bAlignNormal` optionally aligns Surface local +Z. Grid rounds world pivot coordinates using `GridSize`. Bounds places actors successively beside `ReferenceActor` using `Axis`, `Side`, and `Gap`. Vertex snaps to a vertex of `ReferenceActor`'s specified static-mesh `ReferenceComponent` and `LOD` within `SearchRadius` and `MaxVertices`. `Mode=Array` instead duplicates `SourceActor` `Count` times (1–100); `Spacing`, optional per-axis variation, and `Seed` control the resulting world transforms. Review the returned final transforms before saving.

For an ordinary spline component, call `GetSplineData` with `ActorName`, exact `ComponentName`, and optional `CoordinateSpace=Local|World`. Pass its `SnapshotHash` to `ApplySplinePatch` with an ordered `Operations` array (`Add`, `Remove`, `SetLocation`, `SetTangents`, `SetRotation`, `SetScale`, `SetType`, or `SetClosedLoop`). Preview first; execution needs `bDryRun=false` and the preview `PlanHash`. Insertion and deletion shift later indices in the same patch. `SetSplineMesh` edits one spline-mesh component with local `Start`, `StartTangent`, `End`, and `EndTangent`; optional scales, roll, forward axis, and static mesh can be supplied. Landscape splines use the separate tools below.

`ApplySplinePatch` supports Blueprint SCS spline components and verifies the current component again after construction reruns. It rejects splines already modified by a construction script or configured to pass their points to that script. A reconstruction or readback failure may retain changes in the Undo transaction; inspect the reported state before retrying.

### World Partition Data Layers and Level Instances

In a World Partition level, use `GetWorldPartitionInfo` to find Data Layers, `CreateDataLayer` to create one, and `SetDataLayer` to add/remove actors. UE 5.2 requires an existing `DataLayerAssetPath` when creating; later versions create a private layer. Batch `SetDataLayer` calls with `ActorNames` default to a preview and require the fresh `ExpectedHash` to apply. `ManageDataLayer` takes exact `DataLayerName` and `Action=Query|Rename|SetParent|SetVisible|SetLoaded|SetInitialRuntimeState|Delete`. `SetParent` uses `ParentName` (empty clears the parent); visibility/loading use `bValue`; initial runtime state is `Unloaded`, `Loaded`, or `Activated`. Preview `Delete` and pass its `StateHash` as `ExpectedHash` with `bDryRun=false`. Asset-backed layers cannot be renamed independently.

`ManageLevelInstance` uses `Action=Query` to list instances. `Create` takes `ActorNames` and a new `/Game/...` `LevelPackagePath`; preview first and apply with `ExpectedHash`. Creation saves the new target level before creating the instance, but the source level remains dirty: `SourceSaved=false` and `PendingSavePackages` identify the remaining save. Before removing source actors, the tool checks known loaded hard references; unloaded and soft references are not scanned. Review `SoftReferenceRisk`, set `bAcknowledgeUnscannedSoftReferences=true` in the preview and execution, and use the hash produced with that acknowledgement. Save the reported source package separately. A failed create may leave a target map or partial source changes, so inspect the reported state before retrying.

Use `EnterEdit` on an exact `InstanceActor`; its `OperationId` owns the session and is required for `MoveActors`, `Commit`, or `Discard`. `MoveActors` previews an actor batch and needs a fresh `ExpectedHash` to apply. `Break` expands an instance back into actors. `PackBlueprint` creates or updates a Packed Level Actor Blueprint at exact `/Game/...` `BlueprintPath`; `bPendingSave=true` means the Blueprint still needs saving. These actions do not imply that the source level or every resulting asset has been saved.

Packed Blueprint creation runs without an asset dialog and does not save automatically. In UE 5.3 and later, the engine packing stage resets the global editor Undo history. Undo any edits you want to revert before packing. `PendingSavePackages` lists potentially affected packages, including loaded instances. A failed creation may retain the new in-memory asset; the error reports the stage and packages to inspect.

### Landscape paint layers and Landscape splines

`GetLandscapeLayerInfo` lists the main Landscape's paint-layer bindings. `CreateLandscapeLayerInfo` previews a new `/Game/...` asset with `PackagePath`, `LayerName`, and `bWeightBlended`, then creates it with `bDryRun=false` and `ExpectedPlanHash`; the asset remains unsaved until saved separately. `BindLandscapeLayerInfo` previews binding an existing `LayerInfoPath` to `LayerName` on a loaded non-partitioned main Landscape, or unbinding with `bUnbind=true`. Apply with its fresh `ExpectedPlanHash`; unbinding refuses painted allocations and never deletes the asset.

`GetLandscapeWeightData` reads an inclusive vertex region (up to 65,536 row-major `uint8` weights) from an exact bound `LayerInfoPath`; specify `EditLayerGuid` when Edit Layers are enabled. `SetLandscapeWeightData` accepts either exact `WeightData` or brush fields (`BrushCenterX/Y`, `BrushRadius`, `BrushTargetWeight`, `BrushStrength`). It requires a loaded non-partitioned main Landscape and an existing target allocation on every affected component. Preview the bounded region, then apply with both `ExpectedCurrentDataHash` and `ExpectedProposedDataHash`; weight-blended layers can normalize other paint weights. `GetLandscapeEditLayers` lists Edit Layer GUIDs; `EditLandscapeLayer` previews `Create`, `Delete`, `Reorder`, `Rename`, visibility, lock, alpha, or blend-mode changes and requires its fresh current and plan hashes to apply. Deletion requires `bAllowDataLoss=true`; UE 5.8 does not expose the requested edit-layer blend-mode setter.

For Landscape-specific splines, inspect `GetLandscapeSplineTopology`, then use `PatchLandscapeSplineTopology` for `AddPoint`, `UpdatePoint`, `DeletePoint`, `AddSegment`, `UpdateSegment`, or `DeleteSegment`, with exact point/segment paths and local coordinates or tangents as appropriate. Preview first and apply with `ExpectedTopologyHash` and `ExpectedPlanHash`. This bounded topology editor requires a loaded non-partitioned Landscape without a reserved spline Edit Layer. `ApplyLandscapeSplines` separately previews terrain application to an explicit `EditLayerGuid` and applies with `ExpectedPlanHash`; when a reserved spline layer exists, use its GUID. The target must be available and unlocked. Neither operation saves the level or assets.

### Landscape Height Data Safety

`GetLandscapeHeightData` and `SetLandscapeHeightData` exchange row-major `uint16` height samples directly in JSON; they never probe or read caller-selected files. Regions use inclusive Landscape vertex coordinates and are limited to 65,536 samples per call. The target must be the main `ALandscape` actor, and the complete region must be loaded and owned by that actor. UE 5.8 and newer require an exact GUID for a standard `EditLayerGuid`; procedural and spline layers are rejected, and `SetLandscapeHeightData` also rejects locked layers while the read tool may inspect them. UE 5.2-5.7 reject landscapes that contain Edit Layers because those versions need a separate layer-aware contract.

`SetLandscapeHeightData` defaults to `bDryRun=true`. Preflight returns `CurrentDataHash` and `ProposedDataHash` without changing editor state. Applying requires `bDryRun=false`, the exact fresh `ExpectedCurrentDataHash`, and the matching `ExpectedProposedDataHash`; stale state or changed proposed samples are rejected before the transaction. A write is read back and verified. If verification fails, the original samples are restored and verified before the transaction is cancelled; an unverified rollback is reported as potentially state-changing. The atomic boundary is one region call, not a multi-call whole-landscape import, and neither tool saves the level or assets.

`RemoveStreamingLevel` is not itself undoable. It requests that Unreal Engine preserve the existing Undo buffer, but stale references may still cause the engine to reset that buffer, so Undo-history preservation is not guaranteed. The tool removes Actor, Component, Object, and BSP selections that belong to the target level. It snapshots and verifies restoration of selected Actors, Components, and Objects outside that level; BSP selections in other levels are left untouched rather than cleared globally. If removal fails, the tool also attempts to restore the target-level selection and reports whether restoration was verified.

`MoveActorsToLevel` attempts to restore and verify the pre-call Actor, Component, Object, and BSP selection, and returns an error when complete restoration cannot be verified. It rejects the request before moving actors when Selection Lock is enabled or actor identity cannot be mapped uniquely. If the batch result is inconsistent, the tool does not issue an automatic Undo because it cannot prove that the top transaction belongs to this call; inspect both actor locations and selection state before continuing.

## World Partition Region Loading

`LoadRegion` creates and loads a persistent editor region through Unreal Engine's public World Partition loader API. `BoundsMin` and `BoundsMax` must each contain finite numeric `X`, `Y`, and `Z` fields, and every minimum coordinate must be strictly less than its matching maximum coordinate. Invalid bounds are rejected before a loader is created.

## Material Instance Creation

`CreateMaterialInstance` accepts a writable long package name under the project `/Game/` content root, such as `/Game/Materials/MI_Wall`. It rejects Developers and generated ExternalActors/ExternalObjects directories, invalid package names, and destinations that already exist in memory or on disk. The parent may be any loadable `MaterialInterface`, including a material instance.

## Actor Grouping

`GroupActors` resolves and validates the complete actor list before making changes, rejects duplicate or cross-level grouping targets, and calls Unreal Engine's actor-array grouping APIs directly. It does not clear or replace the editor selection to drive the operation, so selections of surviving actors are preserved. If an explicitly selected GroupActor is disbanded and destroyed, that removed object naturally disappears from the selection.

## Actor Attachment Transactions

`AttachActor` and `DetachActor` include both actors and the scene components that actually change hierarchy in the editor transaction, so Undo/Redo restores the component hierarchy, socket, and transform. If the engine rejects an operation or the resulting parent state does not match the request, the tool attempts to restore the original parent component, socket, and transform before returning an error. `Original attachment restored` reports whether that restoration was verified; when it is `false`, treat editor state as potentially changed and inspect it before continuing.

## Component Creation Safety

`AddComponent` accepts an exact full component class path or a short name that uniquely matches a currently loaded class; ambiguous short names are rejected with sorted full-path candidates. It rejects non-`ActorComponent`, abstract, deprecated, superseded, and `ClassWithin`-incompatible classes. Instance components follow the Unreal Editor creation order for serialization, creation notification, and registration; attachment, root assignment, or registration failure cleans up the new component and cancels the transaction.

## World Settings Editing

`GetWorldSettings` returns exact names and UE text values for every property currently editable on the instance. It no longer hides default-valued properties or presents read-only BlueprintVisible fields as writable discovery results. `SetWorldSettings` requires exact case, rejects property flags or `CanEditChange` conditions that disable editing, and pairs `PreEditChange` with `PostEditChangeProperty` inside the transaction. An import failure first restores the old value; if restoration also fails, the error reports the transaction rollback result, and a failed rollback means editor state may have changed.

## World Partition Region Lifetime

`LoadRegion` accepts only finite, strictly increasing bounds. Each axis is limited to 1,000,000 Unreal units, total volume to 1e17 cubic units, and no more than 16 MCP-created regions may remain active. A successful call returns an opaque `RegionHandle`; pass it to `UnloadRegion` to unload and release the corresponding official Editor Loader Adapter. Handles are never guessed or reused from bounds. World cleanup/map replacement and plugin module shutdown automatically release regions still owned by the plugin, after which their old handles expire.

## Restricted Console Commands

`ExecuteConsoleCommand` accepts only the seven documented `stat` commands, routes them through `GEngine->Exec`, and checks whether the engine actually handled the request; an unhandled command returns an error. Unreal `stat` commands toggle their viewport statistic display, so repeating one may turn the display off; they are not side-effect-free measurements. `Handled=true` in a successful result only means that the current engine instance accepted the command, not that the display is necessarily enabled. Some commands produce no text output.

## Material Instance Asset Persistence

`CreateMaterialInstance` accepts only a valid new package path under the project's `/Game/...` root and rejects Developers plus generated ExternalActors/ExternalObjects directories. It rejects packages already present in memory or on disk, then saves the created asset through the official `EditorAssetSubsystem`. Success requires the exact expected object path, a discoverable on-disk package, and a clean package, and returns `Saved=true`. On type, path, or persistence failure, cleanup is force-deleted only when the returned object exactly matches this call's target, and succeeds only when the target object, in-memory package, and on-disk package are all absent afterward. An unexpected object is never blindly deleted, and the error warns that partial state may remain. Asset creation itself is not a normal Undo transaction; however, failure cleanup uses Unreal Force Delete and may clear the editor's existing Undo history.

For questions or feedback, email [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com). I will take care of it when I see your message.
