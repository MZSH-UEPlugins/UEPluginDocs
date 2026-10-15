[English](./README.md) | [中文](./README_CN.md)

# MCPBlueprint User Guide

MCPBlueprint is an editor-only MCP server for discovering and safely editing Unreal Engine Blueprints. This guide covers the **61 tools registered by the current module**. The live editor tools/list response is authoritative for the exact runtime schema.

## Requirements and installation

- Unreal Engine 5.2+ Editor on Windows, macOS, or Linux.
- An MCP client that supports a local HTTP server.
- A package matching the exact Unreal Engine minor version.

Install once per engine:

```text
<UE_5.x>/Engine/Plugins/Marketplace/MCPBlueprint/
```

MCPBlueprint is an optional engine plugin (Installed=true, EnabledByDefault=true). Do not copy it into every project or add it to a project's .uproject; restart the editor after replacing the package. The toolbar shows the local endpoint (normally http://127.0.0.1:8766/mcp; occupied ports advance up to nine times). Use your MCP client's own setup flow.

## Connect an MCP client

MCPBlueprint starts its local HTTP server with the editor when **Auto Start** is enabled. Give the client the endpoint shown by the plugin toolbar; do not hard-code the default when another process may already occupy that port. The server binds only to the local machine.

`initialize` and `tools/list` are normal client handshake calls. `ping` is a troubleshooting check, not a required manual step for every session. Browser-style requests may use only `localhost`, `127.0.0.1`, or `[::1]` as `Origin`; ordinary non-browser MCP clients can omit `Origin`.

The live `tools/list` response is the schema authority. Refresh it after replacing the plugin package or reconnecting the editor instead of relying on a cached tool list.

## Concept map: choose the right state

```text
Blueprint asset
├─ parent class (inheritance) ─────────── ReparentBlueprint
├─ Class Default Object (class defaults) ─ Get/SetClassDefaults
├─ member variable declaration
│  └─ variable default ────────────────── AddVariable / ModifyVariable
├─ function declaration
│  └─ signature pins ──────────────────── CreateFunction / ModifyFunctionSignature
├─ graph node
│  └─ graph-pin defaults ──────────────── SetPinDefaults / ApplyGraphPatch
└─ Actor Blueprint construction script (SCS)
   └─ component template properties ────── Get/SetComponentProperties
      └─ attachment parent ─────────────── AddComponent / ReparentComponent / SetRootComponent
```

- A variable default is a member declaration's initial value—not a node pin or CDO property.
- Function signature pins declare the function contract and affect callers. Graph-pin defaults are literals on one node instance.
- Component template properties are values on a local SCS component template. Its attachment parent is a hierarchy relationship, not the Blueprint parent class.
- Class defaults are editable CDO properties, not level-instance overrides.

## Recommended safe workflow

1. Discover paths and identities with ListBlueprints, GetBlueprintOverview, GetGraphDetail, or GetComponents.
2. Use bDryRun=true whenever supported and read every impact, blocker, and approval requirement.
3. Apply only returned paths, GUIDs, pin IDs, component names, and SpawnerId values—never guesses.
4. Read the changed state back, compile where relevant, then call SaveAsset only when persistence is desired.

Edits are not saved automatically. Use `SaveAsset` when you want to keep a change. Destructive tools require extra confirmation or refuse the operation when references could be broken.

## Complete feature matrix (61 tools)

The table is a quick guide. Live tools/list remains the source for the exact fields and constraints.

