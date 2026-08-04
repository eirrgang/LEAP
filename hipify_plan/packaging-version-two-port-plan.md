# Cross-session plan: align `eirrgang-packaging` and port packaging to `version_two`

**Created:** 2026-08-03  
**Current planning branch when written:** `version_two`  
**Primary goal:** Make the infrastructure and packaging situation similar enough between `eirrgang-packaging` and `version_two` that source-code changes between the `notex` lineage and `version_two` can be reviewed with less path and packaging noise.

This document is intended to be updated from one commit to the next, across sessions. Treat it as the authoritative working checklist for this effort.

---

## 0. Branch and ancestry facts

Known refs at planning time:

```text
notex                         451d29a cleanup code, remove comments
eirrgang-packaging            bb7145f Clarify LEAP_GPU autodetection
version_two                   0c42bb2 remove files
main                          0c8846f updated version
```

Merge bases at planning time:

```text
merge-base(notex, eirrgang-packaging)       = 451d29a811fb46a6f3e6319968391f95b89ad76f
merge-base(version_two, eirrgang-packaging) = 0c8846f42b2e59340d5559fc1271d590a292f9a0
merge-base(notex, version_two)              = 0c8846f42b2e59340d5559fc1271d590a292f9a0
```

Important conclusion:

- Do **not** directly merge `eirrgang-packaging` into `version_two` for this work.
- The useful packaging delta is conceptually `notex..eirrgang-packaging`, but `version_two` has a different source layout and extra source content.
- First align `eirrgang-packaging`'s `src/` layout toward `version_two`; then use that aligned packaging branch as the reference for the `version_two` port.

---

## 1. Progress log

Update this table after each commit or explicit pause point.

| Step | Branch | Commit | Status | Validation | Notes |
|---|---|---:|---|---|---|
| Plan document created | `version_two` | pending | done | not run | Initial cross-session tracking document. |
| Phase 0.1 source moves | `eirrgang-packaging-src-layout-alignment` | f51e125 | done | static include scan | Source files moved under `src/fbp`, `src/projectors`, `src/denoise`, `src/geometry`, and `src/ray_tracing`; cross-subdirectory includes updated to match `version_two` style. |
| Phase 0.2 CMake path update | `eirrgang-packaging-src-layout-alignment` | c3bf0db | done | direct CMake CPU/HIP and `python -m build` CPU/AMD wheels passed | `src/CMakeLists.txt` paths updated; CPU-only and AMD HIP builds passed with explicit ROCm 6.4.3 compilers, including isolated `python -m build` using `.wheelhouse/`. |
| Phase 1 branch prep | `version-two-packaging-port` | 6c8a45f | done | not run | User approved proceeding after Phase 0; branch created from `version_two`; plan document restored from alignment branch. |
| Phase 2a user subtree import | `version-two-packaging-port` | ecc8a6e | done | subtree files verified | User recreated `third_party/hipify_torch` subtree import and merge commit. |
| Phase 2b support files | `version-two-packaging-port` | ec112fc, 74243a5, 544d06f | done | not run | HIPify helper/docs, GPU wheel helper, and cross-runtime smoke test helper committed. |
| Phase 3 pyproject | `version-two-packaging-port` | e0e8cd1 | done | isolated `python -m build` CPU wheel passed | Added scikit-build-core `pyproject.toml`; legacy setup files kept tracked for now and excluded from sdist pending any later deletion decision. |
| Phase 4 Python packages | `version-two-packaging-port` | f32101f | done | isolated CPU wheel and basic imports passed | Converted flat Python modules to package directories and updated `pyproject.toml` wheel package entries; full library-load import remains for Phase 5 loader/CMake install work. |
| Phase 5 loader | `version-two-packaging-port` | 1cbad28 | done | isolated CPU wheel load smoke passed | Resource-based loader added to `xrayphysics`; Phase 7 later verifies the package-resource native library location. |
| Phase 6 top CMake | `version-two-packaging-port` | b001ee0 | done | validated together with Phase 7 | `LEAP_GPU` backend selection added with HIP autodetection when `enable_language(HIP)` can succeed. |
| Phase 7 src CMake | `version-two-packaging-port` | 3c9fa07 | done | CPU, explicit AMD, implicit AMD configure/build and AMD wheels passed | Source lists/target/link/install rules updated for packaged CPU/CUDA/HIP backends; HIPified mapping handles subdirectories. |
| Phase 8 CPU FBP fix | `version-two-packaging-port` | 044ef2e | done | direct CPU CMake build passed | Equivalent CPU-only FBP guards already present in `src/fbp/filtered_backprojection.cpp`; recorded comparison against packaging commit `bbd30e1`. |
| Phase 9 docs | `version-two-packaging-port` | pending | ready to commit | not run | README wheel/editable/backend docs ported and linked to HIPify and cross-runtime docs. |
| Phase 10 validation | `version-two-packaging-port` | pending | not started | not run | CPU/wheel/CUDA/HIP checks. |

---

## 2. Agent operating rules for this plan

### 2.1 When to write commits

An agent may write a git commit only when all of the following are true:

1. The working tree contains one coherent logical change.
2. This planning document has been updated with the branch, intended commit, validation performed, and any known issues.
3. Relevant lightweight validation has been run, or the document explicitly records why validation was skipped.
4. The phase either has prior user approval or is explicitly listed below as commit-eligible.

Every commit message must end with:

```text
Co-Authored-By: Claude <noreply@anthropic.com>
```

When reproducing the packaging work on `version_two`, preserve the `eirrgang-packaging` commit sequence and commit messages as closely as practical. It is acceptable and expected to adapt the patch contents for `version_two`'s source layout and XRayPhysics integration, but the history should remain recognizable when compared against `notex..eirrgang-packaging`.

Exception: the `third_party/hipify_torch` subtree import and associated merge commit may require `git subtree` commands that the agent should not assume it can run or faithfully recreate. For that step, the agent must pause and prompt the user to run the subtree import commands, then resume from the resulting user-created commit.

### 2.2 When to update this planning document

Update this file:

```text
hipify_plan/packaging-version-two-port-plan.md
```

at these points:

- after creating each working branch
- before starting each phase
- after each commit
- after each validation attempt
- when a file is classified as intentionally non-comparable
- when a decision is made about legacy setup files
- when pausing for user approval
- when a command fails and the next action is not purely mechanical

### 2.3 When to pause and ask the user

Pause at these gates:

1. **Before starting Phase 0 implementation**, unless the user has explicitly approved implementation after reading this document.
2. **After Phase 0 on `eirrgang-packaging`**, before modifying `version_two`.
3. **Before deleting or deprecating legacy setup files**, especially:
   - `setup.py`
   - `setup.cfg`
   - `setup_AMD.py`
   - `setup_cpu.py`
   - `setup_torch.py`
   - `setup_old.py`
   - `setup_ctype.py`
