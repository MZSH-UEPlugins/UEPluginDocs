[English](./README.md) | [中文](./README_CN.md)

# MCPBlueprint User Guide

MCPBlueprint exposes Blueprint discovery, graph editing, members, components, asset lifecycle operations, compile feedback, and graph capture to MCP-compatible AI clients. This guide describes the user workflow; the live `tools/list` response is the authority for the exact tool and parameter surface in the current editor session.

## Connect

1. Install and enable MCPBlueprint as an engine plugin.
2. Open **Project Settings → Plugins → MCP Blueprint**.
3. Keep **Auto Start** enabled, or start the server from the plugin toolbar.
4. Add `http://127.0.0.1:8766/mcp` to your MCP client. When the configured port is occupied, MCPBlueprint tries the next ports and shows the actual endpoint in the toolbar.
5. Ask the client to call `initialize`, `tools/list`, and `ping` before editing assets.

The server accepts local HTTP MCP requests. Browser-style `Origin` headers are limited to `localhost`, `127.0.0.1`, and `[::1]`; ordinary non-browser MCP clients may omit `Origin`.

## Recommended workflow

Use the same sequence for every write operation:

1. **Inspect** — identify the exact asset, graph, node GUID, pin, member, or component with read tools.
2. **Preview** — when a tool supports `bDryRun`, run the exact intended operation in dry-run mode and review blockers, affected assets, and normalized operations.
3. **Apply** — send the approved write once. Do not invent node spawner IDs or stable GUIDs.
4. **Read back** — query the affected graph or asset again and verify the intended values, nodes, pins, links, or names.
5. **Compile** — inspect authoritative compile status and diagnostics.
6. **Save explicitly** — write operations mark supported assets dirty but do not silently save them.

High-risk operations such as function rename, function-signature changes, destructive struct/enum edits, and disk reload use explicit approval flags. Treat a reported blocker as a rejection, not as permission to bypass the gate.

## Discover assets and graphs

- Use `ListBlueprints` to find assets with bounded pagination.
- Use `GetBlueprintOverview` to inspect graphs, variables, functions, components, interfaces, and the parent class.
- Use `ListBlueprintMembers` for functions, custom events, dispatchers, and local variables.
- Use `GetGraphDetail` before graph edits to obtain current node GUIDs, pins, defaults, links, and coordinates.
- Use `GetCompileErrors` after a write to obtain compile status and diagnostics.

Always pass exact asset paths returned by the plugin. Read large graphs in bounded detail instead of guessing from a screenshot.

## Add graph logic safely

1. Read the target graph with `GetGraphDetail`.
2. Search Unreal's English action name with `SearchGraphNodes`, for example `Branch`, `For Loop`, `For Each Loop`, `Switch on Int`, `Add`, `Subtract`, `Multiply`, or `Divide`.
3. Copy the returned registry-backed `SpawnerId` into `ApplyGraphPatch`; never fabricate one.
4. Preview the complete patch with `bDryRun=true`.
5. Apply the same patch with `bDryRun=false` only after reviewing every blocker and normalized operation.
6. Read back the graph, compile it, and save it explicitly.

Standard flow-control actions are created from Unreal's real Action Registry. Verify the returned node classes and pins with `GetGraphDetail` instead of treating a screenshot as structural evidence.

`ApplyGraphPatch` can create, remove, connect, disconnect, set defaults, move, and lay out bounded graph changes as one transaction. `FormatGraph` handles an existing graph or selection, while `AddCommentBox` creates native comment boxes around selected nodes. Use `LayoutScope` and `LayoutStyle` to express the desired scope and presentation rather than manually moving unrelated nodes.

### One call or several calls

A complete graph can be created and connected in one patch.

You can also create nodes first and connect them in a later patch; automatic layout uses the real resulting connections.

When a compatible event already exists, reuse it instead of creating a duplicate.

## Members, structs, and enums

- Variables: `AddVariable`, `ModifyVariable`, and `RemoveVariable`.
- Functions: `CreateFunction`, `RenameFunction`, `ModifyFunctionSignature`, and `RemoveFunction`.
- Events and dispatchers: `CreateCustomEvent`, `AddEventNode`, `AddBoundEvent`, `AddEventDispatcher`, and `RemoveEventDispatcher`.
- Local variables: add or remove the declaration, then use function-scoped actions returned by `SearchGraphNodes` for local Get/Set nodes.
- User Defined Structs and Enums: create, inspect, and modify them through stable field or entry identity.

Preview reference impact first. Apply only with the approval flags requested by the tool response, especially when an operation may discard serialized data, change enum value semantics, or update external callers.

## Components and asset lifecycle

Component tools manage Actor Blueprint SCS components and their template properties. Read the current component tree before adding, renaming, reparenting, duplicating, setting the root, or removing a component.

Asset lifecycle tools create, duplicate, rename, reparent, delete, open, close, reload, compile, and save supported Blueprint assets. Use dry-run or explicit discard approval when reloading from disk, and never assume an in-memory edit was saved merely because compilation succeeded.

## Capture a graph screenshot

Use `CaptureGraphScreenshot` only after the graph has been read back and compiled. Prefer a readable graph or node-focused frame over a zoom level that makes pins and labels illegible. For comparable frames, reuse the response's view location, zoom, width, height, and viewport token.

Screenshots in this guide demonstrate user-facing operations. Development validation, historical comparisons, release preparation, and rejected captures are maintained outside the plugin repository.

## Settings

Open **Project Settings → Plugins → MCP Blueprint**:

- **Port** — default `8766`; when occupied, the server tries up to nine higher ports.
- **Auto Start** — starts the server with the editor.
- **Auto Compile After Modify** — compiles after supported write operations.
- **Request Timeout Seconds** — timeout for ordinary tools.
- **Compile Timeout Seconds** — separate timeout for compile operations.

## Troubleshooting

- **Cannot connect:** confirm the endpoint displayed in the toolbar; the configured port may have been occupied.
- **Tool not found:** refresh `tools/list`; do not rely on an old cached tool surface.
- **Write rejected:** inspect the returned blocker or required approval flag and re-read the target state before retrying.
- **Compile failed:** call the compile-diagnostic tool and fix the reported Blueprint error before saving.
- **Unexpected graph result:** compare `GetGraphDetail` before and after; use Undo when appropriate, then re-read the graph.

## Support

For questions or feedback, use the [contact page](https://mengzhishanghun.github.io/mengzhishanghun/contact/).