| Tool | What it does | Important notes |
|---|---|---|
| ListBlueprints | Lists Blueprint assets. | Filter and page large projects. |
| GetBlueprintOverview | Shows a Blueprint's parent, graphs, members, components, and interfaces, including member variable flags, Details metadata, and VarGuid. | `IsSet` fields distinguish absent metadata from an empty string. |
| GetBlueprintDetails | Reads implemented member-variable, CustomEvent, Function or ordinary local Macro declaration properties and editing capability. | Member variables use `Target.Kind: "MemberVariable"`, `Name`, and optional `VarGuid`; CustomEvents use `Target.Kind: "CustomEvent"` and `NodeGuid`; Functions use `Target.Kind: "Function"`, `Name`, and `FunctionGraphGuid` from `ListBlueprintMembers`. Macros use `Target.Kind: "Macro"`, `Name`, and `GraphGuid` returned by `GetGraphDetail`. `CoverageComplete: false` marks the current scope. |
| ListBlueprintMembers | Lists functions, events, dispatchers, and local variables. | Results are paged. |
| GetGraphDetail | Shows graph nodes, pins, links, defaults, and positions. | Use returned identifiers when editing a graph. |
| SearchGraphNodes | Finds available node actions. | Use only returned SpawnerId values. |
| GetCompileErrors | Compiles a Blueprint and shows diagnostics. | Optional warnings-as-errors mode. |
| CreateUserDefinedStruct | Creates a User Defined Struct under /Game. | New assets are dirty until saved. |
| GetUserDefinedStruct | Shows Struct fields and field identifiers. | Use the field identifier for an existing field. |
| ModifyUserDefinedStruct | Changes one Struct field operation. | Start with dry-run; removing or changing a type needs data-loss approval. |
| CreateUserDefinedEnum | Creates a User Defined Enum under /Game. | New assets are dirty until saved. |
| GetUserDefinedEnum | Shows Enum entries and serialized values. | Results are bounded. |
| ModifyUserDefinedEnum | Changes one Enum operation. | Start with dry-run; remove or move needs semantic-change approval. |
| AddVariable | Adds a Blueprint member variable. | Supports type, default, category, and instance-editable settings. |
| ModifyVariable | Changes a member variable and its Details metadata. | Unsupported references block type changes; supported constructor references require the applicable approvals. Remove RepNotify before renaming. |
| RemoveVariable | Removes a member variable. | References block removal; force affects only local references. |
| CreateMacro | Creates a macro with explicit exec or concrete data pins. | Requires a successful current compile; omitted signatures create no pins; use returned graph/tunnel GUIDs. |
| ModifyMacroDetails | Edits a local ordinary or native MacroLibrary macro's category, tooltip, keywords and compact title. | Uses `Target.Kind: "Macro"`, `Name`, and `GraphGuid`; preserves signatures and does not save automatically; library presentation edits preserve loaded authored-graph consumers. |
| CreateFunction | Creates a function graph with inputs and outputs. | Can create pure functions or overrides. |
| RenameFunction | Renames a declared function and safe loaded callers. | Dry-run by default; applying needs explicit reference approval and unsafe bindings block it. |
| ModifyFunctionSignature | Adds, renames, removes, reorders, or edits a function parameter type, flags, or input default. | Dry-run by default; changing/removing a parameter requires explicit approvals. |
| RemoveFunction | Removes a declared function graph. | References and unsafe unloaded descendants block removal. |
| CreateCustomEvent | Adds a Custom Event node. | Optional input parameters are supported. |
| ModifyCustomEvent | Changes editor-callable, deprecation, and RPC declaration properties of a CustomEvent. | Use `Target.Kind: "CustomEvent"` and `NodeGuid`. Supports local editable events; omitted fields keep their values. |
| ModifyFunctionDetails | Changes a local function's display name, tooltip, category, keywords, compact title, editor-call and deprecation details. | Use `Target.Kind: "Function"`, `Name`, and `FunctionGraphGuid` from `ListBlueprintMembers`; omitted fields keep their values. |
| AddEventNode | Adds an overridable engine event node. | Repeating the same event is safe. |
| AddBoundEvent | Adds a bound event for an Actor component or Level Blueprint Actor. | Not for UMG WidgetTree events; use MCPUMG there. |
| AddLocalVariable | Adds a variable scoped to one function. | The function must already exist. |
| ModifyLocalVariable | Edits one local declaration's name, type, default, category or tooltip and its exact Get/Set references, including collapsed graphs. | Copy its `Target` from `ListBlueprintMembers`; preview is the default. Apply requires reference approval, and supplying `TypeName` also requires data-loss approval. |
| RemoveLocalVariable | Removes an unused local variable. | Referenced locals cannot be removed. |
| AddNodePin | Adds optional pins to supported nodes. | Supports Sequence, container makers, and Switch nodes. |
| RemoveNodePin | Removes one optional node pin. | Required or non-removable pins are refused. |
| AddInterface | Adds a Blueprint Interface. | Repeating an implemented interface is safe. |
| AddEventDispatcher | Adds an Event Dispatcher. | Optional input signature; repeating it is safe. |
| RemoveEventDispatcher | Removes a declared Event Dispatcher. | References and unsafe unloaded descendants block removal. |
| ApplyGraphPatch | Applies one atomic graph edit. | Use legal SpawnerIds; start with dry-run; maximum 60 operations. |
| SetPinDefaults | Sets literal values on graph input pins. | Values use UE text; a connected pin keeps its connection. |
| FormatGraph | Plans or applies graph layout. | Supports dry-run, scope, style, and optional reroutes. |
| AddCommentBox | Adds a native comment around nodes. | Dry-run can preview bounds. |
| GetComponents | Shows an Actor Blueprint component hierarchy. | Works with the Actor Blueprint construction script. |
| AddComponent | Adds a component to an Actor Blueprint. | A parent attachment must be a scene component. |
| SetComponentProperties | Sets local component-template properties. | Uses editable non-array UE-text properties; requires an editor transaction. |
| RenameComponent | Renames a local component. | Updates Blueprint member references. |
| RemoveComponent | Removes a local component and promotes its children. | References and unchecked derived Blueprints block removal. |
| ReparentComponent | Moves a local SceneComponent below another SceneComponent. | Cannot create cycles; use SetRootComponent for an actual local root. |
| SetRootComponent | Makes a local SceneComponent the root. | Inherited and native roots cannot be replaced. |
| DuplicateComponent | Duplicates a local component and its template properties. | The actual local scene root cannot be duplicated. |
| GetComponentProperties | Shows editable component-template properties. | Values use UE text. |
| GetClassDefaults | Shows editable Blueprint Class Default Object properties. | Values use UE text. |
| SetClassDefaults | Sets supported Class Default Object properties. | Uses editable non-array UE-text properties; requires an editor transaction. |
| CreateBlueprint | Creates a Blueprint asset in memory. | Provide an asset path and parent class; optional BlueprintKind is Normal (default) or MacroLibrary with a legal native parent; save explicitly. |
| ReparentBlueprint | Changes a Blueprint's parent class. | Start with dry-run; hierarchy changes need potential-data-loss approval. |
| CompileBlueprint | Explicitly compiles a Blueprint. | Supports warnings-as-errors. |
| SaveAsset | Saves a dirty Blueprint, Struct, or Enum. | Saving is always explicit. |
| OpenAsset | Opens a Blueprint in the editor. | Required before capturing its graph. |
| CloseAsset | Closes a Blueprint editor. | Does nothing if it is already closed. |
| ReloadBlueprintFromDisk | Replaces a loaded Blueprint with its disk version. | Dry-run by default; applying discards unsaved changes and resets Undo/Redo. |
| DeleteAsset | Deletes a Blueprint asset. | Not undoable; referenced assets block deletion unless force is explicitly chosen. |
| RenameAsset | Renames or moves a Blueprint asset. | Redirectors preserve references; save at the new path. |
| DuplicateAsset | Duplicates a Blueprint asset in memory. | Save the new asset explicitly. |
| CaptureGraphScreenshot | Captures a PNG of an open Blueprint graph. | Choose one viewport mode; an initialized editor graph is required. |

## Focused write guides

### ApplyGraphPatch: one transactional graph change

Call SearchGraphNodes first and use only returned `SpawnerId` values. A patch can create nodes with temporary IDs, connect or disconnect pins, remove or move already-present nodes by stable GUID, set pin defaults, and apply layout. Pin references use a patch temporary ID for new nodes or `@<NodeGuid>` for already-present nodes. The combined operation limit is 60 across node creation, connections, disconnections, removals, and moves.

Start with `bDryRun=true`. Inspect normalized operations and every blocker; dry-run may explicitly say that Action Registry nodes, generated pins, type promotion, or exact post-layout state cannot be proven without mutation. Apply the same planned patch once. The write executes in one transaction and attempts automatic rollback on failure. If recovery cannot be proven, the response reports that explicitly; stop and read the graph back before any retry. Then call GetGraphDetail, compile, and explicitly save only when persistence is wanted. Use the default Auto layout for normal authored logic; use explicit positions or `LayoutScope=None` only for a real fixed-layout requirement.

