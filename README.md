# Avaris (UC2 PC) — patch bundle

This folder is a hand-off package: drop these patches onto a clean official UC2 source tree, build, and you should end up with a runnable PC version of Avaris on Windows.

What you're looking at is every source-level change made on top of the UC2 source dump to get into a working state.

## Heads-up before you read further

This work is **vibecoded**: most of the patch was produced by working alongside an LLM coding assistant rather than written line-by-line by hand. That has the implications you'd expect.

- The directions taken (toolchain choice, audio backend, video pipeline, input model, settings UI) are pragmatic and "good enough" rather than ideal. See `DIRECTIONS.md` for the rationale; some of those calls could reasonably have gone the other way.
- Comments, names, and structure are uneven in places. Some files read like polished port work; others read like the first thing that compiled and ran.
- The fixes for the three latent source-dump bugs (placement-new, `FNameDebug` COM, UCC stub-ScriptText reparse) are surgical — they unblock boot but aren't full audits of the surrounding subsystems. Other instances of the same patterns may still be lurking.
- Test coverage is "the game launches and the menu/in-game work on my machine." Anything off the golden path may be fragile.
- Performance has not been profiled. The FFmpeg buffer-copy path in particular trades a per-frame memcpy for stability; that was fine for the menu video but hasn't been pushed.
- Third-party version pinning (FFmpeg DLLs in particular) is documented loosely; if you build against different FFmpeg majors expect symbol mismatches.

If you're using this as a starting point for further work, treat it that way: a runnable baseline, not a finished port.

## Folder layout

The diffs are split one file per patch, mirroring the original source tree under `diff/`:

```
diff/
  UC2004/ALAudio/Inc/ALAudio.h.diff
  UC2004/ALAudio/Src/ALAudioSubsystem.cpp.diff
  UC2004/Engine/Src/UnRender.cpp.diff
  ...
  tools/dump_audio.py.diff
```

Each `.diff` is a standalone unified patch for one file — open it in any diff viewer, apply it on its own with `git apply`, or just read it as plain text.

31 patches total.

## What's in the patch (high-level)

- **Toolchain port fixes** (Core, Editor, Engine, GUI, UCElements) — three latent bugs that prevented the original tree from running on modern Windows: a placement-new optimizer mishandle, a static-init COM deadlock in `FNameDebug`, and a UCC `make` reparse against stub ScriptText for native-noexport classes.
- **New ALAudio module** (`UC2004/ALAudio/`) — OpenAL-backed audio subsystem replacing the audio that wasn't in the source dump (UC2 shipped on Xbox with proprietary audio code that Epic didn't release). Wired in via `avaris.ini` → `AudioDevice=ALAudio.ALAudioSubsystem`.
- **Video on materials via FFmpeg** — `D3DDrv/Src/D3DMoviePlayback.cpp` plus material-pipeline hooks in `UnMaterial.cpp`, `D3DMaterialState.cpp`, `D3DRenderDevice.cpp`. PC menu video runs through an FFmpeg buffer-copy path.
- **Mouse / keyboard support** — `UCGameplay/Classes/UCPlayerInput.uc` plus `DefUser.ini` tuning. Controller path stays intact; KB+M is added on top.
- **VIDEO settings page** — new `UCMenu/Classes/UC2Settings_Video.uc` and main-menu rewiring; saved-volume initialization at boot.

## What's NOT in this bundle (and how to get it)

Compiled binaries, third-party runtimes, and game assets are deliberately excluded. You'll need to (re)build or fetch them yourself:

