[English](./README.md) | [中文](./README_CN.md)

# MCPBlueprint User Guide

MCPBlueprint exposes Blueprint discovery, graph editing, members, components, asset lifecycle operations, compile feedback, and graph capture to MCP-compatible AI clients. The live `tools/list` response is always authoritative for the exact tool and parameter surface in the current editor session.

## Requirements and installation

- Unreal Engine 5.2+ Editor on Windows, macOS, or Linux.
- An MCP client that supports local HTTP servers.
- The package must match the exact Unreal Engine minor version.

Install MCPBlueprint once for each engine:

```text
<UE_5.x>/Engine/Plugins/Marketplace/MCPBlueprint/
```

MCPBlueprint is an engine-level optional editor tool. Its descriptor uses `Installed=true` and `EnabledByDefault=true`. Do not copy it into every project and do not add it to a user project's `.uproject`. After replacing an installed package, restart that engine's editor.

## Connect your AI client

MCPBlueprint is enabled automatically and starts its local MCP server when the editor starts. Users do not need to enable it in each project or add it to a project's `.uproject`.

Ask your AI client to configure MCPBlueprint through that client's own MCP setup flow. Give it the endpoint shown by the plugin toolbar; the default is `http://127.0.0.1:8766/mcp`, and MCPBlueprint tries up to nine higher ports when the configured port is occupied. Configuration formats differ between AI clients, so this guide does not prescribe a client-specific JSON file or manual port procedure.

The MCP client normally performs `initialize` and `tools/list` automatically. `ping` is useful for troubleshooting, not a routine manual step for users.

The server binds to the local machine. Browser-style `Origin` headers are accepted only for `localhost`, `127.0.0.1`, and `[::1]`; ordinary non-browser MCP clients may omit `Origin`.

## What is complete today

The current release registers 55 tools:

| Capability | Count | Tools |
|---|---:|---|
| Discovery and reading | 6 | `ListBlueprints`, `GetBlueprintOverview`, `ListBlueprintMembers`, `GetGraphDetail`, `SearchGraphNodes`, `GetCompileErrors` |
| User Defined Struct and Enum | 6 | `CreateUserDefinedStruct`, `GetUserDefinedStruct`, `ModifyUserDefinedStruct`, `CreateUserDefinedEnum`, `GetUserDefinedEnum`, `ModifyUserDefinedEnum` |
| Variables, functions, events, and pins | 17 | `AddVariable`, `ModifyVariable`, `RemoveVariable`, `CreateFunction`, `RenameFunction`, `ModifyFunctionSignature`, `RemoveFunction`, `CreateCustomEvent`, `AddEventNode`, `AddBoundEvent`, `AddLocalVariable`, `RemoveLocalVariable`, `AddNodePin`, `RemoveNodePin`, `AddInterface`, `AddEventDispatcher`, `RemoveEventDispatcher` |
| Graph editing and layout | 4 | `ApplyGraphPatch`, `SetPinDefaults`, `FormatGraph`, `AddCommentBox` |
| Components and defaults | 11 | `GetComponents`, `AddComponent`, `SetComponentProperties`, `RenameComponent`, `RemoveComponent`, `ReparentComponent`, `SetRootComponent`, `DuplicateComponent`, `GetComponentProperties`, `GetClassDefaults`, `SetClassDefaults` |
| Asset lifecycle and capture | 11 | `CreateBlueprint`, `ReparentBlueprint`, `CompileBlueprint`, `SaveAsset`, `OpenAsset`, `CloseAsset`, `ReloadBlueprintFromDisk`, `DeleteAsset`, `RenameAsset`, `DuplicateAsset`, `CaptureGraphScreenshot` |

Completed behavior includes:

- Stable bounded pagination for large asset, member, graph, and pin reads.
- Registry-backed node IDs instead of invented node types.
- Atomic graph patches covering create, connect, disconnect, remove, move, defaults, and layout.
- `Balanced`, `Straight`, and `Compact` deterministic layout styles.
- Native Comment Box containment protection for local and whole-graph layout.
- Dry-run impact analysis and explicit approval gates for high-risk edits.
- Compile/read-back verification, transaction rollback, and single-step Undo where Unreal supports it.
- Explicit save semantics: write tools mark assets dirty but do not silently save.
- Engine-level automatic loading without a project-side MCPBlueprint declaration.

## Recommended workflow

Use the same sequence for every write:

