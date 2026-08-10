> **Note for the repository owner:** this file is written for AI coding
> assistants (and any new contributor) so they don't have to rediscover how
> this project fits together. Nothing here changes how the program works.

# Razor Enhanced — repository guide

Razor Enhanced is a Windows assistant for Ultima Online. It is a .NET
Framework WinForms application that attaches to a UO client, plus a Python
(IronPython) and UOSteam scripting engine layered on top. This repository is
a private personal fork of the upstream `UltimaTools/RazorEnhanced` project.

## Which branch matters

**Work on `release/1.0` unless told otherwise.** It is the live line.

| Branch | State |
|---|---|
| `release/1.0` | **Active.** Current version, latest work. Base new branches here. |
| `release/0.8` | Also recent, despite the name — it carries version `1.0.0.13` and a handful of commits that `release/1.0` does not have. Not dead; see below. |
| `0.9.0` | **Stale dead end** (last touched 2023) — but it is the repository's *default branch*, so fresh clones and CI checkouts land here by mistake. Do not build on it. |
| `release/0.7`, `release/0.6`, `archive/*`, `Orion` | Historical. |
| `pathfinding`, `ScriptEngine`, `UOSEnginePlus`, `experiment.CUO.IO`, `py-modules-builtin-patch-alternative`, `roslyn-defunct` | Old experiments. |

`release/0.8` vs `release/1.0`: `release/0.8` contains an "Agent Status Gump"
in-game HUD and some OSI login validation work. `release/1.0` had the same gump
merged and then **reverted** ("revert 1.0.0.13 because it was breaking people").
So `release/0.8` is not simply behind — it holds code that was deliberately
backed out of the live line. Don't merge it back without understanding why it
was reverted.

## Version number

One place only:

```
Razor/Properties/AssemblyInfo.cs
[assembly: AssemblyVersion("1.0.0.14")]
```

Both the CI workflows and the runtime (`AutoDoc.GetAssemblyVersion()`) read it
from there. Bumping that line is all that is needed to change the version.

Note the directory name: on `release/1.0` the main project lives in `Razor/`.
On the stale `0.9.0` branch it was renamed to `RazorEnhanced/`, which is why
scripts copied from that branch have wrong paths.

## Layout

```
Razor.sln                    the Windows solution — build this
Razor.Linux.sln              managed-only subset, no C++ projects
Razor/                       main app. AssemblyName = RazorEnhanced, WinExe, x86
  Client/                    per-client glue: ClassicUO.cs, OSIClient.cs, UOAssist.cs
  Core/                      Player, Item, Mobile, engine internals
  Network/                   packet handlers
  RazorEnhanced/             the scripting API surface + HotKey.cs, Settings.cs
  UI/                        WinForms screens (Razor.cs is the giant main form)
  Properties/AssemblyInfo.cs the version number
  app.config                 becomes RazorEnhanced.exe.config in the output
UltimaSDK/                   Ultima.dll — reads UO's client data files (art, maps)
FastColoredTextBox/          the script editor control
Crypt/                       C++ → Crypt.dll. Hooks the classic OSI client
Loader/                      C++ → Loader.dll. Injects Crypt.dll into the client
BuildSupplementalFiles/      data + prebuilt native DLLs copied next to the exe
dokuwiki/                    documentation content, not built
Makefile, Makefile.windows,  convenience wrappers around msbuild
  Makefile.linux
CustomIntermediateDirs.props redirects obj/ output into TEMP
```

Two supported clients: **ClassicUO** (the modern open-source client — Razor
loads as a plugin) and **OSI** (the original client — Razor injects
`Crypt.dll` via `Loader.dll` and hooks its window messages). Code paths for
both live side by side in `Razor/Client/`.

## Building

Windows only for a full build — `Crypt` and `Loader` are C++ projects that
need the MSVC toolchain (`v143`, i.e. Visual Studio 2022).

```
nuget restore Razor.sln
msbuild /m /p:Configuration=Debug /p:Platform="Any CPU" Razor.sln
```

Everything lands in **`bin/Win32/Debug/`**. Run `RazorEnhanced.exe` from there.

Things that are easy to get wrong:

- **`Debug` is the shipping configuration.** Upstream releases are built from
  `Debug`, not `Release`. Keep it that way unless you have a reason.
