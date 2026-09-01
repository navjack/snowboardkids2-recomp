# macOS port notes — Snowboard Kids 2: Recompiled

Builds and runs on Apple Silicon; reaches the attract screen ("PUSH START BUTTON") and renders
through Metal. Installed as `SnowboardKids2Recompiled.app`.

This project already ships real macOS support (`src/main/support_apple.mm`,
`.github/macos/apple_bundle.cmake`, Homebrew LLVM detection for the MIPS patches), so only two
things needed fixing.

## Preparation

Stage the ROM and generate sources with this project's **own** vendored recompiler:

```bash
cp "Snowboard Kids 2 (U) [!].z64" snowboardkids2.z64
cmake -S lib/N64ModernRuntime/N64Recomp -B lib/N64ModernRuntime/N64Recomp/build -DCMAKE_BUILD_TYPE=Release
cmake --build lib/N64ModernRuntime/N64Recomp/build -j8
cp lib/N64ModernRuntime/N64Recomp/build/N64Recomp lib/N64ModernRuntime/N64Recomp/build/RSPRecomp .
rm -rf RecompiledFuncs RecompiledPatches
./N64Recomp us.toml && ./RSPRecomp aspMain.us.toml
```

The `N64Recomp` binary must sit in the repo root: the `PatchesLib` target shells out to a literal
`./N64Recomp patches.toml`.

**Do not reuse another project's N64Recomp.** Doing so produced sources that failed to compile
with `no member named 'nullopt' in namespace 'std'` and
`invalid application of 'sizeof' to an incomplete type 'SectionTableEntry[]'` inside the generated
`recomp_overlays.inl` — the generator targets a different runtime API than this project pins, and
nothing reports a version mismatch.

## Changes made

**`.github/macos/apple_bundle.cmake`** — vendor `libSDL3.dylib` into `Contents/Frameworks`.
CMake's `fixup_bundle` only follows link-time dependencies, but sdl2-compat `dlopen`s
`libSDL3.dylib` by name at load time, so nothing copied it. The bundle built, linked, and signed
correctly yet could never reach `main()`: it aborted in `dllinit` with a modal
"Failed loading SDL3 library". The dylib is resolved at configure time via `brew --prefix sdl3`
with a `FATAL_ERROR` if missing, and installed with `-id @loader_path/libSDL3.dylib` since
sdl2-compat searches `@loader_path` first.

**`lib/RecompFrontend/recompui/src/core/ui_context.cpp`** — `context_error()` now also writes the
reason to stderr before showing its message box. The fatal path previously reported only through a
message box that vanishes with the process, making it undiagnosable from a terminal.

## Build

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Release -DCMAKE_OSX_DEPLOYMENT_TARGET=11.0 .
cmake --build build -j8
# produces build/SnowboardKids2Recompiled.app
```

## Known issues

- A `context_error` fired once on the very first run and has not recurred. Most likely first-run
  config initialisation, but **this was not reproduced deliberately** — treat it as unconfirmed.
  The stderr diagnostic above will capture the reason if it returns.
- Crashes on quit via the shared plume Metal teardown bug (see the other ports' notes).
