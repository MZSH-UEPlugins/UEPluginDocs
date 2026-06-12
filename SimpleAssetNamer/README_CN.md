[English](./README.md) | [中文](./README_CN.md)

# Simple Asset Namer 用户指南

支持 UE 5.2 ~ 5.7

## 1. 设置

打开 **编辑 → 项目设置 → Simple Asset Namer**

### 1.1 通用

| 选项 | 说明 |
|------|------|
| Auto Rename on Create | 创建或导入资产时自动重命名 |
| Auto Rename on Rename | 手动重命名后自动修正资产名称 |
| Language | UI 语言：中文 / 英文（默认英文），实时切换所有界面文本、提示信息和日志输出 |

### 1.2 过滤器

| 选项 | 说明 |
|------|------|
| Filter Type | 黑名单（排除指定路径）/ 白名单（仅处理指定路径） |
| Exclude Paths | 黑名单模式下跳过这些路径下的资产 |
| Include Paths | 白名单模式下仅处理这些路径下的资产 |

**默认排除路径**：
- `/Game/__ExternalActors__` - World Partition 自动生成
- `/Game/__ExternalObjects__` - World Partition 自动生成
- `/Game/Developers` - 开发者个人文件夹
- `/Game/Collections` - 收藏夹

### 1.3 分隔符

| 选项 | 说明 | 默认值 |
|------|------|--------|
| Delimiter | 生成名称时使用的分隔符 | `_` |
| Delimiter Detection List | 用于检测现有名称中的分隔符 | `_`、`-`、`.` |
| Legacy Affixes | 用于检测旧命名模式的前缀/后缀列表 | - |

**示例**：输入 `bp-MyActor` 或 `bp.MyActor` → 输出 `BP_MyActor`

### 1.4 命名规则

| 选项 | 说明 |
|------|------|
| Asset Type Naming Rules | 资产类型到前缀/后缀的映射，完全可自定义 |

**命名格式**：`前缀_名称_后缀`（后缀可选）

**默认规则（52 种类型）**：

| 分类 | 资产类型 | 前缀 | 后缀 | 示例 |
|------|---------|------|------|------|
| 游戏逻辑 | GameModeBase | GM | | `GM_MyMode` |
| | GameStateBase | GS | | `GS_MyState` |
| | PlayerController | PC | | `PC_MyController` |
| | PlayerState | PS | | `PS_MyState` |
| | Character | CHAR | | `CHAR_MyCharacter` |
| 蓝图 | Blueprint | BP | | `BP_MyActor` |
| | BlueprintGeneratedClass | BPI | | `BPI_MyInterface` |
| | BlueprintFunctionLibrary | BPFL | | `BPFL_MyLib` |
| | BlueprintMacroLibrary | BPML | | `BPML_MyMacro` |
| | UserDefinedEnum | E | | `E_MyEnum` |
| | UserDefinedStruct | F | | `F_MyStruct` |
| 网格 | StaticMesh | SM | | `SM_Chair` |
| | SkeletalMesh | SK | | `SK_Character` |
| 材质 | Material | M | | `M_Wood` |
| | MaterialInstance | MI | | `MI_Wood_Dark` |
| | MaterialInstanceConstant | MI | | `MI_Wood_Const` |
| | MaterialFunction | MF | | `MF_Blend` |
| | MaterialParameterCollection | MPC | | `MPC_Global` |
| | SubsurfaceProfile | SSP | | `SSP_Skin` |
| 动画 | AnimBlueprint | ABP | | `ABP_Character` |
| | AnimSequence | AS | | `AS_Walk` |
| | AnimMontage | AM | | `AM_Attack` |
| | AnimComposite | AC | | `AC_Combo` |
| | AimOffsetBlendSpace | AO | | `AO_Aim` |
| | BlendSpace | BS | | `BS_Locomotion` |
| | Skeleton | SKEL | | `SKEL_Character` |
| 音频 | SoundWave | S | | `S_Footstep` |
| | SoundCue | SC | | `SC_Ambient` |
| | SoundAttenuation | ATT | | `ATT_Default` |
| | SoundClass | SClass | | `SClass_Music` |
| | SoundMix | Mix | | `Mix_Default` |
| 特效 | ParticleSystem | PS | | `PS_Fire` |
| | NiagaraSystem | NS | | `NS_Smoke` |
| | NiagaraEmitter | NE | | `NE_Spark` |
| 贴图 | Texture2D | T | | `T_Wood_D` |
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
| 数据 | DataTable | DT | | `DT_Items` |
| | DataAsset | DA | | `DA_Config` |
| | CurveFloat | Curve | Float | `Curve_Damage_Float` |
| | CurveVector | Curve | Vector | `Curve_Path_Vector` |
| | CurveLinearColor | Curve | Color | `Curve_Fade_Color` |
| 物理 | PhysicsAsset | PHYS | | `PHYS_Character` |
| | PhysicalMaterial | PM | | `PM_Metal` |
| 输入 | InputAction | IA | | `IA_Jump` |
| | InputMappingContext | IMC | | `IMC_Default` |
| 过场 | LevelSequence | LS | | `LS_Intro` |
| | CameraAnimationSequence | CA | | `CA_Shake` |
| 关卡 | World | MAP | | `MAP_Level1` |
| 地形 | LandscapeLayerInfoObject | LL | | `LL_Grass` |
| | LandscapeGrassType | LG | | `LG_Meadow` |
| 植被 | FoliageType | FT | | `FT_Tree` |
| Paper2D | PaperSprite | SPR | | `SPR_Player` |
| | PaperFlipbook | FB | | `FB_Walk` |
| | PaperTileSet | TS | | `TS_Ground` |
| | PaperTileMap | TM | | `TM_Level1` |