4. **Before converting `xrayphysics.py` and `leapctserver.py` to packages**, if public import/module layout compatibility is uncertain.
5. **If CPU/CUDA/HIP validation fails due to a design choice**, rather than an obvious typo/path fix.
6. **Before the `third_party/hipify_torch` subtree import on `version_two`**, because the user may need to run `git subtree` and create the associated merge commit manually.
7. **Before final branch cleanup, rebase, force-push, branch deletion, or any destructive operation.**

---

## 3. Branch strategy

### 3.1 Packaging layout branch

Create from `eirrgang-packaging`:

```bash
git checkout eirrgang-packaging
git checkout -b eirrgang-packaging-src-layout-alignment
```

Purpose:

- Restructure only enough of `src/` to make source paths comparable to `version_two`.
- Preserve packaging branch behavior.
- Do not import `version_two` feature code unless explicitly approved.

### 3.2 Version-two packaging port branch

Create only after Phase 0 approval:

```bash
git checkout version_two
git checkout -b version-two-packaging-port
```

Purpose:

- Port packaging, HIPify, `LEAP_GPU`, wheel, and loader updates into `version_two`.
- Preserve `version_two` source content and XRayPhysics integration.
- Use `eirrgang-packaging-src-layout-alignment` as the packaging reference.

---

# Phase 0 — Restructure `eirrgang-packaging/src` to match `version_two/src`

## Goal

Make the `src/` directory layout of `eirrgang-packaging` as similar as possible to `version_two`, while preserving the packaging branch's actual source content.

This phase must happen before porting packaging changes to `version_two`.

## Branch

Work on:

```text
eirrgang-packaging-src-layout-alignment
```

based on:

```text
eirrgang-packaging
```

## Phase 0 approval status

- [x] User approved starting Phase 0 implementation (`2026-08-04` request: "Work through the plan...").
- [x] Branch created: `eirrgang-packaging-src-layout-alignment` from `eirrgang-packaging`.
- [x] Source moves committed: f51e125.
- [x] Cross-subdirectory `#include` updates committed: f51e125.
- [x] CMake/build updates committed: c3bf0db.
- [x] Include-style comparison against `version_two` completed/documented: moved cross-subdirectory prefixes now match the `version_two` style in common files; remaining include differences are `version_two` feature additions or packaging-branch-only legacy differences.
- [x] Direct CMake CPU-only and AMD HIP validation passed with explicit ROCm 6.4.3 compilers.
- [x] Python package front-end validation passed for CPU-only and AMD HIP wheels using isolated `python -m build` with `.wheelhouse/`.
- [x] User approved proceeding to `version_two` on 2026-08-04 after Phase 0 validation.

## Phase 0.1 — Move files in `eirrgang-packaging`

Move flat files into the same subdirectories used by `version_two`.

### Move to `src/fbp/`

```text
src/filtered_backprojection.cpp      -> src/fbp/filtered_backprojection.cpp
src/filtered_backprojection.h        -> src/fbp/filtered_backprojection.h
src/ramp_filter.cu                   -> src/fbp/ramp_filter.cu
src/ramp_filter.cuh                  -> src/fbp/ramp_filter.cuh
src/ramp_filter_cpu.cpp              -> src/fbp/ramp_filter_cpu.cpp
src/ramp_filter_cpu.h                -> src/fbp/ramp_filter_cpu.h
src/ray_weighting.cu                 -> src/fbp/ray_weighting.cu
src/ray_weighting.cuh                -> src/fbp/ray_weighting.cuh
src/ray_weighting_cpu.cpp            -> src/fbp/ray_weighting_cpu.cpp
src/ray_weighting_cpu.h              -> src/fbp/ray_weighting_cpu.h
```

### Move to `src/projectors/`

```text
src/backprojectors_VD.cu             -> src/projectors/backprojectors_VD.cu
src/backprojectors_VD.cuh            -> src/projectors/backprojectors_VD.cuh
src/projectors.cpp                   -> src/projectors/projectors.cpp
src/projectors.h                     -> src/projectors/projectors.h
src/projectors_Joseph.cu             -> src/projectors/projectors_Joseph.cu
src/projectors_Joseph.cuh            -> src/projectors/projectors_Joseph.cuh
src/projectors_Joseph_cpu.cpp        -> src/projectors/projectors_Joseph_cpu.cpp
src/projectors_Joseph_cpu.h          -> src/projectors/projectors_Joseph_cpu.h
src/projectors_SF.cu                 -> src/projectors/projectors_SF.cu
src/projectors_SF.cuh                -> src/projectors/projectors_SF.cuh
src/projectors_SF_cpu.cpp            -> src/projectors/projectors_SF_cpu.cpp
src/projectors_SF_cpu.h              -> src/projectors/projectors_SF_cpu.h
src/projectors_Siddon.cu             -> src/projectors/projectors_Siddon.cu
src/projectors_Siddon.cuh            -> src/projectors/projectors_Siddon.cuh
src/projectors_Siddon_cpu.cpp        -> src/projectors/projectors_Siddon_cpu.cpp
src/projectors_Siddon_cpu.h          -> src/projectors/projectors_Siddon_cpu.h
src/projectors_attenuated.cu         -> src/projectors/projectors_attenuated.cu
src/projectors_attenuated.cuh        -> src/projectors/projectors_attenuated.cuh
src/projectors_extendedSF.cu         -> src/projectors/projectors_extendedSF.cu
src/projectors_extendedSF.cuh        -> src/projectors/projectors_extendedSF.cuh
src/projectors_symmetric.cu          -> src/projectors/projectors_symmetric.cu
src/projectors_symmetric.cuh         -> src/projectors/projectors_symmetric.cuh
src/projectors_symmetric_cpu.cpp     -> src/projectors/projectors_symmetric_cpu.cpp
src/projectors_symmetric_cpu.h       -> src/projectors/projectors_symmetric_cpu.h
src/sensitivity.cu                   -> src/projectors/sensitivity.cu
src/sensitivity.cuh                  -> src/projectors/sensitivity.cuh
src/sensitivity_cpu.cpp              -> src/projectors/sensitivity_cpu.cpp
src/sensitivity_cpu.h                -> src/projectors/sensitivity_cpu.h
```

### Move to `src/denoise/`

```text
src/bilateral_filter.cu              -> src/denoise/bilateral_filter.cu
src/bilateral_filter.cuh             -> src/denoise/bilateral_filter.cuh
src/guided_filter.cu                 -> src/denoise/guided_filter.cu
src/guided_filter.cuh                -> src/denoise/guided_filter.cuh
src/matching_pursuit.cu              -> src/denoise/matching_pursuit.cu
src/matching_pursuit.cuh             -> src/denoise/matching_pursuit.cuh
src/noise_filters.cu                 -> src/denoise/noise_filters.cu
src/noise_filters.cuh                -> src/denoise/noise_filters.cuh
src/total_variation.cu               -> src/denoise/total_variation.cu
src/total_variation.cuh              -> src/denoise/total_variation.cuh
```

