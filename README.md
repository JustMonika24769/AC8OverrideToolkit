# AC8 Override Toolkit

`AC8 Override Toolkit` 是面向 ACE COMBAT 8 `1.1.2.0` 的通用 UE5.4 IoStore
资源覆盖构建工具。它把资源提取、路径保留、修改、重新打包、回读验证和安装拆成可重复的
项目工作流，不会修改游戏原始 `pakchunk0-Windows.*`。

本工具仅用于离线单人战役开发与测试。不得在多人、联网活动、排行榜或启用 EAC 的在线
进程中使用。运行自定义容器仍需要 UE4SS 和离线 EAC 启动配置。

## 先区分 Loader 与 Toolkit

`AC8OverrideLoader` 才是实现资源覆盖的运行时核心。它使用 UE4SS 在游戏挂载官方
`pakchunk0-Windows.utoc` 后，临时允许未签名容器，并递归挂载
`AC8OverrideLoader\payloads` 下任意子目录中的有效 UE5.4 `.utoc`。Loader 不识别
“导弹”“贴图”或“数据表”，也不要求容器必须由本工具生成；只要资源路径和容器格式正确，
它就可以加载任意资源覆盖包。

已有做好的未加密 `.utoc/.ucas/.pak` 集合时，不需要运行 Toolkit。把同名的三个文件放在
一起，例如：

```text
Game/Binaries/Win64/ue4ss/Mods/AC8OverrideLoader/
  dlls/main.dll
  payloads/MyMod/MyMod_P.utoc
  payloads/MyMod/MyMod_P.ucas
  payloads/MyMod/MyMod_P.pak
```

在 UE4SS 的 `mods.txt` 中启用 `AC8OverrideLoader` 即可。`.utoc` 和 `.ucas` 是 Loader
接受该集合的必需文件；成品带有 `.pak` 时应保持同名并放在同一目录。

`AC8OverrideToolkit` 是可选的开发工具，解决的是引文中真正困难的部分：从游戏加密
IoStore 中找到和提取资源、进行可表达的修改、重新打包并回读验证。它的内置结构化编辑
能力有限，但这不限制 Loader 能加载的资源类型。对于本工具不会编辑的资源，可以使用
Unreal Editor 或其他专用工具生成 cooked 文件，再交给 Toolkit 打包验证，或直接把已经
完成的容器放入 Loader。

## 能力范围

- 从游戏 IoStore 提取项目清单中的资源。
- 保留 `/Game/...` 对应的 Legacy 目录和 `.uasset/.uexp/.ubulk/.uptnl` 伴随文件。
- 将外部工具生成的 cooked 资源覆盖到 staged 工作区。
- 结构化修改 Blueprint/CDO `NormalExport` 和 `DataTable` 中已有的常见标量属性。
- 生成独立 `.utoc/.ucas/.pak`，执行 `retoc verify`。
- 执行 Legacy -> IoStore -> Legacy 回读，并逐字节比较导出和 BulkData 载荷。
- 使用项目隔离的构建清单安装、验证和卸载。
- 通用 UE4SS 加载器递归发现并按完整路径顺序挂载多个项目容器。

结构化编辑目前支持已有的 `bool`、`byte`、`int`、`int64`、`uint32`、`uint64`、
`float`、`double`、`string`、`name` 和 `enum` 属性，以及嵌套 Struct 和数组索引路径。
工具不会猜测或创建未知 Unreal 属性。

纹理、材质、静态/骨骼网格、动画、音频和 Niagara 等资源通常需要 Unreal Editor、
专用导入器或对应格式工具生成兼容的 cooked 文件。把生成结果按 Legacy 路径放到
`replacementRoots` 后，本工具负责打包和回读验证；它不会把 PNG、FBX、WAV 等源文件
自动烹饪成游戏资源。

## 文件

```text
AC8OverrideToolkit.exe
config.json
project.example.json
Mappings.usmap
tools/
assets/AC8OverrideLoader/main.dll
```

必须完整解压，不能只复制 EXE。

`config.json` 中 `gamePath` 留空时会从 Steam 库自动查找游戏；查找失败时填写完整游戏
目录。`aesKey` 可填写合法取得的 32 字节十六进制密钥；设为 `auto` 时需要
`tools/aes-dumper.exe`。工作区和输出目录默认位于工具目录，可改为其他可写路径。

公开发布包不会附带 Oodle 或 AES 扫描器。开发者必须自行提供合法取得的
`tools/oo2core_9_win64.dll`，并自行提供 AES 密钥或扫描器。包含这些本地依赖的
`LocalFull` 包只供当前开发环境验证，不可原样公开分发；详见 `DISTRIBUTION.md`。

## 快速开始

在一个新的项目目录打开终端：

```text
AC8OverrideToolkit.exe init
```

编辑生成的 `project.json`，然后执行：

```text
AC8OverrideToolkit.exe doctor
AC8OverrideToolkit.exe search BP_plwp_msl_a0
AC8OverrideToolkit.exe extract
AC8OverrideToolkit.exe inspect /Game/Blueprints/Weapons/MSL/Player/Msl/BP_plwp_msl_a0
AC8OverrideToolkit.exe build
AC8OverrideToolkit.exe verify
```