Recovery responses from `ApplyGraphPatch`, `ModifyVariable`, `ModifyLocalVariable`, and `ModifyFunctionSignature` use `RecoveryContractVersion: 2` and remain errors. Check these fields before deciding what to do next:

| Field | Meaning |
|---|---|
| `bUndoSucceeded` | This call's own Undo succeeded. A later transaction from another operation is never undone. |
| `bContentRestored`, `bSnapshotRestored` | Declarations, graphs, nodes, and pins match their previous contents. `bRolledBack` additionally requires the owned Undo to succeed. |
| `RuntimeVerification` | `Verified` means runtime shapes and default values match; `Different` means they differ; `Unavailable` means a complete check was unavailable; `NotRequested` means no check was requested. `bRuntimeRestored` is true only for `Verified`. |
| `RecoveryAssets` | Each Blueprint's actual before/after Status, status code, and package Dirty state. Restored content may still be Dirty. |
| `bRollbackStateRestored` | Content, required runtime state, and native state all match. The tool never rewrites Status or clears Dirty to satisfy this field. |
| `RollbackRecoveryCompileReports` | Actual recovery compiler results when needed. Successful compilation still requires content and runtime comparisons. |
| `DefaultRepairResult`, `DefaultRepairProperties` | Result and property paths for restoring a specified variable default after recovery compilation. `Applied` still requires the final complete comparison. |
| `AuthoringDifference`, `RuntimeDifference`, `RecoveryError` | The location and cause of a difference or unavailable recovery proof. |

Stop editing and inspect the current assets when content/runtime differs or recovery cannot be proven. Dirty graphs and graphs with existing compiler errors remain editable. A Blueprint without the actual generated class/CDO needed by native Undo returns `RecoveryUnavailable`; the tool does not compile it implicitly before the edit.

### ModifyVariable: member-variable declarations and defaults

1. Call `GetBlueprintOverview` with `BlueprintPath` to find a variable **declared by this Blueprint** and its `VarGuid`. Then call `GetBlueprintDetails` with `Target.Kind: "MemberVariable"`, its `Name`, and optionally that `VarGuid`. `CoverageComplete: false` means the tool currently returns implemented declaration properties for supported target kinds; it does not yet cover all Blueprint Details functionality. Inspect `Properties` before editing: `Current` is the value, `IsSet` distinguishes an absent metadata key from an empty one, and `Editable`/`Reason` or `Enable`/`Disable` explain available changes. Within `Enable`/`Disable`, `Allowed` means the action can run, `NoChange` means the requested state is already current, and `Reason` explains a restriction. `Properties.NewName.Editable` describes the current declaration; a proposed new name still needs validation. Inherited variables can be read here, but `ModifyVariable` changes only declarations owned by the target Blueprint.
2. Call `ModifyVariable` with `BlueprintPath`, the existing `VariableName`, and only the fields you want to change. `DefaultValue` changes the UE-text **declaration default**; use `SetPinDefaults` for graph pin literals and `SetClassDefaults` for direct class-default edits. It does not retroactively replace existing instance overrides. A dirty or newly added declaration can receive flags, defaults, and metadata; rename and type changes require a successfully compiled baseline. Compile when relevant, then use `SaveAsset` with `BlueprintPath` only if you want to keep the change on disk.

These are complete tool arguments; replace `/Game/Blueprints/BP_Player` and the sample variable names with your own Blueprint path and declarations:

```json
{"BlueprintPath":"/Game/Blueprints/BP_Player"}
```

Pass that object to `GetBlueprintOverview`, then use `GetBlueprintDetails`:

```json
{"BlueprintPath":"/Game/Blueprints/BP_Player","Target":{"Kind":"MemberVariable","Name":"Health"}}
```

For an `int32` `Health` declaration, this `ModifyVariable` call sets a default, instance display, category, and SaveGame archive flag:

```json
{"BlueprintPath":"/Game/Blueprints/BP_Player","VariableName":"Health","DefaultValue":"100","bInstanceEditable":true,"bSaveGame":true,"Category":"Stats","Tooltip":"Current health"}
```

| Field | Applicable value or type | Effect |
|---|---|---|
| `BlueprintPath`, `VariableName` | Required strings; existing Blueprint and member name | Select the declaration. `VariableName` is the old name even when using `NewName`. |
| `TypeName`, `DefaultValue`, `NewName` | Type syntax accepted by `AddVariable`; UE-text default; nonempty new name | Change type, declaration default, or name. Supported constructor references can migrate to a new type with the required reference-update and data-loss approvals; unsupported references block type changes. RepNotify variables cannot be renamed here. Rename/type also reject dependencies that cannot be safely updated. |
| `bInstanceEditable`, `Category`, `Tooltip` | Boolean; category/tooltip strings | Control instance Details editing, category, and help text. Empty `Category` restores Unreal's default category; empty `Tooltip` removes its metadata. |
| `bSaveGame`, `bTransient`, `bAdvancedDisplay` | Booleans | Select properties for SaveGame archives, exclude them from persistent serialization, or move them to advanced Details. `bSaveGame` does **not** create a save system; `bTransient` is a serialization flag. |
| `bBlueprintReadOnly`, `bPrivate` | Booleans on owned member declarations | Restrict Blueprint Set usage or Blueprint access from other Blueprints. These are independent of `bInstanceEditable`; see the access example below. |
| `UIMin`, `UIMax`, `ClampMin`, `ClampMax` | Finite numeric strings on scalar `int32`, non-enum `byte`, `float`/`double`, or supported range structs | Set slider and clamp bounds. Lower bounds cannot exceed upper bounds, and slider/clamp intervals must overlap. |
| `ForceUnits` | Recognized unit name or abbreviation on scalar numeric variables | Changes the displayed unit without converting the default. UE 5.4+ stores the editor's full unit name; UE 5.2/5.3 store a recognized input. An old abbreviation may display as None after upgrading; set the unit again if needed. |
| `bMultiLine` | Boolean for `string`/`text`, including array/set elements and Map values | Select a multiline text editor. |
| `bBitmask`, `BitmaskEnum` | Boolean and explicit Enum object path; scalar `int32` only | Display a bitmask, optionally using an allowable Blueprint Enum with `Bitflags` metadata. A nonempty `BitmaskEnum` requires `bBitmask` enabled in the final state. |
| `bDeprecated`, `DeprecationMessage` | Boolean and string | Mark the declaration deprecated and show a message. A nonempty message requires `bDeprecated` enabled in the final state. |
| `bConfig` | Types allowed by native Config rules | Set the configuration-property flag without reading or writing ini files. Hard Object and Interface are disallowed; Class, SoftClass, SoftObject, and containers are checked against the actual type. |
| `bExposeToCinematics` | Properties animatable by the current Sequencer | Set Interp for Cinematics use; unavailable Sequencer capability returns a reason. |
| `bMakeEditWidget` | Vector, Transform, and their arrays; Instance Editable required | Provide a property editing widget in the editor viewport. |
| `bFieldNotify` | Members supported by the current Blueprint type and compiler | Set the FieldNotify declaration while preserving existing notification dependencies; disabling it can also remove other fields' dependencies on this field. |
| `bExposeOnSpawn` | Member types supported by native creation nodes | Provide a variable input on creation nodes; preview and approve consumer updates, with separate approval for default, split-field or connection losses. |

