[English](./README.md) | [中文](./README_CN.md)

# MCPAnimation

MCPAnimation is a self-contained Unreal Editor plugin that exposes Animation Blueprint, state machine, Skeleton, AnimSequence, Montage, AnimNotify, and BlendSpace discovery, inspection, and editing through HTTP MCP. It supports Unreal Engine 5.2 and later and has no dependency on other user-created plugins.

## Connection

The default endpoint is `http://127.0.0.1:8760/mcp`. If that port is occupied, the server tries subsequent ports; use the actual endpoint shown in the editor. The server implements `initialize`, `tools/list`, `tools/call`, and `ping`.

## Tools (29)

### Discovery and read-only details

| Tool | Purpose |
|---|---|
| `ListAnimBlueprints` | List Animation Blueprints with stable pagination. |
| `GetAnimBPOverview` | Read Animation Blueprint, AnimGraph, and state machine summaries. |
| `GetStateMachineDetail` | Read bounded state, transition, and rule-graph details. |
| `ListSkeletons` / `GetSkeletonDetail` | Discover Skeletons and read bones, virtual bones, compatible skeletons, blend profiles, and slot groups. |
| `ListAnimSequences` / `GetAnimSequenceDetail` | Discover AnimSequences and read duration, sampling rate, notifies, float curves, and compression status. |
| `ListMontages` / `GetMontageDetail` | Discover Montages and read sections, slots, segments, notifies, and blend settings. |
| `ListNotifyClasses` | Discover native and Blueprint AnimNotify/AnimNotifyState classes. |
| `ListBlendSpaces` / `GetBlendSpaceDetail` | Discover BlendSpaces and read axes and samples. |

### Animation Blueprint and state machine editing

| Tool | Purpose |
|---|---|
| `CreateAnimBlueprint` | Create an Animation Blueprint for a Skeleton. |
| `AddState` / `RemoveState` | Add or remove a state. |
| `SetStateAnimation` | Set an animation only when a unique Sequence Player directly driving the state result can be confirmed. |
| `AddTransition` | Create a transition between two states. |
| `SetTransitionRule` | Set a supported transition rule. |
| `CompileAnimBlueprint` | Compile and return diagnostics from AnimGraphs, state graphs, and transition graphs. |

### Montage, Notify, and BlendSpace editing

| Tool | Purpose |
|---|---|
| `CreateMontage` | Create a Montage from an AnimSequence. |
| `AddMontageSection` / `SetMontageSection` / `RemoveMontageSection` | Manage Montage sections. |
| `AddNotify` / `RemoveNotify` | Add or remove a notify on an AnimSequence or Montage. |
| `CreateBlendSpace` | Create a BlendSpace when its axis ranges are valid. |
| `SetBlendSpaceSamples` | Transactionally replace up to 256 validated samples and roll back on failure. |

### Asset lifecycle

| Tool | Purpose |
|---|---|
| `SaveAsset` | Explicitly save a modified asset. |
| `OpenAsset` | Open an asset in Unreal Editor. |

## Paths, transactions, and saving

- Call the matching `List*` tool first to obtain canonical asset paths. Asset writes are restricted to valid project content under `/Game`.
- Write tools run on the editor game thread and use Undo transactions where applicable. Successful changes mark assets dirty.
- Changes are not saved implicitly. Call `SaveAsset` after verifying the result; Unreal Editor Undo remains available where supported.
- Write tools reject execution with a reason while the editor is busy with PIE, saving, or garbage collection.

## Schema, pagination, and bounded output

- Inputs use strict JSON Schema. Unknown properties are rejected, and array items are validated recursively.
- List tools use `Offset` and `MaxResults`, return at most 256 entries per page, and report `TotalCount`, `Returned`, `HasMore`, and `NextOffset` when available.
- Detail tools cap collections such as bones, states, notifies, and curves at 256 entries and report true totals plus `*Truncated` flags. Detail collections are not pageable yet; inspect the complete data in Unreal Editor when the limit is exceeded, or wait for a future pagination capability.

## Boundary with the official UE 5.8 animation MCP

The official UE 5.8 `AnimationAssistantToolset` focuses on Level Sequence, Control Rig, Sequencer channels, and Sequencer curves. MCPAnimation does not duplicate those cross-domain tools; it focuses on Animation Blueprints, Skeletons, AnimSequences, Montages, Notifies, and BlendSpaces. Use the corresponding Level Sequence MCP capability for Sequencer or Control Rig workflows.

`GetAnimSequenceDetail` exposes read-only float-curve summaries, bone and curve compression settings, compression validity, and approximate sizes. Curve and compression writes are intentionally unavailable: those operations can rebuild the animation data model, trigger expensive recompression, and use editing APIs that evolved across UE 5.2–5.8. Automated writes would carry unacceptable asset-corruption risk until transactional rollback, version compatibility, and bounded execution can all be guaranteed.

## Limitations

- This is an Editor-only plugin; it does not provide runtime or network-game replication features.
- Detail tools provide structured summaries rather than every curve key or per-bone animation track.
- Complex or ambiguous state result graphs are not modified by guesswork. Simplify them to one direct driver or edit them manually in Unreal Editor.
- The tools do not compile C++, build Unreal projects, or package plugins.

For questions or feedback, email [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com). I will take care of it when I see your message.