---

## 2. 使用方法

### 2.1 自动重命名

| 选项 | 触发条件 |
|------|---------|
| Auto Rename on Create | 创建或导入资产时自动重命名 |
| Auto Rename on Rename | 手动重命名后自动修正名称 |

**示例**：创建蓝图 `MyCharacter` → 自动重命名为 `BP_MyCharacter`

### 2.2 通过右键菜单批量重命名

右键菜单项统一收纳在 **Simple Asset Namer** 二级子菜单下。

**资产右键菜单**
1. 选择一个或多个资产
2. 右键 → **Simple Asset Namer** → **Auto Rename**

**文件夹右键菜单**
1. 右键点击文件夹
2. **Simple Asset Namer** → **Auto Rename All Assets**

### 2.3 智能处理策略

文件夹批量重命名使用三级渐进策略：

| 阶段 | 条件 | 处理方式 |
|------|------|---------|
| 批量快速 | 总数 >= 100 | 分 10 批处理 |
| 重试失败 | 有批次失败 | 合并失败批次重试一次 |
| 逐个处理 | 仍然失败 | 逐个处理并提供详细错误 |

总数 < 100 时直接逐个处理。

### 2.4 结果对话框

完成后显示结果窗口，包含三个标签页：
- **成功**：成功重命名的资产
- **错误**：失败的资产及错误原因
- **跳过**：跳过的资产及原因

**常见错误**：
| 错误信息 | 说明 |
|---------|------|
| Source control not connected | 版本控制未连接 |
| Checked out by another user | 文件被其他用户签出 |
| File is read-only | 文件为只读 |
| Target file already exists | 目标文件已存在 |
| Asset has unsaved changes | 资产有未保存的更改 |

---

## 3. 常见问题

**Q：重命名后资产引用会断吗？**

不会。使用 UE 官方 API，自动处理引用并创建重定向器。

**Q：可以撤销吗？**

可以，支持 Ctrl+Z 撤销。

**Q：为什么有些资产没被重命名？**

1. 已经是标准名称
2. 该资产类型没有配置命名规则
3. 路径被过滤器规则排除
4. 文件为只读且无法签出

**Q：如何添加自定义规则？**

设置 → Asset Type Naming Rules → 点击 + 添加新规则。

**Q：批量重命名显示很多未知错误怎么办？**

先修复重定向器：内容浏览器 → 右键文件夹 → Fix Up Redirectors in Folder，然后重新运行重命名。

---

## 支持

如有问题或反馈，请在 Fab 产品页面留言。