| Input | Result |
|---|---|
| Omit an optional field | Leave its current value or metadata unchanged. |
| Pass `false` to a Boolean flag | Clear that flag. `bPrivate: false` removes its metadata key even if the stored value is `"false"` or empty; check `IsSet` and `Disable` in Details. Clearing `bBitmask` or `bDeprecated` leaves the Enum or message until you clear it separately. |
| Pass `""` to `Category`, `Tooltip`, or a string metadata field | Restore the default category or remove the corresponding metadata key. `NewName: ""` does not rename; `DefaultValue` uses UE text and is not a metadata-clear command. |

For a numeric variable, a range edit is another complete `ModifyVariable` request:

```json
{"BlueprintPath":"/Game/Blueprints/BP_Player","VariableName":"Health","UIMin":"0","UIMax":"100","ClampMin":"0","ClampMax":"100"}
```

For a scalar `int32` `AbilityFlags`, use `{"BlueprintPath":"/Game/Blueprints/BP_Player","VariableName":"AbilityFlags","bBitmask":true}`. If you add `BitmaskEnum`, supply the actual object path of your Blueprint-usable `Bitflags` Enum. Bitmask edits rebuild only supported local getter/setter nodes; external references, derived Blueprints, or unsupported local nodes can block the edit. Apply bitmask changes separately from rename or type changes.

For an access-restriction example, first inspect `Properties.bBlueprintReadOnly.Enable` and `Properties.bPrivate.Enable` in Details. If both are allowed, call `ModifyVariable`:

```json
{"BlueprintPath":"/Game/Blueprints/BP_Player","VariableName":"Health","bBlueprintReadOnly":true,"bPrivate":true}
```

ReadOnly keeps Blueprint reads but blocks variable Set nodes; it is **not deep immutability**. A mutable reference consumer can also block enabling it, while some engine-version container operations may still compile with warnings. Enabling ReadOnly removes `MakeEditWidget` metadata. Private restricts Blueprint reads and writes outside the declaring Blueprint; existing external references block it. Unresolved references or unloaded derived Blueprints can block either restriction. A missing inherited-variable action in `SearchGraphNodes` does not prove Private access is safe. To remove these restrictions, use `{"BlueprintPath":"/Game/Blueprints/BP_Player","VariableName":"Health","bBlueprintReadOnly":false,"bPrivate":false}`.

Move any default value you still need to a replacement variable **before** setting `bDeprecated: true`: Unreal does not serialize deprecated property values, so reopening the Blueprint may reset them. To retain an edit, call `SaveAsset` with `{"BlueprintPath":"/Game/Blueprints/BP_Player"}`.

The five editor flags accept JSON `true`/`false` only. Omission preserves the current state; same-state requests report no change without creating a transaction. Edits apply to owned member declarations; inherited and native members can be queried. Inspect direction availability with `GetBlueprintDetails`:

```json
{"BlueprintPath":"/Game/Blueprints/BP_Player","Target":{"Kind":"MemberVariable","Name":"Health"}}
```

Each flag in `Properties` contains `Current` (the declaration), `IsSet` (equal to the declaration for the first four flags; metadata-key presence for ExposeOnSpawn), `Effective` (the current compiled product, or `null` when unknown), `EffectiveReason`, `Editable`, and `Enable`/`Disable` with `Allowed`, `NoChange`, and `Reason`. `GetBlueprintOverview` also returns the five declaration booleans. A pending compilation or native Undo/Redo can leave `Current` different from `Effective`; Undo/Redo does not guarantee automatic recompilation.

For a Vector `HandleOffset`, enable instance editing and the viewport widget together:

```json
{"BlueprintPath":"/Game/Blueprints/BP_Player","VariableName":"HandleOffset","bInstanceEditable":true,"bMakeEditWidget":true}
```

Setting `bInstanceEditable:false` or `bBlueprintReadOnly:true` clears `MakeEditWidget`; the response lists this implicit edit. Check whether the final combination is allowed before applying it.

`bExposeOnSpawn` controls the variable input on creation nodes. Preview the change before approving consumer updates:

```json
{"BlueprintPath":"/Game/Blueprints/BP_Player","VariableName":"Health","bExposeOnSpawn":true,"bInstanceEditable":true,"bDryRun":true}
```

Inspect the nodes, pin changes and `Losses` in `EditorPropertyImpact.Consumers`. Remove `bDryRun` and pass `bApproveReferenceUpdates:true` to update consumers. Removing pins with defaults or connections, or changing to an incompatible type, also requires `bAllowPotentialDataLoss:true`. Compatible changes preserve pin identity, connections and explicit defaults. An approved incompatible-default loss uses the new type's native automatic default. Changing a declaration default preserves loaded derived-class overrides. Actor/Object creation nodes usually also require Instance Editable; legacy AddComponent nodes follow their native rules.

Using `NewName` on an exposed variable lists `EditorPropertyImpact.RenameImpact` and creation-node changes. Candidate-name conflicts, unsupported references, unloaded derived classes or unknown rename handlers block the edit. Resolve the stated restriction and preview again.

