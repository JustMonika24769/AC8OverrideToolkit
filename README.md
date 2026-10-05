<div align="center">

# AC8 Override Toolkit

面向 ACE COMBAT 8 的 UE5.4 IoStore 资源覆盖构建工具

[简体中文](README.md) | [English](README_EN.md)

</div>

> [!WARNING]
> 本工具仅用于离线单人战役的 Mod 开发与测试。请勿在多人模式、联网活动、排行榜或启用 EAC 的在线进程中使用。

`AC8 Override Toolkit` 面向 ACE COMBAT 8 `1.1.2.0`，用于构建通用的 UE5.4 IoStore 资源覆盖包。它将资源提取、路径保留、修改、重新打包、回读验证和安装整理为可重复的项目工作流，并且不会修改游戏原始的 `pakchunk0-Windows.*` 文件。

运行自定义容器仍需要 UE4SS 与离线 EAC 启动配置。

## 目录

- [Loader 与 Toolkit](#loader-与-toolkit)
- [功能与限制](#功能与限制)
- [安装准备](#安装准备)
- [快速开始](#快速开始)
- [项目清单](#项目清单)
- [工作区与验证](#工作区与验证)
- [加载顺序与冲突](#加载顺序与冲突)
- [安全与版本限制](#安全与版本限制)
- [问题排查](#问题排查)

## Loader 与 Toolkit

### AC8OverrideLoader

`AC8OverrideLoader` 是实际实现资源覆盖的运行时核心。它通过 UE4SS 在游戏挂载官方 `pakchunk0-Windows.utoc` 后临时允许未签名容器，并递归挂载 `AC8OverrideLoader\payloads` 任意子目录中的有效 UE5.4 `.utoc`。

Loader 不区分导弹、贴图或数据表，也不要求容器必须由本工具生成。只要资源路径和容器格式正确，它就能加载任意资源覆盖包。

如果已经拥有制作完成且未加密的 `.utoc/.ucas/.pak` 文件集，无需运行 Toolkit。将同名文件放在同一目录中：

```text
Game/Binaries/Win64/ue4ss/Mods/AC8OverrideLoader/
  dlls/main.dll
  payloads/MyMod/MyMod_P.utoc
  payloads/MyMod/MyMod_P.ucas
  payloads/MyMod/MyMod_P.pak
```

然后在 UE4SS 的 `mods.txt` 中启用 `AC8OverrideLoader`。`.utoc` 与 `.ucas` 是 Loader 接受该文件集的必需文件；如果成品包含 `.pak`，也应保持同名并放在同一目录。

### AC8OverrideToolkit

`AC8OverrideToolkit` 是可选的开发工具，用于处理[相关讨论](https://www.nexusmods.com/acecombat8wingsoftheve/mods/24?tab=posts)中提到的主要难点：从游戏的加密 IoStore 中定位和提取资源、执行可表达的修改、重新打包并进行回读验证。

Toolkit 内置的结构化编辑能力有限，但这不会限制 Loader 可加载的资源类型。对于 Toolkit 无法编辑的资源，可以使用 Unreal Editor 或其他专用工具生成 cooked 文件，再交给 Toolkit 打包和验证；也可以将已经完成的容器直接放入 Loader。

## 功能与限制

### 主要功能

- 从游戏 IoStore 中提取项目清单指定的资源。
- 保留 `/Game/...` 对应的 Legacy 目录结构，以及 `.uasset/.uexp/.ubulk/.uptnl` 伴随文件。
- 将外部工具生成的 cooked 资源覆盖到 staged 工作区。
- 结构化修改 Blueprint/CDO `NormalExport` 与 `DataTable` 中已有的常见标量属性。
- 生成独立的 `.utoc/.ucas/.pak` 文件集，并执行 `retoc verify`。
- 执行 Legacy -> IoStore -> Legacy 回读，逐字节比较导出与 BulkData 载荷。
- 通过项目隔离的构建清单完成安装、验证和卸载。
- 使用通用 UE4SS Loader 递归发现并按完整路径顺序挂载多个项目容器。

### 结构化编辑支持

目前支持已有的以下属性：

| 类型 | 支持范围 |
| --- | --- |
| 布尔与整数 | `bool`、`byte`、`int`、`int64`、`uint32`、`uint64` |
| 浮点数 | `float`、`double` |
| 文本与枚举 | `string`、`name`、`enum` |
| 复合路径 | 嵌套 Struct 与数组索引路径 |

工具不会猜测或创建未知的 Unreal 属性。

### 需要外部工具的资源

纹理、材质、静态/骨骼网格、动画、音频和 Niagara 等资源通常需要 Unreal Editor、专用导入器或对应格式工具来生成兼容的 cooked 文件。将生成结果按 Legacy 路径放入 `replacementRoots` 后，Toolkit 可负责打包与回读验证。

Toolkit 不会将 PNG、FBX、WAV 等源文件自动烹饪成游戏资源。

## 安装准备

### 包含的文件

| 路径 | 用途 |
| --- | --- |
| `AC8OverrideToolkit.exe` | 命令行工具 |
| `config.json` | 全局配置 |
| `project.example.json` | 项目清单示例 |
| `Mappings.usmap` | UE5.4 属性映射 |
| `tools/` | 提取与打包依赖目录 |
| `assets/AC8OverrideLoader/main.dll` | UE4SS 运行时 Loader |

必须完整解压发布包，不能只复制 EXE。

### 配置

`config.json` 中的关键设置：

| 字段 | 说明 |
| --- | --- |
| `gamePath` | 留空时自动从 Steam 库查找；失败时填写完整游戏目录 |
| `gameVersion` | 当前仅支持 `1.1.2.0` |
| `aesKey` | 填写合法取得的 32 字节十六进制密钥；设为 `auto` 时需要 `tools/aes-dumper.exe` |
| `workspaceRoot` | 工作区目录，默认位于工具目录下 |
| `outputRoot` | 构建输出目录，默认位于工具目录下 |
| `mappingsPath` | `Mappings.usmap` 的路径 |

公开发布包不会附带 Oodle 或 AES 扫描器。开发者必须自行提供合法取得的 `tools/oo2core_9_win64.dll`，并自行提供 AES 密钥或扫描器。

- AES 扫描器可从 [aes-dumper-rs](https://github.com/chadlrnsn/aes-dumper-rs) 获取。
- `oo2core_9_win64.dll` 通常可从自己合法拥有的 Unreal Engine 或其他 UE5 游戏开发环境中取得。

出于安全与许可原因，本项目不会分发 Oodle 或 AES 扫描器。更多信息参见 [`DISTRIBUTION.md`](DISTRIBUTION.md) 与 [`THIRD_PARTY.md`](THIRD_PARTY.md)。

## 快速开始

在新的项目目录中打开终端并初始化：

```powershell
AC8OverrideToolkit.exe init
```

编辑生成的 `project.json`，然后依次检查环境、搜索资源、提取、检查并构建：

```powershell
AC8OverrideToolkit.exe doctor
AC8OverrideToolkit.exe search BP_plwp_msl_a0
AC8OverrideToolkit.exe extract
AC8OverrideToolkit.exe inspect /Game/Blueprints/Weapons/MSL/Player/Msl/BP_plwp_msl_a0
AC8OverrideToolkit.exe build
AC8OverrideToolkit.exe verify
```

测试通过后安装：

```powershell
AC8OverrideToolkit.exe install
```

卸载当前项目：

```powershell
AC8OverrideToolkit.exe uninstall
```

如果工具与项目文件不在同一目录，请追加：

```text
--config "工具目录\config.json" --project "项目目录\project.json"
```

## 项目清单

`resources` 中的每一项描述一个资源包：

```json
{
  "package": "/Game/Path/MyAsset",
  "extract": true,
  "required": true
}
```

`/Game/...` 会自动映射到 `Live/Content/...uasset`。其他挂载点必须填写明确的 `legacyPath`。`filter` 默认使用包名；当资源重名时，工具仍只会从提取结果中选择清单指定的完整路径。

原资源的依赖元数据由 retoc 保留，未修改的依赖可继续由游戏原始容器提供。需要新增或一同覆盖的依赖，必须作为独立的 `resources` 条目显式列出。

`inspect` 会显示部分硬引用导入，但无法保证从单个资源中静态推断出全部 Unreal 软引用、运行时查找、材质/纹理链或蓝图动态加载依赖。发布者必须在游戏中测试所有使用路径。

### 外部 cooked 文件

`replacementRoots` 是相对于 `project.json` 的目录列表。例如：

```text
replacements/Live/Content/Path/MyAsset.uasset
replacements/Live/Content/Path/MyAsset.uexp
replacements/Live/Content/Path/MyAsset.ubulk
```

`apply`、`build` 与 `install` 会先按相对路径将这些文件覆盖到 staged 工作区。请勿在此处放置 PNG、FBX、工程源文件或未烹饪资源。

### Blueprint/CDO 结构化修改

```json
{
  "kind": "export-property",
  "package": "/Game/Path/BP_Example",
  "export": "Default__BP_Example_C",
  "property": "Settings.Speed",
  "value": 1200.0
}
```

### DataTable 结构化修改

```json
{
  "kind": "datatable-property",
  "package": "/Game/Path/DT_Example",
  "row": "RowName",
  "property": "Nested.Values[0]",
  "value": 1.25
}
```

结构化修改仅作用于已经存在、且 UAssetAPI 能使用当前 `Mappings.usmap` 解析的属性。复杂容器、自定义序列化或未支持类型应使用相应的外部工具处理。

## 工作区与验证

### 目录结构

| 路径 | 用途 |
| --- | --- |
| `workspace/<projectId>/original/` | 原始提取快照，请勿编辑 |
| `workspace/<projectId>/staged/` | 实际构建输入 |
| `workspace/<projectId>/roundtrip/` | 成品反向提取结果 |
| `output/<projectId>/` | 最终容器与构建清单 |

再次运行 `extract --force` 会删除当前项目工作区，包括 staged 修改。工具会拒绝在没有 `--force` 时覆盖非空工作区。

### 验证流程

Legacy/Zen 转换会重建 `.uasset` 头，因此不能要求其逐字节一致。Toolkit 使用以下方式验证构建结果：

1. staged `.uasset` 能使用 UE5.4 映射解析。
2. `retoc verify` 接受生成的 IoStore。
3. 成品反向提取后的 `.uasset` 能再次解析。
4. `.uexp/.ubulk/.uptnl` 等非头部载荷逐字节一致。
5. 构建清单记录 staged 与 round-trip 文件的 SHA-256，供后续 `verify` 复验。

对于 UAssetAPI 尚不支持的资源，可以在 `validation.parseUassets` 中设为 `false`，但这会降低验证强度；载荷字节回读仍应保持开启。

## 加载顺序与冲突

安装文件名格式为 `<loadOrder>_<projectId>_P.*`。通用 Loader 会递归扫描 `payloads`，按不区分大小写的完整路径排序，并从挂载顺序 `1000` 开始依次加载。

Toolkit 安装在同一目录下的项目仍可通过六位 `loadOrder` 前缀控制顺序；手工使用子目录时，目录名也会参与排序。两个项目覆盖同一路径时，结果取决于挂载优先级。发布者应明确声明冲突，不要依赖不透明的组合行为。

旧版 `IoStoreLoaderMod` 与新的 `AC8OverrideLoader` 是两个独立的原生挂钩模块。不要让它们同时覆盖相同资源；发布通用项目时应使用本工具提供的 Loader。

## 安全与版本限制

- 仅支持 ACE COMBAT 8 `1.1.2.0`；Loader 签名不保证兼容后续版本。
- 仅用于离线单人战役；不支持多人、在线服务或排行榜。
- 安装前请退出游戏，并备份 UE4SS 的 `mods.txt`。
- 不修改、删除或替换 EAC 二进制文件。
- 不修改游戏原始 IoStore；卸载只会删除当前项目自己的载荷。
- 不要发布游戏原始资源、AES 密钥或 Epic/Oodle 专有库。

## 问题排查

Loader 日志位于：

```text
Game\Binaries\Win64\ue4ss\Mods\AC8OverrideLoader\AC8OverrideLoader.log
```

正常日志会列出发现的 payload，并为每个容器显示 `custom mount code=0`。如果游戏更新后签名不再唯一，Loader 会拒绝初始化；此时必须重新适配，不能强行复用 DLL。

---

<div align="center">

[简体中文](README.md) | [English](README_EN.md)

</div>
