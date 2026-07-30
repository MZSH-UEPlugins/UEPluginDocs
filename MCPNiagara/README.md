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

## Module details

`GetModuleDetail` inspects only modules on the selected Spawn, Update, or Event parameter-map chain, so `ModuleIndex` follows actual execution order. Duplicate names require an index; when both a name and index are supplied, they must identify the same module. `UsageId` is required when an emitter has multiple Event stages.

Module inputs are paged with `MaxResults` (1–256, default 100) and `Offset`. Each input reports its Niagara type and `ValueSource`; graph overrides, linked or dynamic overrides, and rapid-iteration values are distinguished explicitly. Inputs without an override are reported as `ModuleDefaultOrBinding`. To remain self-contained on UE 5.2+, the tool does not depend on NiagaraEditor-internal default-topology APIs and does not guess whether that source is a module default or an internal binding.

## Emitter editing

`AddEmitter` uses NiagaraEditor's supported copy path to assign a unique name, rebuild emitter nodes, and synchronize the overview graph. `RemoveEmitter` and `RenameEmitter` require an unambiguous name. Add, remove, and rename operations are transactional and broadcast the system edit notification.

`SetEmitterProperties` atomically writes up to 64 properties on `FVersionedNiagaraEmitterData`: every property and value is validated before any change is committed, and each changed property receives a version-aware post-edit notification. Editable scalar, enum, string, name, and struct properties are supported. Object references, containers, delegates, transient, deprecated, and non-editable properties are rejected.

Limitation: UE 5.2 does not export NiagaraEditor's private merge-adapter cache invalidation API. `RemoveEmitter` destroys referencing instances, removes and reconnects the public system graph, synchronizes the overview graph, and broadcasts the edit, but it cannot explicitly clear that private cache. The response includes `Warnings`; refresh or reopen the system before immediately running an inherited-emitter merge workflow. The plugin does not depend on engine Private headers or unexported symbols to bypass this boundary.

## Renderer editing

`SetRendererType` replaces up to 64 renderers on an unambiguously named emitter in one undo transaction. The new renderer is created before old entries are removed, so construction failure preserves the existing configuration. The UE 5.2+ common public types are Sprite, Mesh, Ribbon, Light, Decal, and Component; newer-only Volume support is not exposed unconditionally.

`SetRendererProperties` atomically writes 1–64 properties per call. Every name and value is first validated on a temporary renderer, then committed together with per-property PostEditChange notifications so binding rebuilds and required compile invalidation occur. Editable scalar, enum, string, name, struct, and type-compatible material-reference properties are supported. Containers, delegates, non-material object references, transient, deprecated, and non-editable properties are rejected. `RendererIndex` must be non-negative, and the tool schemas declare the same enum, count, and value-type bounds.

## Validation boundary

The source targets Unreal Engine 5.2 and newer. This pass performs static source verification only; real editor writes, save/reopen behavior, and multi-version packaging still require later validation.

For questions or feedback, email [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com). I will take care of it when I see your message.