Undo or redo the rename, then compile normally to preserve loaded derived-class overrides. Later edits to derived defaults are not overwritten by the rename. This behavior applies while the plugin remains loaded; reloading it does not preserve the guarantee for earlier undo history.
Preview FieldNotify edits that affect consumer Blueprints. For an existing Widget `Health` member:

```json
{"BlueprintPath":"/Game/UI/WBP_Player","VariableName":"Health","bFieldNotify":true,"bDryRun":true}
```

`ModifyVariable` defaults to `bDryRun:false`; preview neither writes nor compiles. Applying consumer updates requires `bApproveReferenceUpdates:true`; removing notification dependencies also requires `bAllowPotentialDataLoss:true`. With `bAutoCompileAfterModify` enabled, the tool compiles affected Blueprints. With it disabled, the declaration can differ from the current `Effective` value and requires explicit compilation. The tool does not save automatically; call `SaveAsset` to persist changes.

UE 5.2 Widgets cannot enable FieldNotify on ReadOnly members. UE 5.2 MVVM also requires an actually registered dedicated compiler; loaded modules or types alone are insufficient for generation. UE 5.3+ checks the notification interface and current compiled product. Unloaded related assets or unconfirmed references return concrete blockers. Related UE 5.8 MVVM conversions support only confirmed static native BlueprintPure Float functions, forward OneTime bindings, one Manual context, and single-property input paths. Other scalar, instance/getter, split-pin, Node, or multipath conversions return a limitation reason.

### ModifyFunctionSignature: function signature pins

Address the declaration with FunctionName plus stable FunctionGraphGuid, and a declared parameter only by stable PinId. Submit exactly one operation. Dry-run is default; apply requires reference approval, and remove/type/flag changes also require potential-data-loss approval. The tool scans callers, rejects unsafe override/interface/RepNotify/delegate/AnimGraph cases, updates ordinary calls transactionally, compiles affected Blueprints, and never saves automatically.

Use `Operation="MovePin"` to reorder a parameter within its input or output group. Get its `PinId` from the function Entry or a Result in `GetGraphDetail`, then provide `NewIndex`, the final zero-based position in that group. For inputs `[A, B, Depth]`, moving `A` to `NewIndex=2` produces `[B, Depth, A]`. Exec and self pins do not count toward this index. `ListBlueprintMembers` displays parameters alphabetically; its `DeclarationIndex` gives their actual signature positions.

For example, copy the Blueprint path, function graph GUID, and declared pin ID into this request:

```json
{
  "BlueprintPath": "/Game/Blueprints/BP_Example",
  "FunctionName": "Evaluate",
  "FunctionGraphGuid": "<function graph GUID>",
  "Operation": "MovePin",
  "PinId": "<Entry or Result pin GUID>",
  "NewIndex": 2
}
```

Review the default dry-run, then send the same request with `bDryRun=false` and `bApproveReferenceUpdates=true`. Moving a parameter to its current position makes no change. `MovePin` accepts no rename, type, default, or parameter-flag edits and needs no potential-data-loss approval. It preserves parameter identities, types, defaults, flags, and connections, updates ordinary same-Blueprint, recursive, and supported external calls, and moves a selected output across all matching Result declarations. Each Result keeps its own literal and pin identity; inconsistent Result signatures reject. Use normal Undo/Redo to undo or reapply a move, and save the affected assets when ready. For `MovePin`, start from the original base function declaration. Loaded and compiled ordinary Blueprint descendants are supported, including function-graph overrides, descendants that only inherit the function, ordinary calls, and parent-function calls. Editing an override directly is unsupported. A CreateDelegate binding to the target function or any of its overrides blocks the move; stable bindings to other functions keep their selected function and connections. Other signature operations retain their existing derived/override restrictions.

Use `Operation="SetPinFlags"` to edit the declaration's reference and const flags. Supply both `bIsReference` and `bIsConst` as booleans. Select an input with its Entry `PinId`, or an output with any Result `PinId`. Outputs are matched by name across all Results, retaining each Result's own pin identity.

```json
{
  "BlueprintPath": "/Game/Blueprints/BP_Calculator",
  "FunctionName": "Evaluate",
  "FunctionGraphGuid": "<function graph GUID>",
  "Operation": "SetPinFlags",
  "PinId": "<Entry or Result pin GUID>",
  "bIsReference": true,
  "bIsConst": true
}
```

Review the default dry-run, then send the same request with `bDryRun=false`, `bApproveReferenceUpdates=true`, and `bAllowPotentialDataLoss=true`. A flag change clears the selected parameter's declaration default. When you save, Unreal can synchronize that declaration with the pin's current default, such as "0" for an integer output. If both flags already match, no default is cleared and no transaction is created; apply still requires those approvals. Approval does not permit removing caller overrides or connections. Mutable reference inputs need a valid variable connection; a literal alone cannot compile. The tool rejects or rolls back an uncompilable edit without creating variables or rewiring callers. A request with `bIsConst=false` is rejected when the native declaration requires const.

`SetPinFlags` accepts no `PinName`, `TypeName`, `NewIndex`, or `DefaultValue`; other operations accept neither flags field. Arrays can remain implicitly referenced with raw `bIsReference=false`, and an output's raw reference flag does not guarantee a compiled reference flag. Use normal Undo/Redo after a successful edit, and save the affected assets when ready.

Use `Operation="SetPinDefault"` to change an input parameter's declaration default. Get its `PinId` from the function Entry and explicitly provide `DefaultValue` as a string, including an empty string. Use `SetPinDefaults` for an individual Result node's output literal.

```json
{
  "BlueprintPath": "/Game/Blueprints/BP_Example",
  "FunctionName": "Evaluate",
  "FunctionGraphGuid": "<GraphGuid returned by GetGraphDetail>",
  "Operation": "SetPinDefault",
  "PinId": "<input parameter PinId from Entry>",
  "DefaultValue": "17"
}
```

Review the dry-run, then add `bDryRun=false` and `bApproveReferenceUpdates=true`. This operation needs no `bAllowPotentialDataLoss`. An identical declaration string with a consistent compiled baseline creates no transaction. Calls inheriting the old default receive the new value; explicit overrides remain intact. Wired inputs retain their links and still obtain their runtime values from those links. Save the affected assets when ready; read back Unreal's native representation after parsing and saving empty or struct values.