### Move to `src/geometry/`

```text
src/find_center_cpu.cpp              -> src/geometry/find_center_cpu.cpp
src/find_center_cpu.h                -> src/geometry/find_center_cpu.h
src/geometric_calibration.cu         -> src/geometry/geometric_calibration.cu
src/geometric_calibration.cuh        -> src/geometry/geometric_calibration.cuh
src/rebin.cpp                        -> src/geometry/rebin.cpp
src/rebin.h                          -> src/geometry/rebin.h
```

### Move to `src/ray_tracing/`

```text
src/analytic_ray_tracing.cpp         -> src/ray_tracing/analytic_ray_tracing.cpp
src/analytic_ray_tracing.h           -> src/ray_tracing/analytic_ray_tracing.h
src/analytic_ray_tracing_gpu.cu      -> src/ray_tracing/analytic_ray_tracing_gpu.cu
src/analytic_ray_tracing_gpu.cuh     -> src/ray_tracing/analytic_ray_tracing_gpu.cuh
src/phantom.cpp                      -> src/ray_tracing/phantom.cpp
src/phantom.h                        -> src/ray_tracing/phantom.h
```

## Include-path and `#include` update strategy

`version_two` uses a mixed include style:

- same-directory headers are usually included without a prefix, for example `#include "filtered_backprojection.h"` from files already under `src/fbp/`;
- root-level headers are usually included without a prefix, relying on `src` in the include search path;
- cross-subdirectory headers are often included with their source-layout prefix, for example `#include "fbp/ramp_filter.cuh"`, `#include "projectors/projectors.h"`, `#include "geometry/rebin.h"`, `#include "ray_tracing/analytic_ray_tracing.h"`, and `#include "physics/xrayphysics_c_interface.h"`.

Therefore Phase 0 should not rely only on adding subdirectories to the include search path. It should also update cross-directory `#include` lines in `eirrgang-packaging` so they match the `version_two` style wherever a directly comparable `version_two` file exists.

Keep the root include directory:

```cmake
include_directories(
  ./
)
```

Add subdirectory include paths only if needed for compatibility with remaining bare includes or third-party/generated HIPified code:

```cmake
include_directories(
  ./
  fbp
  projectors
  denoise
  geometry
  ray_tracing
)
```

When updating includes, prefer these rules:

1. If the included header is in the same moved subdirectory as the including file, leave it bare.
2. If the included header remains in `src/`, leave it bare.
3. If the included header is in another moved subdirectory, add the same prefix used by `version_two`.
4. Do not rewrite external includes such as CUDA/HIP/system/library headers.

Concrete examples to apply to the aligned packaging branch:

```text
#include "projectors.h"              -> #include "projectors/projectors.h"
#include "projectors_symmetric.cuh"  -> #include "projectors/projectors_symmetric.cuh"
#include "projectors_attenuated.cuh" -> #include "projectors/projectors_attenuated.cuh"
#include "ramp_filter_cpu.h"         -> #include "fbp/ramp_filter_cpu.h"       when included from outside `src/fbp/`
#include "ramp_filter.cuh"           -> #include "fbp/ramp_filter.cuh"         when included from outside `src/fbp/`
#include "ray_weighting_cpu.h"       -> #include "fbp/ray_weighting_cpu.h"     when included from outside `src/fbp/`
#include "rebin.h"                   -> #include "geometry/rebin.h"           when included from outside `src/geometry/`
#include "find_center_cpu.h"         -> #include "geometry/find_center_cpu.h" when included from outside `src/geometry/`
#include "analytic_ray_tracing.h"    -> #include "ray_tracing/analytic_ray_tracing.h" when included from outside `src/ray_tracing/`
#include "analytic_ray_tracing_gpu.cuh" -> #include "ray_tracing/analytic_ray_tracing_gpu.cuh" when included from outside `src/ray_tracing/`
#include "noise_filters.cuh"         -> #include "denoise/noise_filters.cuh"  when included from outside `src/denoise/`
```

After Phase 0, run a focused comparison of quoted includes in common files to confirm that path prefixes match `version_two` except where the files intentionally differ.

## Phase 0.2 — Update CMake in `eirrgang-packaging`

Update `src/CMakeLists.txt` to reference the new paths.

Examples:

```cmake
fbp/filtered_backprojection.cpp
fbp/ramp_filter.cu
projectors/projectors_SF.cu
denoise/noise_filters.cu
geometry/geometric_calibration.cu
ray_tracing/analytic_ray_tracing_gpu.cu
```

Keep the packaging branch's existing behavior for:

- `LEAP_GPU`
- HIP source generation
- scikit-build install behavior
- wheel package layout

## HIPify check for Phase 0

Because HIPify operates over the `src/` tree, verify after the move that:

- HIPified paths preserve subdirectory structure.
- `get_hipified_list()` still maps all moved files.
- AMD builds still define `__USE_NOTEX`.

## Validation for Phase 0

Run at least:

```bash
cmake -S . -B /tmp/leap-packaging-cpu -DLEAP_GPU=None
cmake --build /tmp/leap-packaging-cpu -j
```

If CUDA is available:

```bash
cmake -S . -B /tmp/leap-packaging-cuda -DLEAP_GPU=NVIDIA
cmake --build /tmp/leap-packaging-cuda -j
```

If HIP is available:

```bash
cmake -S . -B /tmp/leap-packaging-hip -DLEAP_GPU=AMD
cmake --build /tmp/leap-packaging-hip -j
```

## Commit guidance for Phase 0

### Commit 0.1 — source moves only

Commit message:

```text
Align packaging branch source layout with version_two

Co-Authored-By: Claude <noreply@anthropic.com>
```

Contents:

- file moves under `src/fbp`, `src/projectors`, `src/denoise`, `src/geometry`, `src/ray_tracing`
- cross-subdirectory `#include` updates needed to match `version_two` path-prefix style
- minimal include-search-path changes only if still necessary after source includes are updated
- this planning document update

### Commit 0.2 — CMake/build updates

Commit message:

```text
Update packaging CMake lists for aligned source layout

Co-Authored-By: Claude <noreply@anthropic.com>
```

Contents:

- `src/CMakeLists.txt`
- any HIPify path updates
- this planning document update
- validation notes

## Approval gate after Phase 0

Pause and ask the user to review before modifying `version_two`.

The approval request should include:

- the Phase 0 commit SHAs
- `git diff --stat eirrgang-packaging..eirrgang-packaging-src-layout-alignment`
- validation results
- list of files still not comparable

---

