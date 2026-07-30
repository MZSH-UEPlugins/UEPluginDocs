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

## Validation boundary

The source targets Unreal Engine 5.2 and newer. This pass performs static source verification only; real editor writes, save/reopen behavior, and multi-version packaging still require later validation.

For questions or feedback, email [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com). I will take care of it when I see your message.
