[English](./README.md) | [中文](./README_CN.md)

# MCPMaterial

MCPMaterial exposes Unreal Editor material and material-instance workflows to AI clients through MCP tools. The plugin supports Unreal Engine 5.2 and later. Its default endpoint is `http://127.0.0.1:8763/mcp`; when the port is occupied, the server tries a limited sequence of higher ports.

## Capabilities

- Discover materials, material instances, and available expression classes.
- Read material graphs, connections, properties, and Scalar, Vector, Texture, and Static Switch parameters.
- Create materials and material instances, then batch-edit properties, parameters, expression nodes, and connections.
- Compile materials, save assets, open or close asset editors, and capture material previews.
- Write tools use transactions and mark packages dirty. Call `CompileMaterial` before `SaveAsset` when the material must be validated first.

## Safe workflow

1. Use `ListMaterials`, `GetMaterialDetail`, `GetMaterialParameters`, and `SearchMaterialExpressions` to obtain real paths, GUIDs, pins, and parameter identities.
2. If the target material or material instance is open in an asset editor, save or resolve the editor changes and then call `CloseAsset`.
3. Call `ApplyMaterialPatch`, `SetMaterialProperties`, `SetInstanceParameters`, `ResetInstanceParameter`, `RemoveConnection`, or `FormatGraph`.
4. Call `CompileMaterial` and inspect the terminal compile result.
5. Call `SaveAsset` to persist the changes. Call `OpenAsset` afterward when a manual review is needed.

Write, compile, and save tools reject targets that are still open in an asset editor. This prevents the Material Editor's transient preview and the backing asset from diverging, and prevents a later editor Apply operation from overwriting completed MCP changes.

## Important boundaries

- `SetInstanceParameters` and `ResetInstanceParameter` accept only uniquely identified Global parameters. Material Layer and Blend parameters require structured Association/Index addressing and cannot currently be written by name.
- `SetInstanceParameters` validates texture parameters against the parent material's current compiled resource. 2D, Cube, 2D/Cube Array, Volume, and Virtual textures are not interchangeable. The write is rejected when the parent has no usable resource or the parameter is absent from the active static-switch permutation.
- `SearchMaterialExpressions` returns a stable `ClassPath`, `Creatable`, an uncreatable reason, and input/output indices. When short class names collide, `ApplyMaterialPatch` rejects guessing and requires an exact `ClassPath`; duplicate pin names must be addressed as `[index]`.
- `ApplyMaterialPatch` does not create or remove Composite, PinBase, Function Input/Output, Named Reroute, or other nodes that require specialized editor-managed structure.
- The server recursively enforces every tool input schema, rejecting unknown fields, missing required fields, invalid types, and out-of-range values even for raw `tools/call` requests. `ApplyMaterialPatch` allows at most 1,000 items in each of Nodes, Connections, and RemoveNodes, with a combined limit of 2,000 operations. Known pagination, preview-size, graph-spacing, connection-index, and parameter-type bounds are validated before execution.
- HTTP request bodies default to a 1 MiB limit and JSON-RPC response bodies to 8 MiB; both are configurable in the MCP Material project settings. `CapturePreview` is capped at 1024×1024 and rejects compressed PNG data larger than 4 MiB. Oversized tool results preserve the real success/error terminal state while omitting excessive content; do not blindly retry a successful write tool solely because its result was omitted.
- `GetMaterialDetail` uses `NodeOffset`/`MaxNodes` for nodes and `ConnectionOffset`/`MaxConnections` for connections. `GetMaterialParameters` uses one `Offset`/`MaxResults` cursor across Scalar, Vector, Texture, and StaticSwitch in that order. Responses include totals, returned counts, and next cursors; every page is capped at 1000 entries.
- Newly created assets remain dirty in memory until `SaveAsset` is called explicitly.
- A timeout cannot forcibly stop an editor operation after the GameThread has claimed it. After a timeout error, inspect editor state before retrying.

For questions or feedback, email [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com). I will take care of it when I see your message.
