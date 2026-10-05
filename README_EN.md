<div align="center">

# AC8 Override Toolkit

A UE5.4 IoStore override build toolkit for ACE COMBAT 8

[简体中文](README.md) | [English](README_EN.md)

</div>

> [!WARNING]
> This toolkit is intended only for offline single-player mod development and testing. Do not use it in multiplayer, online events, leaderboards, or any online process with EAC enabled.

`AC8 Override Toolkit` targets ACE COMBAT 8 `1.1.2.0` and builds general-purpose UE5.4 IoStore resource overrides. It turns extraction, path preservation, modification, repacking, round-trip validation, and installation into a repeatable project workflow without modifying the game's original `pakchunk0-Windows.*` files.

Running custom containers still requires UE4SS and an offline EAC launch configuration.

## Contents

- [Loader vs. Toolkit](#loader-vs-toolkit)
- [Features and limitations](#features-and-limitations)
- [Setup](#setup)
- [Quick start](#quick-start)
- [Project manifest](#project-manifest)
- [Workspace and validation](#workspace-and-validation)
- [Load order and conflicts](#load-order-and-conflicts)
- [Safety and version limits](#safety-and-version-limits)
- [Troubleshooting](#troubleshooting)

## Loader vs. Toolkit

### AC8OverrideLoader

`AC8OverrideLoader` is the runtime component that performs resource overrides. Through UE4SS, it temporarily permits unsigned containers after the game mounts the official `pakchunk0-Windows.utoc`, then recursively mounts valid UE5.4 `.utoc` files from any subdirectory under `AC8OverrideLoader\payloads`.

The Loader does not distinguish between missiles, textures, or data tables, and containers do not have to be produced by this toolkit. It can load any resource override whose asset paths and container format are correct.

If you already have a finished, unencrypted `.utoc/.ucas/.pak` set, you do not need the Toolkit. Place the matching files together:

```text
Game/Binaries/Win64/ue4ss/Mods/AC8OverrideLoader/
  dlls/main.dll
  payloads/MyMod/MyMod_P.utoc
  payloads/MyMod/MyMod_P.ucas
  payloads/MyMod/MyMod_P.pak
```

Then enable `AC8OverrideLoader` in UE4SS's `mods.txt`. The `.utoc` and `.ucas` files are required for the Loader to accept the set. If the finished set includes a `.pak`, keep the same base name and place it in the same directory.

### AC8OverrideToolkit

`AC8OverrideToolkit` is an optional development tool that addresses the main challenges described in the [related discussion](https://www.nexusmods.com/acecombat8wingsoftheve/mods/24?tab=posts): locating and extracting assets from the game's encrypted IoStore, applying expressible changes, repacking them, and validating the result through a round trip.

The Toolkit's built-in structured editing is intentionally limited, but that does not restrict the resource types the Loader can load. For resources the Toolkit cannot edit, use Unreal Editor or a specialized tool to create cooked files, then let the Toolkit package and validate them. You can also place an already finished container directly into the Loader.

## Features and limitations

### Core features

- Extract resources listed in a project manifest from the game's IoStore.
- Preserve the Legacy directory corresponding to `/Game/...`, including `.uasset/.uexp/.ubulk/.uptnl` sidecar files.
- Overlay cooked resources produced by external tools onto the staged workspace.
- Apply structured edits to existing common scalar properties in Blueprint/CDO `NormalExport` objects and `DataTable` rows.
- Build standalone `.utoc/.ucas/.pak` sets and run `retoc verify`.
- Perform a Legacy -> IoStore -> Legacy round trip and compare export and BulkData payloads byte for byte.
- Install, verify, and uninstall through project-isolated build manifests.
- Use a general-purpose UE4SS Loader to discover multiple project containers recursively and mount them in full-path order.

### Structured editing support

The following existing property types are currently supported:

| Category | Supported values |
| --- | --- |
| Boolean and integer | `bool`, `byte`, `int`, `int64`, `uint32`, `uint64` |
| Floating point | `float`, `double` |
| Text and enum | `string`, `name`, `enum` |
| Compound paths | Nested structs and array index paths |

The Toolkit never guesses or creates unknown Unreal properties.

### Resources that require external tools

Textures, materials, static or skeletal meshes, animations, audio, Niagara assets, and similar resources usually require Unreal Editor, a dedicated importer, or a format-specific tool to produce compatible cooked files. Place those outputs under `replacementRoots` using their Legacy paths, and the Toolkit can package and round-trip validate them.

The Toolkit does not automatically cook source files such as PNG, FBX, or WAV into game assets.

## Setup

### Included files

| Path | Purpose |
| --- | --- |
| `AC8OverrideToolkit.exe` | Command-line application |
| `config.json` | Global configuration |
| `project.example.json` | Example project manifest |
| `Mappings.usmap` | UE5.4 property mappings |
| `tools/` | Extraction and packing dependencies |
| `assets/AC8OverrideLoader/main.dll` | UE4SS runtime Loader |

Extract the complete release package. Copying only the EXE is not sufficient.

### Configuration

Important fields in `config.json`:

| Field | Description |
| --- | --- |
| `gamePath` | Leave empty to search Steam libraries automatically; set the full game directory if detection fails |
| `gameVersion` | Currently only `1.1.2.0` is supported |
| `aesKey` | A legally obtained 32-byte hexadecimal key; `auto` requires `tools/aes-dumper.exe` |
| `workspaceRoot` | Workspace directory, relative to the Toolkit directory by default |
| `outputRoot` | Build output directory, relative to the Toolkit directory by default |
| `mappingsPath` | Path to `Mappings.usmap` |

Public releases do not include Oodle or an AES scanner. Developers must provide their own legally obtained `tools/oo2core_9_win64.dll` and either an AES key or a scanner.

- The AES scanner is available from [aes-dumper-rs](https://github.com/chadlrnsn/aes-dumper-rs).
- `oo2core_9_win64.dll` can usually be obtained from an Unreal Engine installation or another UE5 game development environment that you legally own.

For security and licensing reasons, this project does not distribute Oodle or an AES scanner. See [`DISTRIBUTION.md`](DISTRIBUTION.md) and [`THIRD_PARTY.md`](THIRD_PARTY.md) for details.

## Quick start

Open a terminal in a new project directory and initialize it:

```powershell
AC8OverrideToolkit.exe init
```

Edit the generated `project.json`, then check the environment, find and extract the resource, inspect it, and build the project:

```powershell
AC8OverrideToolkit.exe doctor
AC8OverrideToolkit.exe search BP_plwp_msl_a0
AC8OverrideToolkit.exe extract
AC8OverrideToolkit.exe inspect /Game/Blueprints/Weapons/MSL/Player/Msl/BP_plwp_msl_a0
AC8OverrideToolkit.exe build
AC8OverrideToolkit.exe verify
```

Install after validation succeeds:

```powershell
AC8OverrideToolkit.exe install
```

Uninstall the current project:

```powershell
AC8OverrideToolkit.exe uninstall
```

If the Toolkit and project files are in different directories, append:

```text
--config "path\to\toolkit\config.json" --project "path\to\project\project.json"
```

## Project manifest

Each entry in `resources` describes one resource package:

```json
{
  "package": "/Game/Path/MyAsset",
  "extract": true,
  "required": true
}
```

`/Game/...` maps automatically to `Live/Content/...uasset`. Other mount points require an explicit `legacyPath`. `filter` defaults to the package name; if multiple resources share a name, the Toolkit still selects only the full path specified by the manifest.

retoc preserves dependency metadata from the original resource, so unmodified dependencies can continue to come from the game's original containers. Any new dependency, or any dependency that must also be overridden, must be listed explicitly as a separate `resources` entry.

`inspect` shows some hard-reference imports, but a single resource cannot reliably reveal every Unreal soft reference, runtime lookup, material/texture chain, or dynamic Blueprint load through static analysis. Publishers must test every relevant path in the game.

### External cooked files

`replacementRoots` is a list of directories relative to `project.json`. For example:

```text
replacements/Live/Content/Path/MyAsset.uasset
replacements/Live/Content/Path/MyAsset.uexp
replacements/Live/Content/Path/MyAsset.ubulk
```

`apply`, `build`, and `install` first overlay these files onto the staged workspace using their relative paths. Do not place PNG, FBX, project source files, or uncooked resources here.

### Blueprint/CDO structured edit

```json
{
  "kind": "export-property",
  "package": "/Game/Path/BP_Example",
  "export": "Default__BP_Example_C",
  "property": "Settings.Speed",
  "value": 1200.0
}
```

### DataTable structured edit

```json
{
  "kind": "datatable-property",
  "package": "/Game/Path/DT_Example",
  "row": "RowName",
  "property": "Nested.Values[0]",
  "value": 1.25
}
```

Structured edits only modify properties that already exist and that UAssetAPI can parse with the current `Mappings.usmap`. Use the appropriate external tool for complex containers, custom serialization, or unsupported types.

## Workspace and validation

### Directory layout

| Path | Purpose |
| --- | --- |
| `workspace/<projectId>/original/` | Original extraction snapshot; do not edit |
| `workspace/<projectId>/staged/` | Actual build input |
| `workspace/<projectId>/roundtrip/` | Files extracted back from the finished container |
| `output/<projectId>/` | Final containers and build manifest |

Running `extract --force` again deletes the current project's workspace, including staged changes. The Toolkit refuses to overwrite a non-empty workspace without `--force`.

### Validation process

Legacy/Zen conversion rebuilds the `.uasset` header, so it cannot be expected to remain byte-identical. The Toolkit validates a build as follows:

1. The staged `.uasset` parses with the UE5.4 mappings.
2. `retoc verify` accepts the generated IoStore.
3. The `.uasset` extracted back from the final container parses again.
4. Non-header payloads such as `.uexp/.ubulk/.uptnl` match byte for byte.
5. The build manifest records SHA-256 hashes for staged and round-trip files so that later `verify` runs can check them again.

For resources UAssetAPI does not yet support, set `validation.parseUassets` to `false`. This weakens validation, so byte-level payload round-trip checks should remain enabled.

## Load order and conflicts

Installed files use the name format `<loadOrder>_<projectId>_P.*`. The general-purpose Loader scans `payloads` recursively, sorts containers by case-insensitive full path, and mounts them sequentially starting at mount order `1000`.

Projects installed by the Toolkit into the same directory can still use the six-digit `loadOrder` prefix to control ordering. For manually managed subdirectories, directory names also affect sorting. If two projects override the same path, the result depends on mount priority. Publishers should declare conflicts instead of relying on opaque combinations.

The legacy `IoStoreLoaderMod` and the new `AC8OverrideLoader` are separate native hook modules. Do not let both override the same resources; general-purpose project releases should use the Loader supplied with this Toolkit.

## Safety and version limits

- Only ACE COMBAT 8 `1.1.2.0` is supported. Loader signatures are not guaranteed to work with later versions.
- Use this project only for offline single-player. Multiplayer, online services, and leaderboards are not supported.
- Exit the game and back up UE4SS's `mods.txt` before installation.
- The Toolkit does not modify, delete, or replace EAC binaries.
- The Toolkit does not modify the game's original IoStore. Uninstalling removes only the current project's payload.
- Do not distribute original game assets, AES keys, or proprietary Epic/Oodle libraries.

## Troubleshooting

The Loader log is located at:

```text
Game\Binaries\Win64\ue4ss\Mods\AC8OverrideLoader\AC8OverrideLoader.log
```

A normal log lists discovered payloads and reports `custom mount code=0` for each container. If a game update makes a signature non-unique, the Loader refuses to initialize. It must be adapted to the new version; do not force reuse of the DLL.

---

<div align="center">

[简体中文](README.md) | [English](README_EN.md)

</div>