1. **Inspect** — identify the exact asset, graph, stable GUID, pin, member, or component.
2. **Search** — obtain legal node `SpawnerId` values from `SearchGraphNodes`.
3. **Preview** — use `bDryRun=true` when supported and review every affected asset, normalized operation, limitation, and blocker.
4. **Apply** — send the reviewed operation once; never invent GUIDs or `SpawnerId` values.
5. **Read back** — query the graph or asset again and compare the intended state.
6. **Compile** — inspect authoritative status, errors, and warnings.
7. **Save explicitly** — call `SaveAsset` only when the user wants the change persisted.

A dry-run may report that Action Registry nodes, generated pins, type promotion, or exact post-patch layout cannot be proven without mutation. That is an expected limitation, not automatic write approval: verify the current graph and the exact `SearchGraphNodes` result before applying.

## Tutorial: build a calculation function

The following example creates:

```text
FinalScore = (BaseScore + Bonus) × Multiplier
```

1. Create an Actor Blueprint with `CreateBlueprint`.
2. Create `CalculateScore` with three `double` inputs and one `double` output using `CreateFunction`.
3. Read the new function with `GetGraphDetail` and record the entry/result node GUIDs.
4. Search `Add` and `Multiply` in that exact graph. Use the returned `Operator:Add` and `Operator:Multiply` IDs.
5. Preview one `ApplyGraphPatch` containing both nodes and all five data connections.
6. Apply the same patch with native `double` promotion and automatic `Balanced` layout.
7. Read back all four nodes and six links, compile, and save.

The resulting graph should be one connected left-to-right component with zero overlap, zero backward edges, and no invented node IDs.

## Real editor screenshots and business-rule comparisons

These images were captured from the actual UE 5.2 Blueprint Editor window. They do not use the off-screen graph PNG path that caused missing Slate brushes and black/white square artifacts.

### Normal graph example

The persisted `CalculateScore` tutorial function contains registry-backed Add and Multiply nodes, complete data/exec connections, successful compilation, and automatic layout.

![CalculateScore in the normal Blueprint Editor](./Images/Tutorial/calculate-score.png)

### Complex production-style workflow

`EvaluateOrderBatchV2` is a 71-node order-settlement function. One connected graph accumulates order lines, applies customer-tier and first-order discounts, selects regional shipping, calculates tax, assigns a review decision, computes points and risk, and assembles the final settlement result.

![Complete order-settlement workflow in the normal Blueprint Editor](./Images/Tutorial/order-settlement-complex.png)

### Change 1: Region 1 adds first-order free shipping

Before: Region 1 goes directly to `ShippingCents = 1000`; the first-order branch is not part of the execution path.

![Before adding first-order free shipping to Region 1](./Images/Tutorial/order-region1-before.png)

After: Region 1 enters the `FirstOrder` branch. First orders set `ShippingCents = 0`; other orders keep the original `ShippingCents = 1000` rule.

![After adding first-order free shipping to Region 1](./Images/Tutorial/order-region1-after.png)

### Change 2: points exclude shipping

Before: `PointsEarned = TotalCents / 100`.

![Before excluding shipping from points](./Images/Tutorial/points-before.png)

After: `PointsEarned = (TotalCents - ShippingCents) / 100`. The same viewport shows the newly inserted Subtract node and the preserved Divide and Result nodes.

![After excluding shipping from points](./Images/Tutorial/points-after.png)

### Change 3: first-order discount changes from fixed value to 5% of subtotal

Before: the function passes the fixed discount through a typed Add-with-zero baseline.

![Before changing the first-order discount rule](./Images/Tutorial/discount-before.png)

After: the fixed-value path is replaced by `SubtotalCents / 20`, producing a 5% discount while keeping the same function output and graph frame.

![After changing the first-order discount rule](./Images/Tutorial/discount-after.png)

All three comparisons show real node, default-value, or connection changes rather than layout-only edits. The compared states use fixed viewports, stable node identities, graph read-back, successful compilation, and explicit save. When dry-run cannot prove generated-pin state without mutation, the existing GUIDs and links are inspected before the single transaction and verified again afterward. These are controlled production-style business examples, not customer project data.

## Discover assets and graphs

- Use `ListBlueprints` to find assets with bounded pagination.
- Use `GetBlueprintOverview` for graphs, variables, functions, components, interfaces, and parent class.
- Use `ListBlueprintMembers` for functions, Custom Events, dispatchers, and local variables.
- Use `GetGraphDetail` before and after graph edits to obtain GUIDs, pins, defaults, links, and positions.
- Use `GetCompileErrors` or `CompileBlueprint` for authoritative diagnostics.

Always pass exact asset paths returned by the plugin. Read large graphs in pages instead of inferring structure from screenshots.

## Add graph logic safely

Search Unreal's English action name, for example `Branch`, `For Loop`, `For Each Loop`, `Switch on Int`, `Switch on Name`, `Switch on String`, `Add`, `Subtract`, `Multiply`, or `Divide`.