`SetPinDefault` accepts no `PinName`, `TypeName`, `NewIndex`, `bIsReference`, or `bIsConst`. It requires an ordinary non-reference input and rejects containers, compiled effective references, hard object/class/interface types, soft Actor references, and exec pins. Loaded native CreateDelegate bindings must pass a read-only proof that their signatures, selectors, and connections remain stable. A binding that would repair or clear its selector rejects before mutation. Existing derived, override, and incomplete-reference safety checks still apply. A current verifiable compile baseline is required. If a related Blueprint needs compilation after Undo/Redo, compile it normally before submitting another default edit.
### SetPinDefaults: graph pin defaults

Use a returned NodeGuid and a nonempty PinDefaults object mapping pin names to UE text literals. This changes one node's input literal—not the function signature, variable default, or CDO. Read back that graph and compile. Invalid, non-input, or read-only pins reject. A connected input keeps its connection, so the stored literal is ignored at runtime.

### SetComponentProperties and ReparentComponent

SetComponentProperties writes editable non-array **local SCS component-template** values from a nonempty UE-text Properties map. Read with GetComponentProperties first, then reread and compile. ReparentComponent changes the **attachment parent**; both sides must be local SceneComponents and cycles reject. An actual local root must use SetRootComponent. Neither changes the Blueprint parent class.

### GetComponentProperties: inherited SCS templates

`GetComponents` returns a separate, independently paginated `InheritedSCSComponents` list. Each row identifies the declaring `DeclaringBlueprintPath`, `ComponentGuid`, component `Name`, effective `TemplatePath`, and `SourceBlueprintPath` (`TemplateSource` is `Declaration` or `Override`). Copy that declaration identity when reading a child Blueprint:

```json
{"BlueprintPath":"/Game/MyActorChild","ComponentName":"InheritedRoot","DeclaringBlueprintPath":"/Game/MyActorParent","ComponentGuid":"11111111-2222-3333-4444-555555555555"}
```

Use the actual GUID returned by `GetComponents`. Both optional identity fields must be supplied together; without them, `GetComponentProperties` continues to read local SCS templates. Inherited reads require an existing, successfully compiled target and consistent ancestor chain. The effective value comes from the nearest existing override or the original declaration; the response names its source. Missing generated state, ambiguous GUIDs, or inconsistent owners/templates return a reason and require separate repair. Queries do not create CDOs, handlers or overrides. These inherited identities are for reading; `SetComponentProperties` continues to accept local SCS components only.

### SetClassDefaults: CDO defaults

Read GetClassDefaults, inspect the returned UE-text property map, apply supported editable non-array Properties, reread, compile, then save. These are class defaults—not instance overrides, graph literals, variable declarations, or component templates.

### ReparentBlueprint: parent class

`NewParentClassPath` accepts a native class path, a unique loaded class name, or another Blueprint class. Preview the change first. If the new parent changes the class hierarchy, the tool asks for explicit data-loss approval. A parent that cannot compile is rejected and the problem is reported.

## Edit function-local variables

1. Find the local with `ListBlueprintMembers` and copy its complete `Target`. `FunctionGraphGuid` identifies the function graph and `VarGuid` identifies the declaration. If supplied, `Name` must exactly match its current name.
2. Supply `BlueprintPath`, the copied `Target`, and at least one of `NewName`, `TypeName`, `DefaultValue`, `Category`, or `Tooltip`. For example, `DefaultValue: ""` previews clearing the default; `NewName: "TemporaryScore"` together with `Category: "Calculation"` previews a rename and category change.
3. `bDryRun` defaults to `true`. The preview lists references and potentially removed links. `MaxResults` accepts 1–100, defaults to 50, and limits displayed rows only.
4. Apply the same request with explicit `bDryRun: false` and `bApproveReferenceUpdates: true`. Whenever `TypeName` is supplied, also set `bAllowPotentialDataLoss: true`. Call `SaveAsset` to keep the change on disk.

Omitted fields keep their values. An actual type change clears an omitted default; supplying the same type preserves it. Empty Category restores the engine default category, and empty Tooltip removes its metadata. Same-name members and locals in other functions keep their identities. A new name conflicting with a direct child's member is refused.

Nonempty defaults support numbers, booleans, enums, Name, strings, safe mathematical structs, and Array/Set/Map values whose elements are equally safe. Top-level hard object/class references must already be resident and satisfy path, type, and the actual function entry's access rules. Private references across packages are refused; top-level soft references follow their type rules. Nonempty Text, nested object/soft references, object-bearing structs, interface/delegate values, and unaudited native structs are refused. An empty default restores the type's initialized value. Name, category, and tooltip edits do not reimport an unchanged default.

## Tutorial: create a calculation function

This example creates:

```text
FinalScore = (BaseScore + Bonus) × Multiplier
```

1. Use `CreateBlueprint` to create an Actor Blueprint.
2. Use `CreateFunction` to add `CalculateScore` with three `double` inputs and one `double` output.
3. Use `GetGraphDetail` to find the new function's Entry and Result node identifiers.
4. Search this function graph for `Add` and `Multiply`, then use the returned `Operator:Add` and `Operator:Multiply` actions.
5. Preview an `ApplyGraphPatch` containing the two nodes and five data connections.
6. Apply the same patch with native `double` promotion and automatic `Balanced` layout.
7. Call `SaveAsset` if you want to keep the change.

The finished graph adds `BaseScore` and `Bonus`, then multiplies the result by `Multiplier` from left to right.

## Visual examples

The examples below show Blueprint functions, events, graphs, and common logic changes. Use them as a visual guide alongside the tool workflow described above.

### Rename a function

`RenameFunction` previews the impact before changing anything. Review the preview, approve reference updates when requested, then use `SaveAsset` if you want to keep the renamed function.

![Renamed function](./Images/Tutorial/function-rename-base.png)

![Updated function caller](./Images/Tutorial/function-rename-caller.png)

### Function, event, and graph showcase

![Function signature and connected implementation](./Images/Tutorial/function-signature-overview.png)

![Component-event helper graph](./Images/Tutorial/component-event-graph.png)

