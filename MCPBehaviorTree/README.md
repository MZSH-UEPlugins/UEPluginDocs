[English](./README.md) | [中文](./README_CN.md)

# MCP Behavior Tree

MCP Behavior Tree is a self-contained Unreal Editor plugin for creating, inspecting, editing, validating, and saving Behavior Tree and Blackboard assets through the Model Context Protocol. It supports Unreal Engine 5.2 and later and does not depend on another user plugin.

## Connection

The plugin starts an MCP HTTP server in the Unreal Editor. Its default port is `8767`; when that port is unavailable, use the actual port reported by the running plugin instead of assuming the default.

Asset arguments use Unreal content paths such as `/Game/AI/BT_Enemy` and `/Game/AI/BB_Enemy`. Newly created or edited assets remain dirty until `SaveAsset` succeeds.

`SaveAsset`, `OpenAsset`, and `CloseAsset` only accept Behavior Tree or Blackboard assets. Saving fails closed when the editor graph cannot be synchronized or Unreal cannot write the package; a successful MCP result therefore means the package save succeeded.

## Tools

| Area | Tools |
|---|---|
| Assets | `CreateBehaviorTree`, `CreateBlackboard`, `SetBehaviorTreeBlackboard`, `ListBehaviorTrees`, `GetBehaviorTreeDetail`, `SaveAsset`, `OpenAsset`, `CloseAsset` |
| Blackboard | `AddBlackboardKey`, `RemoveBlackboardKey` |
| Tree structure | `AddCompositeNode`, `AddTaskNode`, `ConnectNodes`, `RemoveNode` |
| Nodes | `SetNodeProperties`, `AddDecorator`, `AddService` |
| Verification | `ValidateBehaviorTree` |

`GetBehaviorTreeDetail` returns stable `NodeId` values for editing calls, the linked Blackboard and its keys, nested tree structure, decorators, services, reflected editable properties, and floating nodes. Large results are bounded by depth, node, property, and text limits and report `bTruncated` when incomplete. `ListBehaviorTrees` supports deterministic sorting, filtering, and `Offset`/`MaxResults` paging.

## Recommended workflow

1. Create a Blackboard with `CreateBlackboard`.
2. Add keys with `AddBlackboardKey`.
3. Create a tree with `CreateBehaviorTree`, passing the Blackboard path.
4. Add a composite under `Root`, then add tasks, decorators, and services.
5. Use `GetBehaviorTreeDetail` to discover node IDs and editable properties.
6. Run `ValidateBehaviorTree`.
7. Call `SaveAsset` for each dirty Behavior Tree or Blackboard asset.

## Blackboard key rules

Supported key types are `Bool`, `Int`, `Float`, `Vector`, `Rotator`, `Name`, `String`, `Object`, `Class`, and `Enum`.

- `Object` and `Class` keys may specify `BaseClass`.
- `Enum` keys require `Enum`; `BaseClass` and `Enum` are rejected for unrelated key types.
- `KeyName` cannot be empty or `None`.
- A key selector is checked with Unreal's native filter semantics, including Object/Class base-class restrictions and Enum identity. A same-named key of the wrong type is invalid.
- `RemoveBlackboardKey` scans project Behavior Trees and refuses unresolved references or scan failures unless `bForce=true` explicitly accepts the risk.

## Editing behavior

Editing tools use Unreal transactions and mark affected packages dirty. Node properties use Unreal import-text strings; for example, booleans use `true`/`false`, enums use their names, and vectors use `(X=0.0,Y=0.0,Z=0.0)`. Property maps accept string values only, and a failed import leaves the property's previous value unchanged. Blackboard selector properties take a Blackboard key name and are validated against the selector's native type filters.

`SimpleParallel` has `Default` and `Background` output pins. Use `ParentOutputPin="Background"` when adding or moving the background subtree. Run validation before saving; validation reports structural errors, missing classes, invalid key bindings, invalid decorator abort modes, and unconnected nodes.

## Unreal Engine 5.8 comparison

Unreal Engine 5.8's Epic `AIModuleToolset` exposes seven read-only Behavior Tree helpers: get the Blackboard, root decorators, node list, one/all node depths, children, and a referenced subtree. MCP Behavior Tree covers the Blackboard, decorators, hierarchy, children, and depth through `GetBehaviorTreeDetail`, and additionally provides creation and editing operations.

The Epic toolset is Experimental and tied to UE 5.8's Toolset/MCP infrastructure. This plugin cannot depend on it while remaining self-contained and compatible with UE 5.2+. Any official-only behavior not available through the plugin's documented tools must be queried in UE 5.8 with the Epic toolset rather than assumed to exist on earlier engines.

## Limits

- The plugin is editor-only and does not provide runtime or packaged-game control.
- Blueprint task, decorator, and service logic is not authored here. Create and compile those Blueprint classes with a Blueprint-capable tool, then pass the generated `_C` class path.
- Private Behavior Tree editor graph APIs vary by engine version. The plugin contains compatibility paths for UE 5.2+, but projects should still validate assets in their target editor version.
- A Behavior Tree that has runtime nodes but no editor graph is rejected instead of being rebuilt as an empty graph. Repair that asset in the Behavior Tree editor before using MCP editing or save tools.
- Source changes do not imply that an existing packaged plugin has been rebuilt or published.

For questions or feedback, email [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com). I will take care of it when I see your message.