`ApplyGraphPatch` accepts:

- `Nodes` with temporary IDs and registry-backed `SpawnerId` values.
- `Connections` and `Disconnections`.
- `RemoveNodes` by stable node GUID.
- `MoveNodes` by stable node GUID and absolute graph position.
- Optional `PinDefaults`, `PromotedType`, layout scope, and layout style.

The entire patch is one transaction. Failure rolls back the requested graph mutation instead of leaving a partial graph.

## Members, functions, Structs, and Enums

- Variable tools manage type, default, category, tooltip, and safe removal.
- Function tools create, rename, change supported signature operations, or safely remove user functions.
- Local-variable actions are scoped to both the function graph and declaration GUID.
- Struct and Enum modifications address stable fields/entries and report reference impact.
- Potential data loss or Enum value-semantic changes require separate explicit approval.

Function rename and signature mutation default to impact analysis. They reject unsupported overrides, incomplete reference scans, opaque delegate bindings, unloaded derived classes, and other states that cannot be restored safely.

## Components and class defaults

Read the current component hierarchy before adding, renaming, duplicating, reparenting, setting the root, changing template properties, or removing a component. Class-default tools read or update supported Blueprint defaults without pretending they are instance values.

## Asset lifecycle

Asset tools create, duplicate, rename, reparent, compile, open, close, reload, delete, and save supported Blueprint assets.

- Compilation does not imply persistence.
- Disk reload defaults to dry-run and requires explicit approval before discarding in-memory state.
- Destructive deletion has stricter Undo/Redo and package-safety requirements than ordinary graph edits.
- Never report success until read-back and disk/Asset Registry checks agree.

## Capture a graph screenshot

Call `OpenAsset` first, then use `CaptureGraphScreenshot`. Prefer readable full-graph or node-focused frames where titles, pins, and connections remain visible. For comparable frames, reuse the returned view location, zoom, dimensions, and viewport token.

The screenshot tool needs an initialized graph editor widget. A fully headless or zero-size editor widget is rejected instead of producing a misleading blank image.

## Settings

Open **Project Settings → Plugins → MCP Blueprint**:

- **Port** — default `8766`; up to nine higher ports are tried when occupied.
- **Auto Start** — starts the server with the editor.
- **Auto Compile After Modify** — compiles after supported writes.
- **Request Timeout Seconds** — ordinary tool timeout.
- **Compile Timeout Seconds** — separate timeout for compile-heavy work.

## Known limits

- Windows, macOS, and Linux Editor targets; no runtime/shipping module. Fab's official build pipeline is the release gate for platform binaries.
- No generic tool for invoking an arbitrary Blueprint function and returning its runtime result.
- Custom Event and Event Dispatcher signature mutation is not yet as complete as ordinary function-signature mutation.
- Some Blueprint action types expose state that cannot yet be scanned or restored transactionally and therefore fail closed.
- Screenshot capture requires a real initialized Blueprint graph editor widget.

## Planned improvements

The next development directions are:

1. A bounded generic runtime execution and result-readback path for isolated Blueprint verification.
2. Safety-gated Custom Event and Event Dispatcher signature editing with reference impact and rollback.
3. Deeper graph lifecycle, caller/reference, inheritance, and unloaded-asset impact inspection.
4. Support for additional Blueprint asset types and more complex native Action Registry nodes.
5. Richer end-to-end tutorials covering connection, dry-run, approval, persistence, Undo, and rejection cases.

Detailed priorities and acceptance evidence are maintained in AIHub. This list is directional and does not promise a release date.

## Troubleshooting

- **Cannot connect:** use the endpoint displayed by the toolbar; the configured port may be occupied.
- **Tool not found:** refresh `tools/list`; do not rely on an old cached tool surface.
- **Write rejected:** read every blocker and required approval flag, then re-read the target before retrying.
- **Compile failed:** fetch compiler diagnostics and fix the real Blueprint error before saving.
- **Unexpected graph result:** compare `GetGraphDetail` before/after and use Undo when appropriate.
- **Plugin asks to rebuild:** install the package matching the exact engine minor version.
- **Plugin missing from a project:** confirm it is installed under that engine's `Engine/Plugins/Marketplace`; do not add a project dependency as a workaround.
- **UE 5.5 says the project file is out of date because the installed MCPBlueprint plugin is not in the project descriptor:** choose **Not Now**. The engine plugin still loads automatically; choosing **Update** would write MCPBlueprint into the project's `.uproject` and create the project-side dependency this installation model deliberately avoids.

## Support

For questions or feedback, use the [contact page](https://mengzhishanghun.github.io/mengzhishanghun/contact/).