![Actor lifecycle, bound component event, and variable-pin flow](./Images/Tutorial/event-graph-overview.png)

### Create a calculation function

`CalculateScore` adds `BaseScore` and `Bonus`, then multiplies the result by `Multiplier`.

![CalculateScore graph](./Images/Tutorial/calculate-score.png)

### Build a larger workflow

`EvaluateOrderBatchV2` keeps order totals, discounts, shipping, tax, points, risk scoring, and the final result in one connected function graph.

![Order settlement graph](./Images/Tutorial/order-settlement-complex.png)

### Add free shipping for first orders

Before the change, Region 1 always sets `ShippingCents = 1000`.

![Region 1 before](./Images/Tutorial/order-region1-before.png)

After the change, a `FirstOrder` branch sets shipping to `0` for first orders and keeps `1000` for other orders.

![Region 1 after](./Images/Tutorial/order-region1-after.png)

### Exclude shipping from points

Before the change: `PointsEarned = TotalCents / 100`.

![Points before](./Images/Tutorial/points-before.png)

After the change: `PointsEarned = (TotalCents - ShippingCents) / 100`.

![Points after](./Images/Tutorial/points-after.png)

### Replace a fixed discount with 5%

Before the change, the graph always returns a fixed discount of `500`.

![Discount before](./Images/Tutorial/discount-before.png)

After the change, `SubtotalCents / 20` produces a 5% discount.

![Discount after](./Images/Tutorial/discount-after.png)

### Connect an Event Dispatcher

Declare `OnSignal` with an input such as `Payload:int` using `AddEventDispatcher`. Call `SearchGraphNodes` with the target `BlueprintPath`, `GraphName`, and `Query="OnSignal"` to obtain the actual actions and pins.

- `Bind` adds a receiving event.
- `Call` broadcasts the inputs to every bound receiver.
- `Unbind` removes a specific receiver.
- `UnbindAll` removes all receivers from that dispatcher.

Create a receiver with a matching input signature using `CreateCustomEvent`, then create the searched actions with `ApplyGraphPatch`. Connect the receiver's `@<NodeGuid>.OutputDelegate` to the Bind or Unbind `Delegate` pin. Connect the initiating event's `then` to Bind's `execute`, and Bind's `then` to Call's `execute`. Supply Call's `Payload` through `PinDefaults` or a data connection. Read `GetGraphDetail` to verify the wiring and compile result, then call `SaveAsset`.

These four actions support scalar dispatchers declared in the target Blueprint itself. Their `Delegate:<operation>:self:<declarationGUID>` IDs survive compilation and saving. Deleting and recreating the same name produces a new GUID: old IDs are rejected, so search again. Assign, CreateDelegate, inherited or cross-Blueprint targets, and dispatcher signature or rename migration remain unsupported.

### Change local function presentation and editor properties

`ListBlueprintMembers` returns function `Inputs` and `Outputs` in alphabetical order. Each function pin also includes `DeclarationIndex` (the zero-based position in its own entry or result terminal declaration array), `bIsReference` and `bIsConst`. The index is local to each terminal, not a unique sequence across the whole function. `bIsReference` and `bIsConst` are raw declaration flags. When `bCompiledFlagsKnown=true`, `bCompiledReference`, `bCompiledConst`, and `bCompiledOutParm` describe the current owned compiled function. Unavailable metadata returns only `bCompiledFlagsKnown=false`, without replacing unknown flags with false. The query never compiles; compile the Blueprint normally before reading when needed. Reference inputs can also carry OutParm while remaining logical inputs. Existing pagination, 16-pin signature limits and default-value truncation remain in effect. CustomEvent and EventDispatcher results retain their existing fields.

Call `ListBlueprintMembers` with `MemberKind: "Function"`. Copy the function's `Name` and `FunctionGraphGuid` into a `Target` with `Kind: "Function"`; `NodeGuid` identifies the entry node and cannot replace the graph GUID. Use that target with `GetBlueprintDetails` to inspect supported values and editing restrictions, then call `ModifyFunctionDetails`:

```json
{"BlueprintPath":"/Game/Blueprints/BP_Player","Target":{"Kind":"Function","Name":"SendSignal","FunctionGraphGuid":"<FunctionGraphGuid from ListBlueprintMembers>"},"DisplayName":"Send Signal","Tooltip":"Sends the current payload.","Category":"Signals","Keywords":"payload emit","CompactTitle":"SEND","bCallInEditor":true,"bDeprecated":false,"DeprecationMessage":""}
```

`DisplayName` changes call-node titles; `Tooltip` describes the function; `Category` groups it in menus; `Keywords` helps action search; `CompactTitle` selects a compact call-node title. `bCallInEditor` exposes an editor-callable declaration and does not execute the function. `bDeprecated` marks usage deprecated, and `DeprecationMessage` supplies guidance while that flag is enabled. Use `SearchGraphNodes` with the new title, category, or keywords to find the call action.

Omit a field to preserve it. Empty text clears it; an empty `Category` resets to Unreal's default category. Category, keywords, and compact title trim surrounding whitespace. Clearing `CompactTitle` preserves keywords and restores the ordinary call-node title. Deprecated functions appear in action search only when the Blueprint Editor setting to expose deprecated functions (`bExposeDeprecatedFunctions`) is enabled. The tool compiles the Blueprint; use `SaveAsset` to keep the changes. Requests with no effective change are rejected. Override/interface implementations, RepNotify, construction and other protected functions are read-only. This tool does not change purity, constness, access, signatures, or RPC declarations.

### Change CustomEvent properties and RPC declarations

Get the event's `NodeGuid` from `CreateCustomEvent` or `GetGraphDetail`. Use the same `Target` with `GetBlueprintDetails` and `ModifyCustomEvent`. For example, declare a local event as a reliable server RPC:

```json
{"BlueprintPath":"/Game/Blueprints/BP_Player","Target":{"Kind":"CustomEvent","NodeGuid":"<event NodeGuid>"},"Replication":"Server","bReliable":true}
```

`Replication` accepts `None`, `Server`, `Client`, or `Multicast`. `None` disables RPC routing while preserving the stored reliable bit when `bReliable` is omitted; that bit has no effect outside RPC mode. Explicitly setting or clearing `bReliable` requires a final RPC mode, which can be supplied in the same request. Details reports the saved value in `Current` and active routing/reliability in `Effective`.