# Phase 1 — Prepare `version_two` packaging port branch

## Branch

After Phase 0 approval:

```bash
git checkout version_two
git checkout -b version-two-packaging-port
```

## Goal

Port packaging infrastructure into `version_two`, using `eirrgang-packaging-src-layout-alignment` as the packaging reference.

## Checklist

- [x] User approved proceeding after Phase 0 on 2026-08-04.
- [x] Branch created from `version_two`: `version-two-packaging-port`.
- [x] This document updated with branch name and Phase 0 commit SHAs: f51e125, c3bf0db, 4fcfdc8, 7c5d55d, 8ce5d18.
- [x] No code changes committed yet on `version-two-packaging-port` as of branch prep commit 6c8a45f; only this plan document was committed for branch-prep tracking.

---

# Phase 2 — Add packaging-only infrastructure to `version_two`

This phase should reproduce the corresponding `eirrgang-packaging` history as closely as practical, while adapting paths/content for `version_two` when needed.

## Phase 2a — Pause for user-created hipify subtree import

The original packaging branch includes a subtree import sequence for `third_party/hipify_torch`, including the subtree content commit and the associated merge commit:

```text
b971ccd Squashed 'third_party/hipify_torch/' content from commit 1ea3231
c45915c Merge commit 'b971ccd10b0e06f48ed30e6792daffb6b7607a6b' as 'third_party/hipify_torch'
```

The agent may not be able to faithfully recreate this using `git subtree` in the local environment. Therefore, before adding or modifying `third_party/hipify_torch/**` on `version-two-packaging-port`, the agent must pause and ask the user to run the subtree import and create the equivalent merge commit.

Suggested user-facing prompt:

```text
Please recreate the `third_party/hipify_torch` subtree import on the current branch, matching the subtree import used by `eirrgang-packaging` as closely as possible. After the subtree merge commit is present, tell me the resulting commit SHA and I will continue with the remaining packaging commits.
```

If helpful, the agent may show the user the reference commits and inspect their metadata, but should not assume it can run the subtree import itself.

### Phase 2a checklist

- [x] Agent paused before touching `third_party/hipify_torch/**` on `version_two`.
- [x] User recreated the subtree import and merge commit.
- [x] Resulting merge commit SHA recorded: ecc8a6e (`Merge commit 'a0a3df914165f6f6c9dbd0d3e295bb99b3c73042' as 'third_party/hipify_torch'`).
- [x] Agent verified `third_party/hipify_torch/**` exists after the user-created commit.
- [x] Agent resumed remaining Phase 2 work.

## Phase 2b — Add non-subtree packaging support files

Bring in packaging-only non-subtree files from the aligned packaging branch:

```text
cmake/LeapHipify.cmake
docs/hipify-strategy.md
scripts/build-gpu-wheel.sh
scripts/cross-runtime-smoke-test.md
scripts/cross-runtime-smoke-test.sh
```

Also port `.gitignore` additions for generated HIP/build artifacts, but do not replace the entire file blindly.

### Commit-history guidance

Where practical, reproduce the original commit boundaries and messages from `eirrgang-packaging`, especially:

```text
a91a43d Wire vendored hipify into AMD build
eebc6f9 Add GPU wheel version wrapper
28285b1 Add cross-runtime wheel smoke test helper
```

Adapt each commit to `version_two` as needed, but keep the intent and message recognizable.

## Checklist

- [x] User-created subtree import is present: merge commit ecc8a6e.
- [x] `cmake/LeapHipify.cmake` added.
- [x] `docs/hipify-strategy.md` added.
- [x] Scripts added: `scripts/build-gpu-wheel.sh`, `scripts/cross-runtime-smoke-test.md`, and `scripts/cross-runtime-smoke-test.sh`.
- [x] `.gitignore` reviewed manually; no generated HIP/build-artifact ignore additions were needed beyond existing build/dist/wheel patterns.
- [x] No unrelated sample/output files added.
- [x] This document updated for the HIPify support-file and helper-script commits.
- [x] Commits written with messages matching the packaging branch where practical: ec112fc, 74243a5, 544d06f.

## Commit messages to prefer where applicable

```text
Wire vendored hipify into AMD build

Co-Authored-By: Claude <noreply@anthropic.com>
```

```text
Add GPU wheel version wrapper

Co-Authored-By: Claude <noreply@anthropic.com>
```

