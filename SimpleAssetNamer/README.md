# Simple Asset Namer User Guide

## 1. Settings

Open **Edit → Project Settings → Simple Asset Namer**

### 1.1 General

| Option | Description |
|--------|-------------|
| Auto Rename on Create | Automatically rename assets when created or imported |
| Auto Rename on Rename | Automatically correct asset names after manual renaming |

### 1.2 Filter

| Option | Description |
|--------|-------------|
| Filter Type | Blacklist (exclude specified paths) / Whitelist (only process specified paths) |
| Exclude Paths | Assets under these paths will be skipped in Blacklist mode |
| Include Paths | Only assets under these paths will be processed in Whitelist mode |

**Default Excluded Paths**:
- `/Game/__ExternalActors__` - World Partition auto-generated
- `/Game/__ExternalObjects__` - World Partition auto-generated
- `/Game/Developers` - Developer personal folders
- `/Game/Collections` - Collections folder

### 1.3 Delimiter

| Option | Description | Default |
|--------|-------------|---------|
| Delimiter | Delimiter used when generating names | `_` |
| Delimiter Detection List | Delimiters to detect in existing names | `_`, `-`, `.` |
| Legacy Affixes | Legacy prefix/suffix list for detecting old naming patterns | - |

**Example**: Input `bp-MyActor` or `bp.MyActor` → Output `BP_MyActor`

### 1.4 Naming Rules

| Option | Description |
|--------|-------------|
| Asset Type Naming Rules | Mapping of asset types to prefix/suffix, fully customizable |

**Naming Format**: `Prefix_Name_Suffix` (suffix is optional)

**Default Rules (52 types)**:

| Category | Asset Type | Prefix | Suffix | Example |
|----------|-----------|--------|--------|---------|
| Gameplay | GameModeBase | GM | | `GM_MyMode` |
| | GameStateBase | GS | | `GS_MyState` |
| | PlayerController | PC | | `PC_MyController` |
| | PlayerState | PS | | `PS_MyState` |
| | Character | CHAR | | `CHAR_MyCharacter` |
| Blueprint | Blueprint | BP | | `BP_MyActor` |
| | BlueprintGeneratedClass | BPI | | `BPI_MyInterface` |
| | BlueprintFunctionLibrary | BPFL | | `BPFL_MyLib` |
| | BlueprintMacroLibrary | BPML | | `BPML_MyMacro` |
| | UserDefinedEnum | E | | `E_MyEnum` |
| | UserDefinedStruct | F | | `F_MyStruct` |
| Mesh | StaticMesh | SM | | `SM_Chair` |
| | SkeletalMesh | SK | | `SK_Character` |
| Material | Material | M | | `M_Wood` |
| | MaterialInstance | MI | | `MI_Wood_Dark` |
| | MaterialInstanceConstant | MI | | `MI_Wood_Const` |
| | MaterialFunction | MF | | `MF_Blend` |
| | MaterialParameterCollection | MPC | | `MPC_Global` |
| | SubsurfaceProfile | SSP | | `SSP_Skin` |
| Animation | AnimBlueprint | ABP | | `ABP_Character` |
| | AnimSequence | AS | | `AS_Walk` |
| | AnimMontage | AM | | `AM_Attack` |
| | AnimComposite | AC | | `AC_Combo` |
| | AimOffsetBlendSpace | AO | | `AO_Aim` |
| | BlendSpace | BS | | `BS_Locomotion` |
| | Skeleton | SKEL | | `SKEL_Character` |
| Audio | SoundWave | S | | `S_Footstep` |
| | SoundCue | SC | | `SC_Ambient` |
| | SoundAttenuation | ATT | | `ATT_Default` |
| | SoundClass | SClass | | `SClass_Music` |
| | SoundMix | Mix | | `Mix_Default` |
| VFX | ParticleSystem | PS | | `PS_Fire` |
| | NiagaraSystem | NS | | `NS_Smoke` |
| | NiagaraEmitter | NE | | `NE_Spark` |
| Texture | Texture2D | T | | `T_Wood_D` |
| | TextureCube | T | Cube | `T_Sky_Cube` |
| | TextureRenderTarget2D | RT | 2D | `RT_Scene_2D` |
| | TextureRenderTargetCube | RT | Cube | `RT_Reflect_Cube` |
| | TextureRenderTargetVolume | RT | Volume | `RT_Cloud_Volume` |
| | MediaTexture | MT | | `MT_Video` |
| UI | UserWidget | WBP | | `WBP_MainMenu` |
| | Font | Font | | `Font_Default` |
| | EditorUtilityWidget | EUW | | `EUW_MyTool` |
| AI | BehaviorTree | BT | | `BT_Enemy` |
| | BlackboardData | BB | | `BB_Enemy` |
| | BTDecorator | BTD | | `BTD_IsAlive` |
| | BTService | BTS | | `BTS_UpdateTarget` |
| | AIController | AIC | | `AIC_Enemy` |
| | EnvQuery | EQS | | `EQS_FindCover` |
| Data | DataTable | DT | | `DT_Items` |
| | DataAsset | DA | | `DA_Config` |
| | CurveFloat | Curve | Float | `Curve_Damage_Float` |
| | CurveVector | Curve | Vector | `Curve_Path_Vector` |
| | CurveLinearColor | Curve | Color | `Curve_Fade_Color` |
| Physics | PhysicsAsset | PHYS | | `PHYS_Character` |
| | PhysicalMaterial | PM | | `PM_Metal` |
| Input | InputAction | IA | | `IA_Jump` |
| | InputMappingContext | IMC | | `IMC_Default` |
| Sequencer | LevelSequence | LS | | `LS_Intro` |
| | CameraAnimationSequence | CA | | `CA_Shake` |
| Level | World | MAP | | `MAP_Level1` |
| Landscape | LandscapeLayerInfoObject | LL | | `LL_Grass` |
| | LandscapeGrassType | LG | | `LG_Meadow` |
| Foliage | FoliageType | FT | | `FT_Tree` |
| Paper2D | PaperSprite | SPR | | `SPR_Player` |
| | PaperFlipbook | FB | | `FB_Walk` |
| | PaperTileSet | TS | | `TS_Ground` |
| | PaperTileMap | TM | | `TM_Level1` |

