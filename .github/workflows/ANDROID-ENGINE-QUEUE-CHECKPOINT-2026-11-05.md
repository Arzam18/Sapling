# Android ARM64 chess-engine queue checkpoint

Date: 2026-11-05

## Completed / verified by user

1. Askaig — Android ARM64 NEON runtime confirmed.
2. Chessnix — Android ARM64 NEON runtime confirmed.
3. NARC Genesys — Android runtime confirmed; evaluator is NNUE with correction-history adjustment, not conventional NNUE+HCE.
4. Pea — Android runtime confirmed.
5. RukChess — Android runtime confirmed; embedded NNUE and Android-specific build fixes accepted.
6. Solo — Android runtime confirmed.
7. Peacekeeper — Android build/runtime accepted; preserve original NNUE hash and stack fix.
8. Quirky — Android build/runtime accepted; stack-bounds fix.
9. Kociolek 2.3 — Android build/runtime confirmed by user after embedded-NNUE and workflow-permission corrections.

## Active and next

10. **Tarnished 6.0 — active / awaiting final verification.** The first Android compile fix added the missing `store_epi8` SIMD helper and cleaned the warnings. Await the next GitHub Actions build and device test before marking complete.

11. **Sapling 2.1.0 — newly inserted, next after Tarnished and before Lynx.** C#/.NET NativeAOT ARM64 port rebased onto the newer `Sapling-main(original)` archive. The old Android workflow explicitly forced `SaplingBackend=Scalar`, which defines `SAPLING_SCALAR` and disables `UseVector128Nnue`; the rebased workflow uses `SaplingBackend=Auto` and reports the runtime NNUE backend in the UCI handshake. Expected on ARM64: `Vector128` (.NET AdvSIMD/NEON).

12. **Lynx — queued after Sapling.** Also C#. Reuse the Sapling lessons: inspect runtime intrinsic capability selection, NativeAOT/RID/runtime-pack workflow, backend compile constants, and UCI backend reporting before assuming that source-level SIMD is active.

13. **Booot — queued after Lynx.**

## Sapling merge/rebase notes

- Base the reimplementation on `Sapling-main(original)`, not the older `Sapling-main (android)` folder.
- The original tree contains the two later updates and additions: version 2.1.0 changes, updated time management and TT sizing, newer HumanPlay calibration/direct-capture-risk code and tests, the MAUI project, the .NET quality workflow, plus newer web and tooling files.
- The only Android-specific file in the old Android tree was `.github/workflows/android.yml`; the old project code itself had no Android-specific edits in the archive.
- Preserve the newer source tree and add the Android workflow plus UCI backend diagnostic. Do not copy the old Android tree over the newer original tree.
- The uploaded ZIP omitted `src/Sapling.Engine/Resources/sapling.nnue` and `src/Sapling.Engine/Resources/human_policy_v1.weights`, although the project file and source expect those resources. Apply the modified-files ZIP to the actual repository branch so existing model assets remain untouched. The complete source reference archive cannot supply missing model binaries.
- The Android workflow produces the UCI command-line engine for `linux-bionic-arm64`; it does not build the separate MAUI application or an APK.

## Standard Sapling verification

1. Run the `Sapling Android ARM64 NativeAOT (Vector128)` workflow.
2. Confirm the compile-time constants do not contain `SAPLING_SCALAR`.
3. Download the `Sapling-Android-ARM64-Vector128-Auto` artifact and verify on-device UCI output reports `NNUE backend: Vector128`.
4. Run the same binary once with `SAPLING_BACKEND=Vector128`, then with `SAPLING_BACKEND=Scalar` to create an A/B reference on identical source, NNUE, time control, hash and thread count.
5. Compare bench/NPS and tournament strength. Do not compare unmatched Hash defaults: the newer Sapling source changes default Hash from 256 MB to 16 MB.
6. Only mark Sapling complete after a legal bestmove, deep-search stability and repeatable strength comparison on-device.
