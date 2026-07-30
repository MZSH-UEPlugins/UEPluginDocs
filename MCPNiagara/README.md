[English](./README.md) | [中文](./README_CN.md)

# MCP Niagara

MCP Niagara provides MCP tools in Unreal Editor for Niagara systems, emitters, module stacks, parameters, renderers, compile status, and asset lifecycle operations.

## Asset discovery and pagination

`ListNiagaraSystems`, `ListNiagaraEmitters`, and `SearchModules` provide stable pagination:

- `MaxResults` must be between 1 and 500 and defaults to 100.
- `Offset` must be a non-negative integer and defaults to 0.
- Responses include `TotalCount`, `Returned`, `Offset`, and `HasMore`; `NextOffset` is included when another page exists.
- Results are sorted by full asset object path, so consecutive pages do not depend on Asset Registry enumeration order.

`SearchModules` returns only visible Niagara module library assets accepted by the Niagara editor. Deprecated, hidden, and non-library scripts are excluded, and the search does not load every script asset individually.

`GetSystemDetail` pages emitters and user parameters independently: `MaxEmitters` is 1–128, `MaxUserParameters` is 1–256, and both offsets must be non-negative. Each emitter returns at most 64 renderers and 64 simulation stages with total and truncation fields. User parameters are stably sorted by name and type; value text is limited to 4096 characters, while object and data-interface parameters return object paths. Every tool schema rejects undeclared root fields by default, and tools with enums, pagination, or dynamic objects declare their additional bounds.

## Module details

`GetModuleDetail` inspects only modules on the selected Spawn, Update, or Event parameter-map chain, so `ModuleIndex` follows actual execution order. Duplicate names require an index; when both a name and index are supplied, they must identify the same module. `UsageId` is required when an emitter has multiple Event stages.

Module inputs are paged with `MaxResults` (1–256, default 100) and `Offset`. Each input reports its Niagara type and `ValueSource`; graph overrides, linked or dynamic overrides, and rapid-iteration values are distinguished explicitly. `Value` is limited to 4096 characters, and `bValueTruncated` explicitly reports truncation. Inputs without an override are reported as `ModuleDefaultOrBinding`. To remain self-contained on UE 5.2+, the tool does not depend on NiagaraEditor-internal default-topology APIs and does not guess whether that source is a module default or an internal binding.

## Emitter editing

`AddEmitter` uses NiagaraEditor's supported copy path to assign a unique name, rebuild emitter nodes, and synchronize the overview graph. `RemoveEmitter` and `RenameEmitter` require an unambiguous name. Add, remove, and rename operations are transactional and broadcast the system edit notification.

`SetEmitterProperties` atomically writes 1–64 properties on `FVersionedNiagaraEmitterData`: every property and value is validated before any change is committed, and each changed property receives a version-aware post-edit notification. Editable scalar, enum, string, name, and struct properties are supported. Object references, containers, delegates, transient, deprecated, and non-editable properties are rejected. The dynamic-object schemas for `SetEmitterProperties` and `SetModuleParameters` accept only strings, numbers, or booleans and match the runtime count bounds. String inputs and dynamic property names have schema and runtime limits of at most 4096 characters, with stricter limits for some asset paths; input JSON is also limited to 8 levels and 2048 values. Every tool that loads an existing system returns its canonical object `SystemPath` on success. Final error text is limited to 4096 characters, and available-name candidates have both count and text bounds.

Limitation: UE 5.2 does not export NiagaraEditor's private merge-adapter cache invalidation API. `RemoveEmitter` destroys referencing instances, removes and reconnects the public system graph, synchronizes the overview graph, and broadcasts the edit, but it cannot explicitly clear that private cache. The response includes `Warnings`; refresh or reopen the system before immediately running an inherited-emitter merge workflow. The plugin does not depend on engine Private headers or unexported symbols to bypass this boundary.

## Renderer editing

`SetRendererType` replaces up to 64 renderers on an unambiguously named emitter in one undo transaction. The new renderer is created before old entries are removed, so construction failure preserves the existing configuration. The UE 5.2+ common public types are Sprite, Mesh, Ribbon, Light, Decal, and Component; newer-only Volume support is not exposed unconditionally.

`SetRendererProperties` atomically writes 1–64 properties per call. Every name and value is first validated on a temporary renderer, then committed together with per-property PostEditChange notifications so binding rebuilds and required compile invalidation occur. Editable scalar, enum, string, name, struct, and type-compatible material-reference properties are supported. Containers, delegates, non-material object references, transient, deprecated, and non-editable properties are rejected. `RendererIndex` must be non-negative, and the tool schemas declare the same enum, count, and value-type bounds.

## User parameter editing

`AddUserParameter` adds a float, int, bool, Vector, Vector2, Vector4, or Color value parameter in an undo transaction and sends a system PostEditChange. `SetSystemParameters` atomically writes 1–64 existing value parameters per call: all names and values are validated and old values are snapshotted before commit. JSON booleans are stored using Niagara's `FNiagaraBool` representation. `RemoveUserParameter` uses the same public exposed-parameter removal semantics as UE 5.8 External and requires an unambiguous name.

Object and data-interface parameters require typed asset or instance handling and cannot safely use text-value import, so `AddUserParameter` and `SetSystemParameters` do not claim support for them. Removal does not guess how graph references should be rewritten; callers must update references separately. UE 5.2–5.8 do not share an exported user-parameter rename and reference-propagation API, so the plugin does not depend on Private headers to offer `RenameUserParameter`.

## Asset lifecycle and preview

`CreateNiagaraSystem` accepts only a valid long package name in a writable mount and rejects a `.ObjectName` suffix. Disk packages, loaded packages, and exact Asset Registry entries are checked before creation, so an existing asset is never overwritten. `TemplatePath` performs a real template duplication; without it, the Niagara factory creates an empty system. The response reports the canonical object path, package name, and dirty state; call `SaveAsset` to persist it.

`SaveAsset` uses non-fatal package-name conversion and rejects read-only targets. It succeeds only when `SavePackage` succeeds, the file exists, and the package is no longer dirty. `OpenAsset` checks both the editor and AssetEditorSubsystem lifecycle. `CapturePreview` reads only a thumbnail already cached in memory or stored in a saved package; it does not render or replace the thumbnail cache. Exactly one PNG is returned, bounded to 1024×1024 pixels, 16 MiB raw data, and 8 MiB compressed data.

## Validation boundary

The source targets Unreal Engine 5.2 and newer. This pass performs static source verification only; real editor writes, save/reopen behavior, and multi-version packaging still require later validation.

For questions or feedback, email [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com). I will take care of it when I see your message.
