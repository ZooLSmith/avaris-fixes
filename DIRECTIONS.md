# Directions taken on the PC revival

This is the "why" behind the patch bundle. The diffs in this folder show *what* changed; this document explains the design choices behind them.

## 1. Toolchain — Visual Studio 2003 SP1 (VC7.1)

The patch is intended to be built with the original toolchain the source was written for: VC7.1 / Visual Studio 2003 SP1.

The reason is that UE2.5's codebase has hundreds of small placement-new and POD-construction sites where modern MSVC optimizers make assumptions the original code doesn't hold. Building with a current MSVC produces code that compiles cleanly but crashes at runtime in places that don't reproduce on VC7.1. Patching every one of those sites would mean rewriting large parts of `Core/` and `Engine/`, with new instances liable to surface under any optimizer change.

VC7.1 is the toolchain the engine was written, optimized, and debugged against. Building on it eliminates an entire class of port bugs at the cost of a slower iteration loop (the toolchain doesn't install on Windows 10/11, so the build runs in a Windows XP guest in VirtualBox).

## 2. Audio — new module backed by OpenAL

UC2 shipped on Xbox with a proprietary audio implementation. That implementation isn't in the source dump — only stubs and the public-side glue. Without intervention the engine boots with `AudioDevice=Engine.NullAudioDevice` and runs silent.

The patch adds a new module at `UC2004/ALAudio/` that exposes the same `UAudioSubsystem` interface UC2 already expects, backed by **OpenAL**. The choice of OpenAL is driven by API surface and longevity: it's small, well-documented, cross-platform, and likely to remain available on future Windows versions, unlike DirectSound which is deprecated. There's no commercial-license dependency.

The module includes a `CueTable` layer that maps UC2's cue-driven audio model onto OpenAL sources. It registers in `avaris.ini` as `AudioDevice=ALAudio.ALAudioSubsystem`.

## 3. Video on materials — FFmpeg, decode-then-upload

UC2 plays pre-rendered cinematics as textures on materials (the menu background is the most visible example). On Xbox this used Bink. Bink for PC is a paid-license codec, and the goal here was no commercial dependencies.

The path the patch implements (in `D3DDrv/Src/D3DMoviePlayback.cpp` plus material-pipeline hooks in `UnMaterial.cpp`, `D3DMaterialState.cpp`, `D3DRenderDevice.cpp`) is **decode-to-buffer, then upload**: FFmpeg decodes into a CPU-side staging buffer, and on each frame the engine uploads from staging into the D3D texture. The reason for the buffer hop, rather than decoding straight into a locked D3D surface, is that the engine's material system reuses textures across frames in ways that don't line up with a decoder's frame cadence — surface-lifetime races become possible. The staging-buffer arrangement keeps the decoder and the renderer on independent timelines at the cost of one memcpy per frame.

Five FFmpeg runtime DLLs (`avcodec`, `avformat`, `avutil`, `swresample`, `swscale`) need to ship alongside the executable.

## 4. Input — keyboard + mouse on top of controller, not in place of it

UC2 is controller-first and the controller path in the original source is intact. The patch adds keyboard + mouse without removing or replacing the controller code — gamepads still work, KB+M is layered on.

The non-obvious part is mouse feel. UC2's input pipeline (in `UCPlayerInput.uc`) hard-codes a multiplier ratio between `aMouseX_Limit` and `aTurn_Limit` (1024 / 100 stock, i.e. 10.24×) calibrated for an Xbox stick at full deflection. Combined with a mouse bind's `Speed` factor, that multiplier sits well above the in-game sensitivity slider, and turning the slider down never reaches a usable feel.

`DefUser.ini` therefore ships with the ratio dropped to 1.0 (`aTurn_Limit=100`, `aLookUp_Limit=100`) and nonlinear acceleration disabled. That mirrors how UnrealWarfare's stock input pipeline handles the math (`aTurn += aMouseX`) and brings sensitivity into a normal range. The rationale is reproduced inline in the `.ini` so it isn't reverted by accident.

## 5. Settings UI — new VIDEO page

The stock UC2 settings UI is Xbox-tailored: a handful of preset resolutions, no display-mode toggle, no PC-specific controls. The patch adds a dedicated VIDEO settings page (`UCMenu/Classes/UC2Settings_Video.uc`) rather than crowbar PC controls into the existing settings page. A separate page keeps the original Xbox-friendly UI flow intact for anyone using a controller and gives PC controls room to breathe.

Alongside the new page, `UC2MainMenu.uc` now re-applies the saved master volume at boot. In the stock build the menu reads the slider into a UI variable but never pushes the saved value back into the audio subsystem, so every launch starts at default volume regardless of what was last set.

## 6. Three latent bugs in the source dump

Three bugs in the original sources prevent the build from running cleanly on modern Windows even with the right toolchain. They don't bite on the Xbox runtime; they trip the moment the binary launches on a desktop OS. Each is fixed at the source level by this patch.

- **Placement-new optimizer mishandle** (`Core/Inc/UnObjBas.h`). A small POD construction at a placement-new site can be reordered such that the object is partially initialized when `UObject::StaticConstructor` reads it. The fix makes the construction order explicit.
- **`FNameDebug` static-init COM deadlock** (`Core/Inc/FNameDebug.h`). The debug-name table's static initializer makes a COM call. On modern Windows the DLL load order has shifted relative to Windows XP, and COM isn't fully ready by the time static init runs in some modules — the call deadlocks. The fix defers the COM call out of static init.
- **UCC `make` reparse against stub ScriptText** (`Editor/Src/UnScrCom.cpp`). For native-noexport classes, UCC was re-parsing the public stub ScriptText against the real native implementation and fabricating bogus errors. The fix skips the reparse for native-noexport classes — the stub exists only for the script-side declaration and isn't meant to round-trip.

The fixes are commented in-place in the source.

## 7. What is deliberately out of scope

A revival has to draw a line somewhere. Several areas were audited and left alone:

- **Networking**: the multiplayer / master-server code is present and links, but the online infrastructure (master servers, CD keys, etc.) is dead. Bringing online play back is a separate project.
- **HUD / GUI scaling**: works at 1280×720 (the new default) and at 1080p. Non-16:9 aspect ratios may show alignment quirks.
- **Translucency and distortion materials**: visible regressions are gone, but the original Xbox look isn't pixel-matched.
- **Asset cooking**: untouched. The patch assumes you're using existing UC2 cooked assets (`.uax` / `.utx` / `.usx` / `.umx` / movies).
- **`UnrealEd`**: builds, launches, basic editing works. Not a focus area; expect rough edges.

## Summary

The shape of the work, in one line: **stay on the toolchain the code was written for, fill in the audio that wasn't shipped, get cinematics and KB+M working, and leave alone anything that already works.** The patches in this folder are what that shape produces.