| What | Files | How to get it |
|---|---|---|
| Module DLLs | `UC2004/System/Core.dll`, `Engine.dll`, `D3DDrv.dll`, `Editor.dll`, `GUI.dll`, `GUIDesigner.dll`, `IpDrv.dll`, `UCElements.dll`, `UCGameplay.dll`, `UWeb.dll`, `UnrealGame.dll`, `UnrealUtil.dll`, `WinDrv.dll`, `Window.dll`, `ALAudio.dll` | Build `UC2004/Avaris.sln` in Visual Studio 2003 SP1. |
| Executables | `UC2004/System/avaris.exe`, `UnrealEd.exe`, `UCC.exe` | Same — produced by `Avaris.sln`. |
| UnrealScript packages | `UC2004/System/UCGameplay.u`, `UCMenu.u` | After the C++ side builds, run `UCC make` from `UC2004/System/`. It will compile the `.uc` sources delivered by this patch. |
| FFmpeg runtime | `UC2004/System/avcodec-62.dll`, `avformat-62.dll`, `avutil-60.dll`, `swresample-6.dll`, `swscale-9.dll` | Build a matching FFmpeg redistributable or download a known-good drop. |
| VC7.1 runtime | `UC2004/System/msvcp71.dll`, `msvcr71.dll` | Ship with Visual Studio 2003 SP1 — copy from the toolchain install or the VS 2003 redist. |
| Game assets | `*.uax` / `*.utx` / `*.usx` / `*.umx` / `*.wav` / etc. | None are touched by this patch — your existing UC2 cooks are reused as-is. |

## Take-off procedure

### 1. Prerequisites

- A clean UC2 source tree. The patches expect the layout to start at `UC2004/...` from your project root, with `tools/` alongside it.
- Visual Studio 2003 SP1 (VC7.1). The toolchain doesn't install cleanly on Windows 10/11, so the recommended setup is an XP guest in VirtualBox.
- (Optional) Python 3 if you plan to use `tools/dump_audio*.py` for audio asset extraction.

### 2. Apply the patches

The diffs are CRLF-encoded to match the Epic source dump's Windows line endings. You apply them with **`git apply`** — it preserves bytes exactly and does not normalize line endings. (You don't need to be in a git repo. `git apply` works fine against a plain source tree as long as `git` is on your PATH, which it will be if you have Git for Windows installed.)

Don't use GNU `patch` on Windows — its text-mode handling rewrites CRLF files to LF on save, which silently corrupts the working tree.

The `--ignore-whitespace` flag is required: a couple of patches contain trailing-whitespace lines and `git apply` is strict about that by default.

**All at once**:

```powershell
# PowerShell, from the project root:
Get-ChildItem -Recurse -Filter *.diff diff |
  ForEach-Object { git apply --ignore-whitespace $_.FullName }
```

```bash
# Linux / git-bash / WSL, from the project root:
find diff -name '*.diff' -exec git apply --ignore-whitespace {} +
```

**One at a time** (if you want to apply a subset):

```
git apply --ignore-whitespace diff/UC2004/Engine/Src/UnRender.cpp.diff
```

If a hunk fails to apply, your starting tree differs from the Epic source dump in that spot — open the failing `.diff` next to the target file and reconcile by hand. Each patch is small and self-contained, so this is rarely more than a few minutes' work per file.

### 3. Build the C++ side

Open `UC2004/Avaris.sln` in VS 2003 SP1 and build the solution. Outputs land in `UC2004/System/`.

**You'll need to do it in two passes** the first time you build a clean tree:

1. **Build `FNameDebugCOM` first** (right-click the project in VS, "Build", or from the command line: `devenv.com Avaris.sln /build "Release" /project FNameDebugCOM`). This runs MIDL on the IDL file and produces `FNameDebugCOM/FNameDebugCOM.h` + `FNameDebugCOM/FNameDebugCOM_i.c`. **`Core` `#include`s those headers and the .sln does not declare the dependency**, so on a clean tree without those generated files, Core's compile fails and the failure cascades to the entire solution (`cannot open Core.lib` on every downstream project).
2. **Build the full solution** (`devenv.com Avaris.sln /build "Release"` or VS → Build → Build Solution). Expected result: 27 succeeded, 0 failed, 7 skipped. The 7 skipped projects are Xbox-targeted (`Gameplay`, `bundler`, `XboxAssetConversion`, `UCMenu`, `DebuggerLaunch`, `UccDepend`) plus `FNameDebugCOM` (skipped at the Release-Win32 config because it's set up under a different config flavour). None of those skipped projects are needed for a runnable Avaris.exe.

Once `FNameDebugCOM`'s MIDL outputs exist on disk, future incremental builds work fine in a single pass.

You may also see a one-line `Project 'XboxLaunch\Src\XboxLaunch.vcproj' failed to open.` message when VS loads the .sln. The Xbox launch project isn't in the source dump and isn't needed for a PC build — the warning is harmless and can be ignored. Removing the entry from `Avaris.sln` is optional cleanup.

### 4. Build the UnrealScript side

From `UC2004/System/`:

```
del UCMenu.u UCGameplay.u
UCC make
```

`UCC make` regenerates `UCGameplay.u` and `UCMenu.u` from the `.uc` sources delivered by this patch. **This step is required for the new VIDEO settings page, the main-menu rewiring, and saved-volume restoration to actually appear in-game** — without re-running `UCC make`, the engine loads the original Epic-built `.u` packages from `System/` and you'll see the stock menu instead of the patched one.

The `del UCMenu.u UCGameplay.u` is on purpose: `UCC make` decides what to rebuild by comparing source timestamps to the existing `.u`, and `git apply` doesn't always produce `.uc` files that look "newer" than the `.u` shipped in the Epic source dump. If you skip the delete you may see `Success - 0 error(s)` with no rebuild actually happening; deleting the targets forces a full re-emit.

`UCC` exits with a cosmetic `General protection fault!` after the `Success` line on this codebase. As long as you see `Success - 0 error(s)` first and the `.u` files have fresh timestamps and bigger sizes, the rebuild worked.

### 5. Drop in third-party runtimes

Copy the FFmpeg DLLs and the VC7.1 runtime DLLs (see the "What's NOT in this bundle" table) into `UC2004/System/`.

**FFmpeg DLLs are required for in-game video playback.** Without them, the FFmpeg-backed material movie path can't initialize and the engine falls back to a flat green texture wherever a movie material would have rendered. The most visible case is the **map selection screen** — every map preview will be a solid green tile. All five DLLs (`avcodec-62.dll`, `avformat-62.dll`, `avutil-60.dll`, `swresample-6.dll`, `swscale-9.dll`) need to be present and at matching majors; symbol mismatches between e.g. `avcodec-60` and `avcodec-62` will also produce the green-fallback symptom rather than a visible error.

### 6. First run

Launch `UC2004/System/avaris.exe`. The patched `avaris.ini` selects `ALAudio.ALAudioSubsystem` and 1280×720 windowed by default; tweak in-game from the new VIDEO settings page if needed.

### 7. Verify the patch is live

If something looks wrong, the table below maps each visible symptom back to the step that probably wasn't completed.

| What to check | Where it comes from | If it's wrong |
|---|---|---|
| Sound plays at all | `avaris.ini` → `AudioDevice=ALAudio.ALAudioSubsystem`, plus `ALAudio.dll` from step 3 | Check `avaris.ini` wasn't reverted; confirm `ALAudio.dll` is in `System/`. |
| Mouse feels normal (not 10× oversensitive) | `DefUser.ini` mouse block (which the engine copies into `User.ini` on first run) | Delete `User.ini` and relaunch — the engine will regenerate it from the patched `DefUser.ini`. If you launched once before applying the patch, your `User.ini` was generated from the stock Xbox-tuned values and overrides the fix. |
| New **VIDEO** settings page is visible from the main menu | `UCMenu/Classes/UC2Settings_Video.uc` (and main-menu rewiring in `UC2MainMenu.uc`, `UC2Settings_Main.uc`) — compiled into `UCMenu.u` by step 4 | Re-run `UCC make` from `UC2004/System/`. If `UCMenu.u`'s timestamp is older than your build outputs, `UCC make` was skipped and the engine is still loading the original Epic-shipped `.u`. |
| Saved master volume is restored at boot (not always default) | `UC2MainMenu.uc` — same `UCMenu.u` story | Same as above — re-run `UCC make`. |
| Map preview videos render correctly in the map selection screen (not solid green) | FFmpeg DLLs in `System/` (step 5) plus `D3DDrv.dll`, `Engine.dll` from step 3 | Confirm all five FFmpeg DLLs are in `System/` at matching majors. Solid-green map tiles is the diagnostic signature of FFmpeg failing to initialize. |
| Window opens at 1280×720 windowed (not 640×480 fullscreen) | `avaris.ini` `[WinDrv.WindowsClient]` section | Check `avaris.ini` wasn't reverted by a previous launch. The engine writes back to this section on shutdown, so if you ran a stock Avaris.exe between applying the patch and using the patched build, the values may have been overwritten. |