- **`Platform="Any CPU"` is not what it sounds like.** At solution level it
  maps `Razor` → `x86`, `Crypt`/`Loader` → `Win32`, and the managed libraries
  → `Any CPU`. All of them write into `bin\Win32\<Configuration>\`. Passing
  `x86` or `Win32` instead will not resolve for every project.
- **`nuget restore` must run before `msbuild`.** These are old-style
  `packages.config` projects, not `PackageReference`, so `msbuild -restore`
  is not enough. `Razor.csproj` also hard-errors if certain package `.targets`
  files are missing.
- **There are no git submodules** in this repository, on any branch. If a
  tool or instruction tells you to initialise them, there is nothing to do.

Linux/macOS can build the managed subset only, via `Razor.Linux.sln` or
`make` (see `Makefile.linux`). The result cannot attach to the OSI client
because `Crypt.dll` and `Loader.dll` are Windows-native.

## What the app needs next to the exe

`RazorEnhanced.exe` on its own will not start. A `PostBuild` target in
`Razor/Razor.csproj` assembles the rest of the payload after every build:

1. copies the IronPython standard library out of the restored
   `packages/IronPython.StdLib.*` folder into `bin/Win32/Debug/Lib/`
2. copies everything in `BuildSupplementalFiles/` next to the exe

So a working folder contains, at minimum:

| Item | Comes from |
|---|---|
| `RazorEnhanced.exe`, `RazorEnhanced.exe.config` | the `Razor` project |
| `Ultima.dll`, `FastColoredTextBox.dll` | the other managed projects |
| `Crypt.dll`, `Loader.dll` | the C++ projects (native) |
| `uo.dll`, `zlib.dll`, `UOMod.dll` | prebuilt native DLLs in `BuildSupplementalFiles/` |
| `Lib/` | IronPython standard library, copied from NuGet at build time |
| `Config/`, `Definitions/`, `Language/`, `Scripts/` | `BuildSupplementalFiles/` |
| `NLog.config` | `BuildSupplementalFiles/` |
| third-party DLLs from ~90 NuGet packages (IronPython, NLog, Accord, Grpc, WebView2, …) | NuGet, copied by msbuild |

If you package a build, package the **whole** `bin/Win32/Debug` folder. An exe
plus a few DLLs will not run.

## CI workflows

In `.github/workflows/`:

- **`build-latest.yml`** — this fork's build. Runs on every push to
  `release/1.0` and on manual dispatch. Builds on `windows-latest`, verifies
  the output folder actually contains the runtime payload listed above, zips
  it, and publishes it as the single asset on a release tagged `latest`,
  deleting the previous `latest` release and tag first so the download link
  never changes. Uses only `GITHUB_TOKEN`; no secrets to configure.
- **`autobuild.yml`** — inherited from upstream. Compiles on push to
  `release/0.8` and `release/1.0` as a build check. Publishes nothing.
- **`release.yml`** — inherited from upstream, manual trigger only. Tags
  `v<version>`, publishes a versioned release, and then pushes updates to the
  upstream `UltimaTools/razorenhanced.github.io` site repo. **It will fail on
  this fork** — it needs a `DEPLOYMENT_TOKEN` with write access to a repo we
  do not own. Do not run it. `build-latest.yml` replaces it for our purposes.
- **`release-deprecated.yml`** — dead, kept for reference.

## Conventions and gotchas

- Target framework is **.NET Framework 4.7.2** (README says 4.8; the project
  files say `v4.7.2`). This is not .NET Core / .NET 5+.
- Some comments in the codebase are in Italian — a legacy of the project's
  history. Leave them alone unless you are editing that line anyway.
- `Razor/UI/Razor.cs` is a ~11,000-line designer-generated form. Avoid
  reformatting it; diffs there are unreviewable.
- Hotkeys are dispatched in `Razor/RazorEnhanced/HotKey.cs`. Keys arrive from
  ClassicUO as SDL keycodes (translated by `Win32Platform.MapKey` in
  `Razor/Platform.cs`) and from the OSI client as Windows virtual-key codes
  via the `Crypt.dll` message hook.
- Settings persist as JSON per profile via `Razor/RazorEnhanced/Settings.cs`.
- The user-facing changelog is **not** in this repo. The in-app ChangeLog
  window loads `https://raw.githubusercontent.com/UltimaTools/razorenhanced.github.io/main/changelog.html`.
