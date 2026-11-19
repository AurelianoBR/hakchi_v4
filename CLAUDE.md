# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

hakchi2 CE: Windows Forms app (.NET Framework 4.8, C#) that adds games/mods to NES/SNES Classic Mini and Famicom Mini consoles over USB (FEL mode for kernel/memboot work, then Clovershell or SSH to the running console). Windows-only build; the post-build steps rely on `tools/busybox.exe` and NSIS.

## Build

Submodules are required (`git submodule update --init --recursive`) — SSH.NET, FelLib, tiny7z, CueSharp, ProgressODoom, SpineGen, LibUsbDotNet, libwdi live under `Libraries/`, `driver/source`, `libwdi`.

```
nuget restore hakchi_gui.sln
msbuild hakchi_gui.sln /p:Configuration=Debug /target:hakchi_gui          # app only
msbuild Installer/Installer.vcxproj /p:SolutionDir=<repo>\ /p:SolutionPath=<repo>\hakchi_gui.sln "/p:Platform=Any CPU"   # app + NSIS installer (what CI runs)
```

Output: `hakchi_gui/bin/Debug/net48/hakchi.exe`. CI (`.github/workflows/dotnet-desktop.yml`) builds on `windows-latest` in Debug, then runs `hakchi.exe --versionFormat {0} --versionFile <path>` to extract the display version.

Build pipeline details worth knowing:
- `ce-data` project has no code; its pre-build runs `tools/downloadfiles` to fetch external assets (romfiles.xml, hakchi.hmod, bluetooth.hmod, detached-fallback, classic_driver.exe, puttytel) into `GeneratedData/`, and tars `hakchi_gui/data/{libretro_cores,desktop_entries,original_art}` and `Licenses/`. Network access needed on first build.
- `hakchi_gui` pre-build writes `commit.txt`, `tag.txt`, `git-commits-since-last-tag.txt` (via `tools/git*` scripts) and an empty `ApiKeys/TheGamesDB.txt`; post-build copies `GeneratedData/` into the output dir. A real TheGamesDB key goes in `hakchi_gui/ApiKeys/TheGamesDB.txt` (CI injects it from a secret).
- Costura.Fody embeds referenced assemblies into `hakchi.exe`.
- `Dumper` configuration builds a stripped variant (only basehmods/driver/detached-fallback copied).

## Tests

No unit test suite. `Scrapers.Test` is a netcoreapp3.0 console app that exercises the TheGamesDB scraper manually: `dotnet run --project Scrapers.Test`.

## Running

`hakchi.exe` is portable by default (data next to exe). `/nonportable` (or a `nonportable.flag` file) uses `Documents/hakchi2` as `Program.BaseDirectoryExternal`; `/debug` enables the debug log window. See `Program.Main`.

## Architecture (hakchi_gui)

- `hakchi.cs` — static hub for console state: current `ISystemShell` connection, console type detection, hmod locations, connect/disconnect events. Most features go through `hakchi.Shell`.
- Connections: `Clovershell/` (USB gadget shell protocol), `SshClient/` (SSH over USB-network, devices discovered via mDNS/DNS listeners), `FtpClient/` for file transfer. FEL/memboot handled by `Libraries/FelLib` + `Tasks/MembootTasks.cs`.
- `Tasks/` — all long-running work is a `Tasker.TaskFunc` (`Conclusion fn(Tasker, object)`) chained via `Tasker.AddTask(s)` and run with a progress UI (`TaskerForm`/`TaskerTransferForm`). `SyncTask`, `MembootTasks`, `ModTasks`, `AddGamesTask` etc. are task collections; follow this pattern for new operations.
- `Apps/` — game model. `NesApplication` is the base for everything on the console menu (`NesGame`, `SnesGame`, `FdsGame`, `LibretroGame`, ...); `.desktop` files via `DesktopFile`; system/core mapping in `CoreCollection`, `AppTypeCollection`, `SystemCollection`. `NesMenuCollection` + `NesMenuFolder` build the folder tree.
- `Hmod/` — hakchi mod (hmod) metadata, readme rendering, install/uninstall selection UI. `ModHub/` — downloads mods from remote repositories.
- `Scrapers/` (separate project) — TheGamesDB API client used for game metadata.
- `data/` — bundled DBs (nescarts/snescarts XML, GameGenieDB, libretro core defs, desktop entries, original art).
- `ConfigIni.cs` — persisted user settings.

## Conventions

- Localization: every Form has per-culture `.resx` (ar-SA, de-DE, es, fr-FR, it, ko, pt-BR, sv); strings go in `Properties/Resources*.resx`. When adding UI strings, add to `Resources.resx` (+ `Resources.Designer.cs`); translations optional.
- Line endings enforced by `.gitattributes`: `*.cs`, `*.resx`, `*.sln` CRLF; `*.xml`, `*.desktop`, `data/moon_hashes/*` LF. Preserve them when editing.
- Default branch: `mainline`.