---

## 2. Usage

### 2.1 Auto Rename

| Option | Trigger |
|--------|---------|
| Auto Rename on Create | Automatically renames when assets are created or imported |
| Auto Rename on Rename | Automatically corrects names after manual renaming |

**Example**: Create Blueprint `MyCharacter` → Auto renamed to `BP_MyCharacter`

### 2.2 Batch Rename via Context Menu

**Asset Context Menu**
1. Select one or more assets
2. Right-click → **Auto Rename**

**Folder Context Menu**
1. Right-click on a folder
2. Select **Auto Rename All Assets**

### 2.3 Smart Processing Strategy

Folder batch rename uses a three-tier progressive strategy:

| Phase | Condition | Processing |
|-------|-----------|------------|
| Batch Fast | Total >= 100 | Split into 10 batches |
| Retry Failed | Any batch failed | Merge failed batches and retry once |
| Individual | Still failed | Process one by one with detailed errors |

When total < 100, directly uses individual processing.

### 2.4 Result Dialog

A result window appears after completion with three tabs:
- **Success**: Successfully renamed assets
- **Error**: Failed assets with error reasons
- **Skipped**: Skipped assets with reasons

**Common Errors**:
| Error Message | Description |
|---------------|-------------|
| Source control not connected | Source control is not connected |
| Checked out by another user | File is checked out by another user |
| File is read-only | File is read-only |
| Target file already exists | Target file already exists |
| Asset has unsaved changes | Asset has unsaved changes |

---

## 3. FAQ

**Q: Will asset references break after renaming?**

No. Uses official UE API, automatically handles references and creates redirectors.

**Q: Can I undo?**

Yes, Ctrl+Z undo is supported.

**Q: Why are some assets not renamed?**

1. Already has standard name
2. No naming rule configured for that asset type
3. Path excluded by filter rules
4. File is read-only and cannot be checked out

**Q: How to add custom rules?**

Settings → Asset Type Naming Rules → Click + to add new rule.

**Q: What if batch rename shows many unknown errors?**

Try fixing redirectors first: Content Browser → Right-click folder → Fix Up Redirectors in Folder, then run rename again.

---

## Author

Copyright (c) 2025
Author: mengzhishanghun
Email: mengzhishanghun@outlook.com
 
