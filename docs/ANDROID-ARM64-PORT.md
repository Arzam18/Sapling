# Sapling Android ARM64 NativeAOT port

## What was carried forward

The previous Android tree contributes its `.github/workflows/android.yml` build path. The current source tree in `Sapling-main(original)` is the base for this rebased port, so its later changes are preserved: version 2.1.0, the revised time management and transposition sizing, HumanPlay temperature/calibration and direct-capture-risk work, the new tuning/tests, the MAUI project, the quality workflow, and the newer web/tooling files.

The Android workflow does not overwrite or downgrade any of those files. It builds the actual UCI CLI at `src/Sapling/Sapling.csproj`; it does not try to publish `Sapling.Mobile`.

## Critical discovery: the old Android workflow forced scalar NNUE

The old Android workflow passed `-p:SaplingBackend=Scalar` to both `dotnet restore` and `dotnet publish`.

In `src/Sapling.Engine/Sapling.Engine.csproj`, that property defines `SAPLING_SCALAR`. In `EngineCapabilities.cs`, `SAPLING_SCALAR` makes `ForceScalar` a compile-time `true`. The backend resolver then sets both `UseX86Nnue` and `UseVector128Nnue` to false. Therefore, despite the source having a `Vector128` path, the old Android artifact deliberately skipped that path.

The rebased workflow now passes `-p:SaplingBackend=Auto`. It also checks that `SAPLING_SCALAR` is absent from evaluated compiler constants.

## How the C# SIMD path maps to Android

Sapling uses `System.Runtime.Intrinsics.Vector128<T>` for its non-x86 NNUE backend. On ARM64, .NET's ARM64 runtime maps the supported 128-bit vector operations to ARM Advanced SIMD (NEON). It is not a handwritten C/C++ NEON backend and this code does not promise SDOT/Dotprod instructions. The x86 AVX2/AVX-512 aliases in `GlobalUsings.cs` are only used when `EngineCapabilities.UseX86Nnue` is true.

`EngineCapabilities.NnueBackend` is now reported in the UCI handshake. On the Galaxy A16 ARM64 runtime, the expected line is:

```text
info string NNUE backend: Vector128
```

The build's default Auto mode selects the best supported implementation at runtime. To require an explicit vector path on the phone, set `SAPLING_BACKEND=Vector128`. If that fails, the runtime reports that Vector128 is not hardware accelerated rather than silently pretending to use it. For a scalar A/B reference, run the same binary with `SAPLING_BACKEND=Scalar`.

## NativeAOT and Android

- Target RID: `linux-bionic-arm64`, matching Android's Bionic libc environment.
- Build host: Ubuntu x86_64.
- Android NDK: r27c; the workflow exposes its LLVM toolchain and uses the .NET NativeAOT Bionic runtime pack.
- The CLI project is published as a self-contained NativeAOT executable.
- `sapling.nnue` and `human_policy_v1.weights` are embedded resources in `Sapling.Engine`; no separate NNUE file should be required beside the executable.
- The Android workflow injects the exact NativeAOT runtime package request into the ephemeral GitHub Actions checkout only. It does not permanently edit `Sapling.csproj`.
- `SaplingBackend=Auto` is essential. Do not copy the old `Scalar` property into Lynx or another SIMD-capable C# engine without first checking what that property does.

## Search and stack observations

`ParallelSearcher.Execute` runs a single worker directly (no `Task.Run`) when `Threads=1`. For multiple workers it uses `Parallel.For`. Search code uses bounded `stackalloc` buffers in the recursive search and quiescence paths (move lists, scores, captures, and occupancy arrays). The current project does not set an Android-specific thread stack size. This port does not change those search allocations or search heuristics. The device test should include a deeper search and a longer game to expose any actual stack/runtime issue before we change them.

## Important comparison controls

The newer original source changes the UCI default Hash from 256 MB to 16 MB, and lowers the minimum accepted Hash from 32 MB to 16 MB. That is an upstream change preserved by this rebase; it is not an Android compile option. For meaningful strength A/B tests, set the same `Hash`, `Threads`, time control, book, and engine options for both builds. Do not compare the old Android build's 256 MB default with the rebased build's 16 MB default and attribute any difference solely to SIMD.

The previous Android workflow forced `Scalar`, so a fair baseline comparison should use this rebased executable twice: once with `SAPLING_BACKEND=Scalar`, once with `SAPLING_BACKEND=Vector128`. This isolates the NNUE SIMD backend on the same newer source revision.

## Phone-side UCI check

Expected handshake includes `info string NNUE backend: Vector128`. Then send:

```text
isready
position startpos
go depth 10
```

Require a legal `bestmove`. Also run the same test with `SAPLING_BACKEND=Scalar`; compare `bench` output, NPS, and tournament strength with identical options. Record the backend line for each run.

## Limitations of this package

This workspace has no installed .NET SDK and cannot fetch the pinned SDK/NativeAOT pack, so a full NativeAOT publish and actual Galaxy A16 execution were not performed here. The package is rebased and statically inspected; GitHub Actions and Termux runtime verification remain required.