`bCallInEditor` controls the editor-callable declaration; the tool does not execute the event. `bDeprecated` marks usage deprecated, and `DeprecationMessage` stores guidance; an empty string clears the message. Booleans must be JSON `true`/`false`. Omitted fields stay unchanged, and same-state requests report no change. Only local editable events in a successfully compiled Blueprint are supported; overrides, interface implementations, and dispatcher-bound receivers are refused. Call `SaveAsset` to persist changes. These settings do not change Actor replication: multiplayer use still requires Actor replication, ownership, and the correct calling endpoint.

## Capture, settings, limits, and troubleshooting

Call `OpenAsset` before `CaptureGraphScreenshot`. You can capture the full graph, focus on one node, or reuse a specific viewport. Check that titles, pins, and connections are readable before sharing the image.

In **Project Settings → Plugins → MCP Blueprint**, configure Port, Auto Start, Auto Compile After Modify, request timeout, and compile timeout.

- The plugin is editor-only; there is no generic arbitrary Blueprint runtime-invocation tool.
- Custom Event and Event Dispatcher signature mutation is less complete than ordinary function signatures.
- Some unsafe actions are refused instead of being partially applied.
- Compilation is not persistence: save explicitly.
- If a tool is missing, refresh `tools/list`. If an edit is refused, read the reason before trying again.
- If UE asks to add this installed engine plugin to a project descriptor, choose **Not Now**; updating creates the project dependency this installation model avoids.

## Support

For questions or feedback, use the [contact page](https://mengzhishanghun.github.io/mengzhishanghun/contact/).

### Read Blueprint class settings

For an ordinary Blueprint, call `GetBlueprintDetails` with `Target: {"Kind":"ClassSettings"}` to read the asset's raw `DisplayName`, `Description` and `Category`, including empty strings. This target accepts only `Kind`; do not include member names or GUIDs. The fields return `Current`, `Editable: false` and a `Reason` stating that this tool does not currently expose class-setting changes. `CoverageComplete` remains false. This reads the Blueprint declaration, without compiling it or creating a CDO; it does not read instance defaults.

### Create and use macros

`ModifyMacroDetails` edits the native input-tunnel metadata of a local macro in a successfully compiled ordinary Blueprint. Use `GetGraphDetail` to obtain the macro's `GraphGuid`, then read `GetBlueprintDetails` with `Target: {"Kind":"Macro","Name":"Relay","GraphGuid":"<returned GUID>"}`. The response exposes raw `Current`, `Editable`, `Reason` and `Impact` for each supported property.

Pass any changed `Category`, `Tooltip`, `Keywords` or `CompactTitle` string to `ModifyMacroDetails` with that same target. Omitted fields retain their values; `Category`, `Keywords` and `CompactTitle` are trimmed. Empty category restores Unreal's default category; empty tooltip, keywords or compact title clears only that field. Clearing compact title retains independent keywords. The action search reads native category, keywords and tooltip while preserving the name-based `SpawnerId`; an empty raw tooltip can display Unreal's macro-instance fallback text. Category changes are refused if a same-name skeleton or inherited function exists. Unknown fields, null values, stale identities and requests with no actual change are rejected. For ordinary Blueprints, the tool preserves existing graph identities, pins and links and compiles the edit. Signature/name changes are outside this tool's scope. Save assets explicitly.

Native MacroLibraries use the same target and four fields. `GetBlueprintDetails` reports their raw values and editing capability. A library must have a valid GeneratedClass and an UpToDate, Warning or Dirty status; missing classes, Error/Unknown states and compilation in progress are refused before writing. Explicitly compile a new library if its class is unavailable. Presentation edits preserve the library's existing status and the content, status and dirty flags of consumers in loaded authored asset graphs, including nested graphs. Unsupported Blueprint subclasses, custom macro-instance classes and unresolved related references are refused. Unloaded consumer packages are not loaded for this operation.

Library presentation edits do not request full Blueprint compilation. Native editor Undo/Redo may regenerate skeletons and leave the library Dirty; you can continue editing presentation fields in that state. `bCompilationRequired`, `AfterStatus` and `bPackageDirty` report the asset's actual state. If an operation fails, `bContentRestored` and `bRollbackStateRestored` distinguish restored content from restored editor state; a successful content rollback can retain Dirty. The response remains an error. Use `SaveAsset` explicitly to persist the library; consumers are not modified or saved by this metadata operation.

`CreateBlueprint` accepts optional `BlueprintKind: "MacroLibrary"`; omission keeps the ordinary Blueprint behavior. Macro libraries currently require a legal native parent, such as `/Script/Engine.Actor` or `/Script/CoreUObject.Object`. A Blueprint-generated parent is rejected.

Compile the new asset with `CompileBlueprint` before `CreateMacro`. Creation requires a current successful compile and preserves existing graphs and links. Define execution pins explicitly with `TypeName: "exec"`; omitting `Inputs` or `Outputs` creates no implicit execution pins. Data pins use the existing concrete type syntax; wildcard, by-reference and const signatures are unsupported.

```json
{
  "BlueprintPath": "/Game/Blueprints/BP_MacroLibrary",
  "MacroName": "Relay",
  "Inputs": [{"Name": "Enter", "TypeName": "exec"}, {"Name": "Value", "TypeName": "int"}],
  "Outputs": [{"Name": "Leave", "TypeName": "exec"}, {"Name": "Result", "TypeName": "int"}]
}
```

The response includes `GraphGuid`, `EntryNodeGuid`, `ExitNodeGuid` and actual pin details. Inputs appear as output pins on the entry tunnel; outputs appear as input pins on the exit tunnel. Connect the macro body through `ApplyGraphPatch`. `GetGraphDetail` also returns `GraphGuid`.

Use `SearchGraphNodes` in the intended consumer graph to obtain the exact macro `SpawnerId`, then place and connect it with `ApplyGraphPatch` and compile the consumer. Ordinary Blueprint macros are local; macro-library availability follows its native parent and Unreal's action filtering. Self-recursion and latent macro use in a function are rejected by the existing graph-context checks. Editing existing macro signatures, renaming or deleting macros is outside this creation tool. Save the library and consumer explicitly with `SaveAsset`.
