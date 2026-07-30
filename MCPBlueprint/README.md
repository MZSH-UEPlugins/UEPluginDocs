[English](./README.md) | [中文](./README_CN.md)

# MCPBlueprint

MCPBlueprint is a self-contained Unreal Editor plugin that exposes Blueprint discovery, graph editing, members, components, asset lifecycle operations, and compile feedback through MCP over HTTP. It supports Unreal Engine 5.2 and later and does not depend on another user plugin.

## Connection

The default endpoint is `http://127.0.0.1:8766/mcp`. If that port is occupied, the server tries the next ports and the toolbar displays the actual endpoint. Add that URL as an HTTP MCP server in your AI client.

The server implements `initialize`, `tools/list`, `tools/call`, and `ping`. Write operations run on the game thread, participate in editor transactions where Unreal supports Undo, mark assets dirty, and do not save automatically.

Requests must use JSON-RPC `2.0` with object-valued `params` and tool `arguments`. Browser-style `Origin` headers are accepted only for `localhost`, `127.0.0.1`, or `[::1]`; non-browser clients may omit `Origin`.

## Tools (38)

### Discovery and reading

| Tool | Purpose |
|---|---|
| `ListBlueprints` | List Blueprint assets with stable bounded pagination. |
| `GetBlueprintOverview` | Read graphs, variables, functions, components, interfaces, and parent class. |
| `GetGraphDetail` | Read graph nodes, pins, defaults, and links. |
| `SearchGraphNodes` | Search Blueprint action spawners and return stable `SpawnerId` values. |
| `GetCompileErrors` | Compile and return the authoritative Blueprint status and diagnostics. |

### Variables, functions, and events

| Tool | Purpose |
|---|---|
| `AddVariable` / `ModifyVariable` / `RemoveVariable` | Manage Blueprint member variables. |
| `CreateFunction` | Create a function or override with validated input/output signatures. |
| `CreateCustomEvent` / `AddEventNode` / `AddBoundEvent` | Add custom, engine, component, or level-actor events. |
| `AddLocalVariable` | Add a function-local variable. |
| `AddNodePin` | Add supported dynamic pins to Sequence, container, and Switch nodes. |
| `AddInterface` | Implement a Blueprint interface. |
| `AddEventDispatcher` | Create a multicast event dispatcher with an optional signature. |

### Graph editing

| Tool | Purpose |
|---|---|
| `ApplyGraphPatch` | Apply bounded node create/remove, connect/disconnect, defaults, and layout changes as one declarative transaction. |
| `SetPinDefaults` | Set validated literal defaults on existing input pins. |
| `FormatGraph` | Lay out a whole graph or a selected node set. |

### Actor Blueprint components and defaults

| Tool | Purpose |
|---|---|
| `GetComponents` / `GetComponentProperties` | Read the local SCS hierarchy and editable component-template properties. |
| `AddComponent` / `DuplicateComponent` / `RenameComponent` / `RemoveComponent` | Manage local SCS components. |
| `ReparentComponent` / `SetRootComponent` | Edit the local SceneComponent hierarchy. |
| `SetComponentProperties` | Atomically import editable component-template values in Unreal text format. |
| `SetClassDefaults` | Atomically set editable Class Default Object values. |

### Asset lifecycle and capture

| Tool | Purpose |
|---|---|
| `CreateBlueprint` | Create an in-memory Blueprint asset under `/Game`. |
| `CompileBlueprint` | Compile and return authoritative status and diagnostics. |
| `SaveAsset` / `OpenAsset` / `CloseAsset` | Save or control the editor for a standalone Blueprint asset. |
| `DeleteAsset` / `RenameAsset` / `DuplicateAsset` | Manage standalone Blueprint assets. |
| `CaptureGraphScreenshot` | Return a PNG and optionally save it below `Saved/MCPBlueprint/Screenshots`. |

## Safety and behavior boundaries

- New or moved Blueprint asset paths must be valid standalone package paths under `/Game`; existing in-memory and on-disk package collisions are rejected.
- Level Blueprints can be read and graph-edited by object path, but `SaveAsset`, `DeleteAsset`, `RenameAsset`, and `DuplicateAsset` do not operate on them because those actions affect the owning level package. Use a World Editor workflow for the level itself.
- `CloseAsset` reports whether an editor was open and verifies that all editors for the Blueprint and its subobjects actually closed; cancellation is returned as an error.
- Member and local variable defaults are validated with Unreal's K2 pin rules before mutation; malformed UE text values are rejected.
- Variable type changes, renames, and removals reject unsafe operations while derived Blueprint classes are unloaded. Load descendants first so inherited references can be checked; child references must be removed before deleting a declaration.
- With automatic compilation enabled, transactional Blueprint mutations return bounded compiler diagnostics and roll back when compilation does not reach `UpToDate` or `Warning`; when disabled, mutation results report `CompileStatus: Skipped`. Rollback status is reported explicitly.
- Component names are validated against the complete Blueprint member namespace. Ambiguous short component class names are rejected; use a full class path to disambiguate.
- Component classes must be Blueprint-spawnable. A new SceneComponent without an explicit parent attaches to the unique local, inherited, or native scene root; ambiguous multi-root hierarchies are rejected instead of creating a second root.
- SCS hierarchy edits snapshot the local hierarchy for Undo and roll back on compile failure. `SetRootComponent` only replaces one unambiguous local scene root, never an inherited/native root.
- `GetComponentProperties` uses `Offset`/`MaxResults` pagination (maximum 100 properties) and truncates individual exported values at 8,192 characters while reporting affected property names.
- `GetGraphDetail` requires bounded pagination and returns at most 200 nodes per call; non-positive unlimited requests are rejected.
- Tool schemas reject unknown top-level arguments and validate the nested graph-patch, signature, pin-default, component-property, and class-default structures before execution.
- HTTP request bodies are rejected above 1 MiB after the engine HTTP server has received them. Serialized responses are limited to 16 MiB; tool text is limited to 1 Mi characters, and image output is limited to four images, 8 Mi base64 characters per image, and 12 Mi total.
- Compile diagnostics return at most 200 error/warning entries, 4,096 characters per message, 256 Ki characters in total, and 32 unique node GUIDs per entry. `TotalErrors`, `TotalWarnings`, returned counts, and `bTruncated` disclose omitted diagnostics. `CompileBlueprint` and `GetCompileErrors` return MCP errors for non-success terminal states; `bWarningsAsErrors` can make warnings fail the call.
- Request timeout settings bound dispatcher waiting, but synchronous Blueprint/UObject work already running on the game thread cannot be preempted. Avoid immediate retries after a client-side timeout until editor state is known.
- Destructive asset deletion is not undoable. Without `bForce`, referenced assets are refused; forced deletion may clear references and break dependents.
- Screenshot file output is restricted to `Saved/MCPBlueprint/Screenshots`; absolute paths and traversal are rejected.
- Source-only compatibility review is not a substitute for compiling and testing inside each target engine version.

## Configuration

Open **Project Settings > Plugins > MCP Blueprint**.

| Setting | Default | Description |
|---|---:|---|
| Port | 8766 | First HTTP port to try. |
| Auto Start | true | Start the server when the editor module loads. |
| Request Timeout | 30 seconds | Normal tool timeout. |
| Compile Timeout | 120 seconds | Compile/patch timeout. |
| Auto Compile After Modify | true | Compile after supported write operations. |

For questions or feedback, email [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com). I will take care of it when I see your message.