测试通过后安装：

```text
AC8OverrideToolkit.exe install
```

卸载当前项目：

```text
AC8OverrideToolkit.exe uninstall
```

工具和项目文件不在同一目录时，追加：

```text
--config "工具目录\config.json" --project "项目目录\project.json"
```

## 项目清单

`resources` 中每项描述一个资源包：

```json
{
  "package": "/Game/Path/MyAsset",
  "extract": true,
  "required": true
}
```

`/Game/...` 自动映射到 `Live/Content/...uasset`。其他挂载点必须填写明确的
`legacyPath`。`filter` 默认使用包名；资源名重名时仍会从提取结果中只选取清单指定的
完整路径。

原资源中的依赖元数据会由 retoc 保留，未修改的依赖可继续由游戏原始容器提供。需要新增
或一同覆盖的依赖必须作为单独的 `resources` 条目显式列出。`inspect` 会显示部分硬引用
导入，但 Unreal 软引用、运行时查找、材质/纹理链和蓝图动态加载无法保证从单个资源静态
推断完整依赖。发布者必须在游戏中覆盖所有使用路径进行测试。

### 外部 cooked 文件

`replacementRoots` 是相对于 `project.json` 的目录列表。例如：

```text
replacements/Live/Content/Path/MyAsset.uasset
replacements/Live/Content/Path/MyAsset.uexp
replacements/Live/Content/Path/MyAsset.ubulk
```

`apply`、`build` 和 `install` 会先把这些文件按相对路径覆盖到 staged。不要在这里放
PNG、FBX、工程源文件或未烹饪资源。

### 结构化修改

Blueprint/CDO：

```json
{
  "kind": "export-property",
  "package": "/Game/Path/BP_Example",
  "export": "Default__BP_Example_C",
  "property": "Settings.Speed",
  "value": 1200.0
}
```

DataTable：

```json
{
  "kind": "datatable-property",
  "package": "/Game/Path/DT_Example",
  "row": "RowName",
  "property": "Nested.Values[0]",
  "value": 1.25
}
```

结构化修改只修改已存在且 UAssetAPI 能用当前 `Mappings.usmap` 解析的属性。复杂容器、
自定义序列化或未支持类型应使用相应外部工具。

## 工作区

```text
workspace/<projectId>/original/   原始提取快照，不要编辑
workspace/<projectId>/staged/     实际构建输入
workspace/<projectId>/roundtrip/  成品反向提取结果
output/<projectId>/               最终容器和构建清单
```

再次运行 `extract --force` 会删除当前项目工作区，包括 staged 修改。工具拒绝在没有
`--force` 时覆盖非空工作区。

## 验证含义

`.uasset` 头在 Legacy/Zen 转换中会被重建，不能要求逐字节相同。因此验证方式是：

1. staged `.uasset` 能使用 UE5.4 映射解析；
2. `retoc verify` 接受生成的 IoStore；
3. 成品反向提取后的 `.uasset` 再次可解析；
4. `.uexp/.ubulk/.uptnl` 等非头部载荷逐字节一致；
5. 构建清单记录 staged 和 round-trip 的 SHA-256，供后续 `verify` 复验。

对 UAssetAPI 尚不支持的资源，可在 `validation.parseUassets` 设为 `false`，但这会降低验证
强度；载荷字节回读仍应保持开启。

## 加载顺序与冲突

安装文件名为 `<loadOrder>_<projectId>_P.*`。通用加载器递归扫描 `payloads`，按不区分
大小写的完整路径排序，并从挂载顺序 `1000` 开始依次加载。由 Toolkit 安装在同一目录
下的项目仍可通过六位 `loadOrder` 前缀控制顺序；手工使用子目录时，目录名也会参与排序。
两个项目覆盖同一路径时，结果取决于挂载优先级，发布者应声明冲突而不是依赖不透明的
组合行为。

旧的 `IoStoreLoaderMod` 与新的 `AC8OverrideLoader` 是两个原生挂钩模块。不要让它们同时
覆盖相同资源；为通用项目发布时应使用本工具提供的加载器。

## 安全与版本限制

- 只支持 ACE COMBAT 8 `1.1.2.0`，加载器签名不保证兼容更新版本。
- 只用于离线单人战役；多人、在线服务和排行榜不受支持。
- 安装前退出游戏并备份 UE4SS 的 `mods.txt`。
- 不修改、删除或替换 EAC 二进制。
- 不修改游戏原始 IoStore；卸载只删除当前项目自己的载荷。
- 不要发布游戏原始资源、AES 密钥或 Epic/Oodle 专有库。

## 问题排查

加载器日志：

```text
Game\Binaries\Win64\ue4ss\Mods\AC8OverrideLoader\AC8OverrideLoader.log
```

正常日志会列出 payload，并为每个容器显示 `custom mount code=0`。游戏更新后若签名不再
唯一，加载器会拒绝初始化，必须重新适配，不能强行复用 DLL。
