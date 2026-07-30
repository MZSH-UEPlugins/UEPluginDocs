[English](./README.md) | [中文](./README_CN.md)

# MCP Level Sequence

MCP Level Sequence is a self-contained Unreal Editor plugin that lets AI clients discover and edit Level Sequences through an MCP-compatible JSON-RPC 2.0 HTTP service. It supports Unreal Engine 5.2 and later without depending on other user plugins or cross-plugin shared modules.

## Installation and connection

1. Place `Plugins/MCPLevelSequence` in a project or engine plugin directory and enable it.
2. Start Unreal Editor. By default, the service starts at `http://127.0.0.1:8762/mcp`.
3. If the port is occupied, the server tries up to 10 following ports. Use the actual endpoint shown by the editor toolbar and log.
4. Configure the port, automatic startup, and request timeout under `Project Settings > Plugins > MCP Level Sequence`.

The service supports `initialize`, `ping`, `tools/list`, and `tools/call`. HTTP request bodies are limited to 1 MiB; larger requests receive HTTP 413. Requests must use `jsonrpc: "2.0"`; `params` and `arguments` must be objects, and each tool is checked against its schema for unknown fields, required fields, types, array items, and numeric bounds.

## 23 tools

| Group | Tools | Purpose |
|---|---|---|
| Sequence | `ListSequences`, `GetSequenceDetail`, `CreateSequence` | Discover, inspect with pagination, and create sequences |
| Time | `SetPlaybackRange`, `SetSequenceRates` | Set playback ranges, display rates, and tick resolutions |
| Binding | `ListBindings`, `AddBinding`, `RemoveBinding` | Page, add, and cascade-remove object bindings |
| Track | `ListTrackTypes`, `AddTrack`, `RemoveTrack` | Query, add, and cascade-remove tracks |
| Section | `AddSection`, `RemoveSection`, `SetSectionRange` | Create, remove, and resize sections |
| Key | `GetKeyframes`, `SetKeyframes`, `RemoveKeyframes` | Page, write, and remove keys |
| Camera | `AddCameraCut`, `SetCameraCut` | Create and modify camera cuts |
| Lifecycle | `OpenAsset`, `SaveAsset`, `CloseAsset` | Open, save one asset, and verify close results |
| Preview | `CaptureSequencerView` | Capture the current Sequencer view as PNG |

## Tracks, sections, and channels

`AddTrack` supports eight track classes:

- Binding tracks: `MovieScene3DTransformTrack`, `MovieSceneFloatTrack`, `MovieSceneBoolTrack`, and `MovieSceneSkeletalAnimationTrack`.
- Root or binding tracks: `MovieSceneAudioTrack` and `MovieSceneEventTrack`.
- Root-only tracks: `MovieSceneCameraCutTrack` and `MovieSceneFadeTrack`.

Float and Bool property tracks require `PropertyName` and accept `PropertyPath` for nested properties. Skeletal Animation sections require `AnimationAssetPath`, validated as `AnimSequenceBase`; Audio sections require `SoundAssetPath`, validated as `SoundBase`. Event tracks currently create section containers only; they do not create Director Blueprint endpoints or event payloads.

Keyframe tools support Float, Double, Bool, and Byte channel families. Use the channels returned by `GetSequenceDetail` as the authoritative editable surface for Transform, Fade, property, and other tracks.

## Time units and frame rates

Every `StartFrame`, exclusive `EndFrame`, key position, and camera-cut frame uses the sequence's `TickResolution`, not its `DisplayRate`. `GetSequenceDetail` returns both rational rates.

`SetSequenceRates` can set the Display Rate, Tick Resolution, or both. Each supplied rate requires a positive integer numerator and denominator. When Tick Resolution changes, the plugin uses the engine's frame-time migration path for playback/selection ranges, tracks, sections, keys, and marked frames, preserving real time. It does not recursively modify Subsequence assets.

## Pagination and output limits

- `ListSequences`: 100 by default, capped at 500 per page.
- `ListBindings`: 100 by default, capped at 500 and stably sorted by BindingId.
- `GetKeyframes`: 200 by default, capped at 1000 and returns `NextOffset`.
- `GetSequenceDetail`: pages a stable combined Binding/Track/Section traversal with `Offset` and `MaxItems`.
- `CaptureSequencerView`: at most 16 Mi pixels and 8 MiB of compressed PNG data.

Paged responses report totals, the current offset, whether more data exists, and a `NextOffset` when applicable. Clients should continue paging instead of assuming that one response contains the whole asset.

## Writes, transactions, and asset lifecycle

Mutation and save tools accept only valid `/Game/...` Level Sequence paths. Read-only discovery, inspection, opening, capture, and closing may access other readable sequence locations. Editing tools reject PIE/save/GC busy states and read-only MovieScenes, and use editor transactions for Undo.

A successful edit only marks the package dirty; it never saves implicitly. `SaveAsset` saves only the specified Level Sequence package. After requesting a close, `CloseAsset` queries editors for the asset and its subobjects again; if an editor refuses to close, the tool reports failure instead of a false success.

Binding and track removal cascade through owned objects and reject destructive operations when an invalid, locked, or read-only section is present. Name selectors report ambiguity instead of choosing the first match. Prefer `BindingId` from `ListBindings` and `TrackName` from `GetSequenceDetail`.

## Call example

This request sets a 24 fps display rate and migrates Tick Resolution to 24000 fps:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "SetSequenceRates",
    "arguments": {
      "AssetPath": "/Game/Cinematics/MySequence",
      "DisplayRateNumerator": 24,
      "DisplayRateDenominator": 1,
      "TickResolutionNumerator": 24000,
      "TickResolutionDenominator": 1
    }
  }
}
```

The recommended flow is to call `ListSequences` and `GetSequenceDetail`, perform mutations with stable IDs/names, and then call `SaveAsset` explicitly.

## Boundary versus Unreal Engine 5.8's experimental toolset

UE 5.8's experimental Animation Assistant Toolset is a broad Python surface that depends on SequencerScripting. This plugin deliberately keeps a self-contained C++ HTTP boundary compatible with UE 5.2+. This pass adds the capabilities that have stable cross-version APIs: sequence rate editing, Skeletal Animation tracks/sections, and Audio asset sections.

The following differences are intentional:

- Subsequence creation/navigation needs cycle checks, hierarchy ownership, and cross-sequence time semantics, so write tools are not exposed yet.
- Spawnable creation has version-sensitive template ownership, Undo, and spawner paths and needs dedicated runtime validation before exposure.
- Event payloads depend on Director Blueprint endpoints/graphs. The current plugin creates Event sections but does not promise a concrete trigger.
- Arbitrary track/class creation, folders, tags, selection state, and advanced bone-level operations remain outside the reviewed strict-schema surface, although the official experimental toolset is broader.

The source has been statically checked against UE 5.2 and 5.8 headers and call paths. Compilation, live editor behavior, asset save/close interaction, and complete Undo behavior remain part of the later coordinated build and runtime validation boundary.

For questions or feedback, email [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com). I will take care of it when I see your message.
