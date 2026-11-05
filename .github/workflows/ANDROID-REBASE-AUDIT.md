# Sapling Android rebase audit

## Input and source comparison

The uploaded `New Folder.zip` contains two project roots:

- `Sapling-main (android)` — 225 files, 1,866,217 uncompressed bytes.
- `Sapling-main(original)` — 263 files, 2,014,832 uncompressed bytes.

The rebase uses `Sapling-main(original)` as the base. A recursive diff of the final rebased tree against that base shows only:

- New `.github/workflows/android.yml`.
- New `docs/ANDROID-ARM64-PORT.md`.
- New `docs/ANDROID-ENGINE-QUEUE-CHECKPOINT-2026-11-05.md`.
- New `docs/ANDROID-REBASE-AUDIT.md`.
- One additive UCI diagnostic line in `src/Sapling/UciEngine.cs`.

No current-source files were replaced by the older Android snapshot. The updated release workflow, `dotnet-quality.yml`, mobile app, engine code, tests, web files and human-policy tooling remain from `Sapling-main(original)`.

## Critical old-workflow defect

The old Android workflow passed `-p:SaplingBackend=Scalar` to both restore and publish. In `src/Sapling.Engine/Sapling.Engine.csproj`, that property defines `SAPLING_SCALAR`. In `EngineCapabilities.cs`, `SAPLING_SCALAR` makes `ForceScalar` a compile-time constant `true`, so `UseVector128Nnue` is false regardless of the runtime's ARM64 Vector128 support. `NnueRefresh.cs` and `Unrolled.cs` then take the scalar path.

The new workflow uses `-p:SaplingBackend=Auto` and explicitly checks evaluated `DefineConstants` for absence of `SAPLING_SCALAR`.

## How Android SIMD is expressed in this C# engine

Sapling's non-x86 backend uses `System.Runtime.Intrinsics.Vector128<T>`. On an ARM64 .NET runtime with Vector128 hardware acceleration, these operations lower to ARM Advanced SIMD (NEON). This is a .NET intrinsic path rather than handwritten C/C++ NEON. There is no promise of SDOT/Dotprod instructions; the SIMD NNUE path should be identified as Vector128/NEON, not dotprod.

The UCI handshake now reports `EngineCapabilities.NnueBackend`. On the Galaxy A16, expected: `info string NNUE backend: Vector128`. `SAPLING_BACKEND=Vector128` explicitly requires the vector backend; `SAPLING_BACKEND=Scalar` makes a same-binary scalar comparison possible.

## Build pipeline preserved and adjusted

The old Android workflow targets `linux-bionic-arm64`, uses Android NDK r27c, the pinned .NET SDK 11.0.100-rc.2.26504.105, and the matching NativeAOT Bionic runtime pack. It publishes `src/Sapling/Sapling.csproj` as a self-contained NativeAOT UCI executable. The updated workflow retains that pipeline but fixes backend selection and adds a compile-constant guard and artifact build metadata.

The new source tree also contains `src/Sapling.Mobile`, a separate MAUI project. The engine artifact workflow intentionally publishes the CLI `src/Sapling/Sapling.csproj`, not the MAUI application/APK.

## Other upstream differences preserved

The newer original source includes, among other changes:

- Shared project version 2.1.0.
- Newer UCI defaults: 16 MB Hash instead of 256 MB, with minimum 16 MB instead of 32 MB; a Hash replacement no-op guard; HumanTemperature; bounded/time-budgeted human move ranking; and the revised human selection API.
- Revised time management and hard/soft clock limits.
- Updated transposition-size inversion math and tests.
- Parallel root-ranking time limits and interruption handling.
- Human-policy minimum rating 500 (previously 800), calibration and direct-capture-risk support, related code and tests.
- MAUI app, new .NET quality workflow, and newer web/human-policy training and calibration files.

Those are updates from the newer source, not Android-port changes, and are deliberately retained. A strength A/B must use equal Hash settings because the 256-to-16 MB default change can confound results.

## Stack and worker model inspection

The single-worker search (`Threads=1`) executes directly in `ParallelSearcher.Execute` without `Task.Run` or `Parallel.For`. Multiple workers use `Parallel.For`. Recursive negamax and quiescence paths use bounded `stackalloc` arrays for legal-move candidates, score arrays, capture lists and occupancy. No Android-specific stack size is set by this port. This rebase does not alter search stack allocations or pruning; test depth and long games on the device before considering stack changes.

## Binary resources omitted from the uploaded archive

The uploaded source ZIP does not contain the two files referenced as `EmbeddedResource` in `Sapling.Engine.csproj`:

- `src/Sapling.Engine/Resources/sapling.nnue`
- `src/Sapling.Engine/Resources/human_policy_v1.weights`

The C# source and project file expect them. They may remain in the user's GitHub checkout but were not included in the provided ZIP. The modified-files ZIP preserves them by not replacing or deleting these paths. The complete source-reference ZIP necessarily lacks them and must not be used to overwrite an existing checkout that has them.

## Validation boundary

Static archive comparison, workflow/YAML parsing, shell syntax checks and project XML parsing were performed. A .NET SDK was not installed in this workspace, and the required model files were omitted from the uploaded ZIP; therefore no full .NET compile, NativeAOT link, backend runtime test or Galaxy A16 execution is claimed. GitHub Actions plus Termux testing remain required.