```text
Add cross-runtime wheel smoke test helper

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

# Phase 3 — Introduce `pyproject.toml` in `version_two`

Port and adapt `pyproject.toml`.

## Required adaptations

1. Use `scikit-build-core`.
2. Use a `version_two`-appropriate fallback version, likely:

   ```toml
   fallback_version = "2.0+untagged"
   ```

3. Include all relevant Python packages/modules:
   - `leapctype`
   - `leaptorch`
   - `leap_filter_sequence`
   - `leap_preprocessing_algorithms`
   - `xrayphysics`
   - possibly `leapctserver`
4. Include in the sdist:
   - `src/**`
   - `cmake/**`
   - `scripts/**`
   - `docs/hipify-strategy.md`
   - `third_party/hipify_torch/**`
   - `CMakeLists.txt`
   - `pyproject.toml`
   - `README.md`
   - `LICENSE`
   - `NOTICE`, if present
5. Exclude examples, outputs, tests, utils, legacy setup helpers, and backup files.

## Approval checkpoint before deleting legacy setup files

Pause before deleting or replacing canonical legacy files if there is any ambiguity about whether these should remain:

```text
setup.py
setup.cfg
setup_AMD.py
setup_cpu.py
setup_torch.py
setup_old.py
setup_ctype.py
```

Recommended default:

- delete `setup.py` and `setup.cfg` as canonical packaging entry points
- keep other setup helpers only if useful locally, excluded from sdist

## Checklist

- [x] `pyproject.toml` added/adapted for `version_two`, including flat Python modules and XRayPhysics/leapctserver modules.
- [x] Legacy setup-file decision recorded: keep existing `setup.py`, `setup.cfg`, and setup helper scripts tracked for now; exclude them from the sdist so `pyproject.toml` is the packaging front end.
- [x] User approval not needed for deletion because no legacy setup files are being deleted in this phase.
- [x] This document updated.
- [x] Commit written: e0e8cd1.
- [x] Validation: isolated CPU-only `python -m build` passed with `.wheelhouse/`, using `-Ccmake.define.CPUONLY=ON`; wheel contained flat Python modules plus `lib_cpu/libleapct_cpu.so`.

## Commit message

```text
Introduce scikit-build-core packaging

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

# Phase 4 — Decide Python package layout in `version_two`

## Recommended approach

Convert flat Python modules to packages to match the packaging branch:

```text
src/leapctype.py                         -> src/leapctype/__init__.py
src/leaptorch.py                         -> src/leaptorch/__init__.py
src/leap_filter_sequence.py              -> src/leap_filter_sequence/__init__.py
src/leap_preprocessing_algorithms.py     -> src/leap_preprocessing_algorithms/__init__.py
src/xrayphysics.py                       -> src/xrayphysics/__init__.py
src/leapctserver.py                      -> src/leapctserver/__init__.py
```

This maximizes layout similarity and makes wheel package-data installation cleaner.

## Checklist

- [x] Move Python modules to package directories, including `xrayphysics` and `leapctserver`.
- [x] Update imports if required: existing absolute imports remain valid with package directories; `pyproject.toml` wheel package entries updated from module files to package directories.
- [x] Run import smoke checks: isolated CPU wheel built and installed; basic imports for `leapctype`, `xrayphysics`, and `leap_filter_sequence` passed. Importing `leap_preprocessing_algorithms` still triggers a default `tomographicModels()` shared-library load that fails until Phase 5/7 resource loading and install destination are adapted.
- [x] This document updated.
- [x] Commit written: f32101f.

## Commit message

```text
Convert Python modules to package directories

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

# Phase 5 — Adapt shared library loading in `version_two`

## Key difference

In original `eirrgang-packaging`, library loading was changed in:

```text
src/leapctype/__init__.py
```

In `version_two`, `leapctype` inherits from `xrayPhysics`, and loading currently happens in:

```text
src/xrayphysics.py
```

or after package conversion:

```text
src/xrayphysics/__init__.py
```

Therefore, adapt the resource-based loader into `xrayphysics`, not blindly into `leapctype`.

## Requirements

The loader should:

1. Preserve explicit `lib_dir`.
2. Prefer installed package resources.
3. Load the library from the installed `leapctype` package directory.
4. Preserve direct-build fallbacks.
5. Keep `only_cpu=True` compatibility if possible.
6. Raise a useful error when the library cannot be found.

## Checklist

- [x] Add resource-based loader to `xrayphysics`.
- [x] Preserve explicit `lib_dir` behavior.
- [x] Preserve direct-build fallback behavior.
- [x] Confirm `leapctype.tomographicModels()` still calls through `super().__init__`.
- [x] Run import/library-load smoke test: CPU-only wheel build/install passed, `leapctype.tomographicModels(only_cpu=True)` loaded the installed library via the current `lib_cpu/` install-layout fallback, and explicit `lib_dir` load passed.
- [x] This document updated.
- [x] Commit written: 1cbad28.

## Commit message

```text
Load LEAP shared library from package resources

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

# Phase 6 — Port top-level CMake GPU backend selection

Replace old `CPUONLY`-centric top-level CMake with:

```text
LEAP_GPU=NVIDIA|AMD|None
```

## Required behavior

- auto-detect only when unambiguous
- fail if both CUDA and HIP are detected and `LEAP_GPU` is unspecified
- report selected accelerator type
- set:
  - `LEAP_CUDA`
  - `LEAP_HIP`
  - `LEAP_CPU_ONLY`

## Checklist

- [x] Top-level `CMakeLists.txt` updated.
- [x] `LEAP_GPU` cache values set.
- [x] NVIDIA path configures or fails clearly: current environment has no CUDA compiler, and `LEAP_GPU=NVIDIA` fails with `LEAP_GPU=NVIDIA requires a CUDA compiler`.
- [x] AMD path configures and builds explicitly with `LEAP_GPU=AMD`.
- [x] AMD path configures and builds implicitly when HIP is detected and CUDA is not detected.
- [x] None path configures and builds with `LEAP_GPU=None`.
- [x] This document updated.
- [x] Commit written: b001ee0.

## Commit message

```text
Add LEAP_GPU backend selection to version_two

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

# Phase 7 — Adapt `version_two/src/CMakeLists.txt`

Use the aligned packaging branch as reference, but preserve `version_two`'s larger source set.

## Must include `version_two`-only native sources

```text
src/physics/**
src/inpainting.cpp
src/segmentation.cpp
src/statistics.cpp
src/finite_difference_filters.cpp
src/maximally_flat_filter.cpp
src/ring_removal.cpp
src/texture_compat.h
```

## Port packaging behavior

- `find_package(OpenMP REQUIRED COMPONENTS CXX)`
- CUDA via `CUDAToolkit`
- HIP via `hip` and `hipfft`
- HIPify handling
- target compile definitions:
  - CUDA: `__USE_GPU`, `__INCLUDE_CUFFT`
  - HIP: `__USE_GPU`, `__USE_NOTEX`, `__INCLUDE_CUFFT`
  - CPU: `__USE_CPU`
- scikit-build install into `leapctype/`

## Checklist

- [x] Source lists updated for `version_two` layout.
- [x] CPU-only source set builds without CUDA/HIP sources.
- [x] CUDA/HIP source set includes expected `.cu` files, including `version_two` physics/billiards and moved subdirectory sources.
- [x] HIPified source mapping handles subdirectories; explicit and implicit AMD builds completed after adding the missing `vector_types.h` include to `ray_tracing/analytic_ray_tracing_gpu.cuh`.
- [x] scikit-build install destination checked: native library now installs into `leapctype/` for package builds.
- [x] This document updated.
- [x] Commit written: 3c9fa07.

## Commit message

```text
Update version_two CMake build for packaged GPU backends

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

# Phase 8 — Apply CPU-only filtered backprojection fix

Apply the packaging branch's CPU-only filtered backprojection fix to the moved path:

```text
src/fbp/filtered_backprojection.cpp
```

not the old flat path:

```text
src/filtered_backprojection.cpp
```

## Checklist

- [x] Locate equivalent packaging hunk: packaging commit `bbd30e1` guarded `zeroPadForOffsetScan_GPU` and `cudaFree(g_pad)` in the old flat `src/filtered_backprojection.cpp` path.
- [x] Apply to `src/fbp/filtered_backprojection.cpp`: equivalent guards are already present in `version_two`; no additional code changes were needed.
- [x] CPU-only configure/build validation run with `LEAP_GPU=None` and explicit ROCm 6.4.3 compilers; build passed.
- [x] This document updated.
- [ ] Commit written.

## Commit message

```text
Fix CPU-only filtered backprojection build

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

# Phase 9 — Port README and workflow documentation

Port and adapt:

- wheel install docs
- editable install docs
- build wheel docs
- `LEAP_GPU=None`
- `LEAP_GPU=NVIDIA`
- `LEAP_GPU=AMD`
- GPU wheel helper usage
- cross-runtime smoke testing docs

## Checklist

- [x] README updated with wheel install, editable install, build wheel, `LEAP_GPU=None`, `LEAP_GPU=NVIDIA`, `LEAP_GPU=AMD`, and GPU wheel helper usage.
- [x] `docs/hipify-strategy.md` referenced.
- [x] `scripts/cross-runtime-smoke-test.md` referenced.
- [x] This document updated.
- [ ] Commit written.

## Commit message

```text
Document wheel and backend build workflows

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

# Phase 10 — Validation and comparison pass

## CPU validation

```bash
python -m pip install -e . -Ccmake.define.LEAP_GPU=None
python - <<'PY'
import leapctype
import xrayphysics
m = leapctype.tomographicModels(only_cpu=True)
m.about()
PY
```

## Wheel validation

```bash
python -m build --wheel -Ccmake.define.LEAP_GPU=None .
```

Install into a clean venv and verify native library placement.

## CUDA validation, if available

```bash
python -m build --wheel -Ccmake.define.LEAP_GPU=NVIDIA .
```

## HIP validation, if available

```bash
python -m build --wheel -Ccmake.define.LEAP_GPU=AMD .
```

## Cross-runtime validation, if available

```bash
scripts/cross-runtime-smoke-test.sh \
  --build-module rocm/6.3.1hangfix \
  --run-module rocm/6.4.3leakfix \
  --gpu AMD \
  --variant rocm6.3
```

## Final comparison checks

Compare source layout:

```bash
git ls-tree -r --name-only eirrgang-packaging-src-layout-alignment:src | sort > /tmp/packaging-src.txt
git ls-tree -r --name-only version-two-packaging-port:src | sort > /tmp/version-two-src.txt
diff -u /tmp/packaging-src.txt /tmp/version-two-src.txt
```

Compare same-path files:

```bash
comm -12 /tmp/packaging-src.txt /tmp/version-two-src.txt
```

Use the common-path list to run targeted diffs on directly comparable files.

## Commit or pause

If validation passes, commit any final fixes and update this document.

If validation fails in a way requiring a design choice, pause and seek user approval.

---

# 11. Source files not expected to be directly comparable

After Phase 0 and the `version_two` port, most shared core files should have the same path. The following categories will still not be directly comparable.

## 11.1 `version_two`-only source files

These exist in `version_two` but not in `eirrgang-packaging`, because they came from the XRayPhysics / feature expansion and later cleanup:

```text
src/physics/billiards.cu
src/physics/billiards.cuh
src/physics/dual_energy_decomposition.cpp
src/physics/dual_energy_decomposition.h
src/physics/xrayphysics.cpp
src/physics/xrayphysics.h
src/physics/xrayphysics_c_interface.cpp
src/physics/xrayphysics_c_interface.h
src/physics/xscatter.cpp
src/physics/xscatter.h
src/physics/xscatter_raw.cpp
src/physics/xscatter_raw.h
src/physics/xsec.cpp
src/physics/xsec.h
src/physics/xsec_raw.cpp
src/physics/xsec_raw.h
src/physics/xsource.cpp
src/physics/xsource.h

src/xrayphysics.py
src/leapctserver.py

src/inpainting.cpp
src/inpainting.h
src/segmentation.cpp
src/segmentation.h
src/statistics.cpp
src/statistics.h
src/finite_difference_filters.cpp
src/finite_difference_filters.h
src/maximally_flat_filter.cpp
src/maximally_flat_filter.h
src/ring_removal.cpp
src/ring_removal.h
src/texture_compat.h

src/fbp/pocketfft_hdronly.h
src/fbp/ramp_filter_hip.cuh
src/geometry/geometric_calibration_cpu.cpp
src/geometry/geometric_calibration_cpu.h
```

Treat these as additive `version_two` functionality, not packaging-port conflicts.

## 11.2 `eirrgang-packaging`-only source files

These exist in `eirrgang-packaging` but not directly in `version_two`:

```text
src/cpu_CMakeLists.txt
src/cuda_utils.h_backup
src/cuda_utils.h_backup_working
src/scatter_models_old.cu
```

Recommended handling:

- Do not port backup files into `version_two`.
- Exclude backup files from sdist/wheels.
- Treat `scatter_models_old.cu` as non-comparable unless the user explicitly wants to preserve it.
- `cpu_CMakeLists.txt` should not be needed once both branches use `LEAP_GPU=None`.

## 11.3 Python module/package layout differences

If `version_two` converts these to packages, they become comparable by path:

```text
src/leapctype/__init__.py
src/leaptorch/__init__.py
src/leap_filter_sequence/__init__.py
src/leap_preprocessing_algorithms/__init__.py
```

If `version_two` also converts these, they become internally consistent but still mostly `version_two`-specific:

```text
src/xrayphysics/__init__.py
src/leapctserver/__init__.py
```

## 11.4 Files comparable by path but not by feature scope

These should share paths after restructuring, but diffs will still be large because `version_two` contains real feature changes:

```text
src/CMakeLists.txt
src/cpu_utils.cpp
src/cpu_utils.h
src/cuda_utils.cu
src/cuda_utils.h
src/fbp/filtered_backprojection.cpp
src/fbp/ramp_filter.cu
src/projectors/projectors_SF.cu
src/projectors/projectors_attenuated.cu
src/projectors/projectors_extendedSF.cu
src/scatter_models.cu
src/scatter_models.cuh
src/tomographic_models.cpp
src/tomographic_models.h
src/tomographic_models_c_interface.cpp
src/tomographic_models_c_interface.h
```

These are directly comparable by path, but review should distinguish packaging/build changes from algorithmic changes.

---

# 12. Packaging branch changes to preserve

The aligned packaging branch should still preserve the functional changes from `notex..eirrgang-packaging`:

| Area | Commits | Summary |
|---|---:|---|
| CMake cleanup | `89dd918`, `18f775b` | Sort source lists; require only OpenMP CXX component. |
| Modern packaging | `de3a1c6`, `0577a9c`, `945ce52` | Replace legacy `setup.py`/`setup.cfg` packaging with `pyproject.toml` using `scikit-build-core` and `setuptools_scm`; curate sdist/wheel contents. |
| GPU backend selection | `0faa6b9`, `6effaa9`, `bb7145f` | Introduce `LEAP_GPU=NVIDIA|AMD|None` autodetection and reporting. |
| CPU-only fix | `bbd30e1` | Fix CPU-only filtered backprojection build. |
| HIP/ROCm support | `b971ccd`, `c45915c`, `a91a43d` | Vendor `third_party/hipify_torch`, add `cmake/LeapHipify.cmake`, and wire HIP source generation into AMD builds. |
| Wheel helpers | `eebc6f9`, `28285b1` | Add GPU wheel version wrapper and cross-runtime smoke test helper. |
| Shared library loading | `26762f1`, `ec8a80c` | Load the installed LEAP shared library via Python package resources instead of relying on build-tree paths. |
| Docs | `5e1a894`, `bb7145f` | Document wheel, editable install, GPU backend, and autodetection behavior. |

---

# 13. Validation results

Append validation results here as they are run.

## Phase 0 validation

Include-style comparison notes:

```text
2026-08-04: Compared quoted includes in common source paths against `version_two`. Moved cross-subdirectory include prefixes now match `version_two` in files such as `src/fbp/filtered_backprojection.cpp`, `src/fbp/filtered_backprojection.h`, `src/geometry/rebin.cpp`, `src/tomographic_models.cpp`, `src/tomographic_models.h`, and `src/tomographic_models_c_interface.cpp`. Remaining differences are additive `version_two` feature includes (for example physics/statistics/inpainting/ring-removal and newer CPU helper includes) or packaging-branch-only legacy differences; no additional path-prefix-only mismatches were found for moved headers.
```

```text
2026-08-04: Initial direct CMake CPU configure attempt with default compiler failed before project configure because `/usr/lib64/ccache/c++` tried to create `/g/g11/eirrgang/.ccache/tmp` on a read-only filesystem. Per user guidance, subsequent validation avoided ccache and set `CC`/`CXX` explicitly to ROCm 6.4.3 compilers.

2026-08-04: Direct CPU-only CMake validation passed after setting explicit compilers:
  rm -rf /tmp/leap-packaging-cpu
  CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ cmake -S . -B /tmp/leap-packaging-cpu -DLEAP_GPU=None
  CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ cmake --build /tmp/leap-packaging-cpu -j
Result: configure and build completed; `libleapct.so` built. One pre-existing warning remained in `src/tomographic_models.cpp` about implicit conversion of NULL to bool.

2026-08-04: Direct AMD HIP CMake validation passed with explicit ROCm 6.4.3 compilers:
  rm -rf /tmp/leap-packaging-hip
  CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 cmake -S . -B /tmp/leap-packaging-hip -DLEAP_GPU=AMD -DCMAKE_HIP_ARCHITECTURES=gfx90a
  CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 cmake --build /tmp/leap-packaging-hip -j
Result: configure, HIPify, and build completed; `leapct` target built. Build log contained HIP warning noise but no error/failure markers; final marker was `[100%] Built target leapct`.

2026-08-04: Python package front-end CPU-only wheel validation passed using isolated `python -m build` and the local `.wheelhouse/`:
  rm -rf /tmp/leap-phase0-wheel-cpu
  PIP_NO_INDEX=1 PIP_FIND_LINKS=$PWD/.wheelhouse CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 .venv13/bin/python -m build --wheel --outdir /tmp/leap-phase0-wheel-cpu -Ccmake.define.LEAP_GPU=None .
Result: `leapct-1.27.dev32+g7c5d55def-0-py3-none-linux_x86_64.whl` built successfully. Build dependencies were installed from `.wheelhouse/` with `PIP_NO_INDEX=1`.

2026-08-04: Python package front-end AMD HIP wheel validation passed using isolated `python -m build` and the local `.wheelhouse/`:
  rm -rf /tmp/leap-phase0-wheel-amd
  PIP_NO_INDEX=1 PIP_FIND_LINKS=$PWD/.wheelhouse CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 .venv13/bin/python -m build --wheel --outdir /tmp/leap-phase0-wheel-amd -Ccmake.define.LEAP_GPU=AMD -Ccmake.define.CMAKE_HIP_ARCHITECTURES=gfx90a .
Result: `leapct-1.27.dev32+g7c5d55def-0-py3-none-linux_x86_64.whl` built successfully. Build dependencies were installed from `.wheelhouse/` with `PIP_NO_INDEX=1`.
```

## Version-two port validation

```text
2026-08-04 Phase 3: Isolated Python package front-end CPU-only wheel validation passed using the local `.wheelhouse/`:
  rm -rf /tmp/leap-phase3-wheel-cpu
  PIP_NO_INDEX=1 PIP_FIND_LINKS=$PWD/.wheelhouse CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 .venv13/bin/python -m build --wheel --outdir /tmp/leap-phase3-wheel-cpu -Ccmake.define.CPUONLY=ON .
Result: `leapct-1.27.dev9+g114070718.d20260804-0-py3-none-linux_x86_64.whl` built successfully. Wheel contents included flat Python modules (`leapctype.py`, `leaptorch.py`, `leap_filter_sequence.py`, `leap_preprocessing_algorithms.py`, `xrayphysics.py`, `leapctserver.py`) and `lib_cpu/libleapct_cpu.so`.

2026-08-04 Phase 4: Isolated Python package front-end CPU-only wheel validation passed after converting Python modules to package directories:
  rm -rf /tmp/leap-phase4-wheel-cpu
  PIP_NO_INDEX=1 PIP_FIND_LINKS=$PWD/.wheelhouse CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 .venv13/bin/python -m build --wheel --outdir /tmp/leap-phase4-wheel-cpu -Ccmake.define.CPUONLY=ON .
Result: `leapct-1.27.dev10+ge0e8cd1e6.d20260804-0-py3-none-linux_x86_64.whl` built successfully. Wheel contents included package directories (`leapctype/__init__.py`, `leaptorch/__init__.py`, `leap_filter_sequence/__init__.py`, `leap_preprocessing_algorithms/__init__.py`, `xrayphysics/__init__.py`, `leapctserver/__init__.py`) and `lib_cpu/libleapct_cpu.so`.

2026-08-04 Phase 4: Installed the CPU wheel into `/tmp/leap-phase4-smoke` using `.wheelhouse/`; basic imports of `leapctype`, `xrayphysics`, and `leap_filter_sequence` passed. Importing `leap_preprocessing_algorithms` still triggers a default `tomographicModels()` shared-library load that fails because the library is installed to `lib_cpu/`, not package resources; this is expected to be addressed in Phase 5/7.

2026-08-04 Phase 5: Isolated Python package front-end CPU-only wheel validation passed after adapting the loader in `src/xrayphysics/__init__.py`:
  rm -rf /tmp/leap-phase5-wheel-cpu /tmp/leap-phase5-smoke
  PIP_NO_INDEX=1 PIP_FIND_LINKS=$PWD/.wheelhouse CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 .venv13/bin/python -m build --wheel --outdir /tmp/leap-phase5-wheel-cpu -Ccmake.define.CPUONLY=ON .
  .venv13/bin/python -m venv /tmp/leap-phase5-smoke
  PIP_NO_INDEX=1 PIP_FIND_LINKS="$PWD/.wheelhouse /tmp/leap-phase5-wheel-cpu" /tmp/leap-phase5-smoke/bin/python -m pip install leapct
  MPLCONFIGDIR=/tmp/leap-phase5-mpl /tmp/leap-phase5-smoke/bin/python - <<'PY'
  import leapctype
  import xrayphysics
  m = leapctype.tomographicModels(only_cpu=True)
  print(type(m).__name__)
  PY
Result: `tomographicModels` printed. The loader preferred the `leapctype` package resource path, then used the current CMake install-layout fallback at `site-packages/lib_cpu/libleapct_cpu.so` because Phase 7 had not yet moved the native install destination into `leapctype/`. Explicit `lib_dir` loading from the installed `lib_cpu` directory was also tested and passed.

2026-08-04 Phase 6/7: Direct CMake CPU backend validation passed:
  rm -rf /tmp/leap-version-two-cpu
  CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 cmake -S . -B /tmp/leap-version-two-cpu -DLEAP_GPU=None
  CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 cmake --build /tmp/leap-version-two-cpu -j
Result: configure selected `LEAP_GPU selected accelerator type NONE`; build completed with `[100%] Built target leapct`.

2026-08-04 Phase 6/7: Direct CMake explicit AMD backend validation passed:
  rm -rf /tmp/leap-version-two-hip-explicit
  CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 cmake -S . -B /tmp/leap-version-two-hip-explicit -DLEAP_GPU=AMD -DCMAKE_HIP_ARCHITECTURES=gfx90a
  CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 cmake --build /tmp/leap-version-two-hip-explicit -j
Result: configure selected `LEAP_GPU selected accelerator type AMD`; HIPify preserved subdirectories; build completed with `[100%] Built target leapct`.

2026-08-04 Phase 6/7: Direct CMake implicit AMD backend validation passed by omitting `LEAP_GPU` while making sure `enable_language(HIP)` can succeed:
  rm -rf /tmp/leap-version-two-hip-implicit
  CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 cmake -S . -B /tmp/leap-version-two-hip-implicit -DCMAKE_HIP_ARCHITECTURES=gfx90a
  CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 cmake --build /tmp/leap-version-two-hip-implicit -j
Result: configure detected `/opt/rocm-6.4.3/lib/llvm/bin/clang++` as the HIP compiler, selected `LEAP_GPU selected accelerator type AMD`, and build completed with `[100%] Built target leapct`.

2026-08-04 Phase 6/7: NVIDIA backend clear-failure check passed in this non-CUDA environment:
  rm -rf /tmp/leap-version-two-cuda-clear-fail
  CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 cmake -S . -B /tmp/leap-version-two-cuda-clear-fail -DLEAP_GPU=NVIDIA
Result: configure failed with `LEAP_GPU=NVIDIA requires a CUDA compiler`.

2026-08-04 Phase 6/7: Isolated Python package front-end explicit AMD wheel validation passed using `.wheelhouse/`:
  rm -rf /tmp/leap-version-two-wheel-amd-explicit
  PIP_NO_INDEX=1 PIP_FIND_LINKS=$PWD/.wheelhouse CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 .venv13/bin/python -m build --wheel --outdir /tmp/leap-version-two-wheel-amd-explicit -Ccmake.define.LEAP_GPU=AMD -Ccmake.define.CMAKE_HIP_ARCHITECTURES=gfx90a .
Result: configure selected `LEAP_GPU selected accelerator type AMD`; `leapct-1.27.dev12+g1cbad2845.d20260804-0-py3-none-linux_x86_64.whl` built successfully.

2026-08-04 Phase 6/7: Isolated Python package front-end implicit AMD wheel validation passed using `.wheelhouse/` and no `LEAP_GPU` setting:
  rm -rf /tmp/leap-version-two-wheel-amd-implicit
  PIP_NO_INDEX=1 PIP_FIND_LINKS=$PWD/.wheelhouse CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 .venv13/bin/python -m build --wheel --outdir /tmp/leap-version-two-wheel-amd-implicit -Ccmake.define.CMAKE_HIP_ARCHITECTURES=gfx90a .
Result: configure detected the HIP compiler, selected `LEAP_GPU selected accelerator type AMD`, and `leapct-1.27.dev12+g1cbad2845.d20260804-0-py3-none-linux_x86_64.whl` built successfully.

2026-08-04 Phase 7: Isolated Python package front-end CPU wheel validation passed with `LEAP_GPU=None` after installing native libraries into `leapctype/`:
  rm -rf /tmp/leap-version-two-wheel-cpu-none /tmp/leap-version-two-cpu-none-smoke
  PIP_NO_INDEX=1 PIP_FIND_LINKS=$PWD/.wheelhouse CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 .venv13/bin/python -m build --wheel --outdir /tmp/leap-version-two-wheel-cpu-none -Ccmake.define.LEAP_GPU=None .
  .venv13/bin/python -m venv /tmp/leap-version-two-cpu-none-smoke
  PIP_NO_INDEX=1 PIP_FIND_LINKS="$PWD/.wheelhouse /tmp/leap-version-two-wheel-cpu-none" /tmp/leap-version-two-cpu-none-smoke/bin/python -m pip install leapct
  MPLCONFIGDIR=/tmp/leap-version-two-cpu-none-mpl /tmp/leap-version-two-cpu-none-smoke/bin/python - <<'PY'
  import leapctype
  m = leapctype.tomographicModels(only_cpu=True)
  print(type(m).__name__)
  PY
Result: wheel contained `leapctype/libleapct.so`; `tomographicModels` printed, confirming the loader found the native library through the `leapctype` package resource path.

2026-08-04 Phase 8: Compared packaging commit `bbd30e1` against `version_two`'s moved `src/fbp/filtered_backprojection.cpp`. The equivalent CPU-only guards around `zeroPadForOffsetScan_GPU` and `cudaFree(g_pad)` were already present in `version_two`, so no code changes were needed. Direct CPU validation passed:
  rm -rf /tmp/leap-phase8-cpu
  CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 cmake -S . -B /tmp/leap-phase8-cpu -DLEAP_GPU=None
  CC=/opt/rocm-6.4.3/bin/amdclang CXX=/opt/rocm-6.4.3/bin/amdclang++ CMAKE_PREFIX_PATH=/opt/rocm-6.4.3 cmake --build /tmp/leap-phase8-cpu -j
Result: configure selected `LEAP_GPU selected accelerator type NONE`; build completed with `[100%] Built target leapct`.

2026-08-04 Phase 9: README documentation ported/adapted for wheel install, editable install, `python -m build`, `LEAP_GPU=None|NVIDIA|AMD`, GPU wheel helper usage, HIPify strategy docs, and cross-runtime smoke-test docs. Documentation-only change; no build validation required.
```


---

# 14. Open questions

- [ ] Should `version_two` delete canonical `setup.py`, or leave a compatibility stub that points users to `pyproject.toml`?
- [ ] Should `setup_AMD.py`, `setup_cpu.py`, `setup_torch.py`, `setup_old.py`, and `setup_ctype.py` remain tracked as local helper scripts?
- [ ] Should `scatter_models_old.cu` be retained on the aligned packaging branch after the source-layout move?
- [ ] Should the fallback package version be exactly `2.0+untagged`, or should it use another `version_two`-specific identifier?
