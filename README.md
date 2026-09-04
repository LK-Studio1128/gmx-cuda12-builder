# GROMACS 2026.3 + CUDA 12.8 — Windows x64 Build via GitHub Actions

[![Build status](https://img.shields.io/github/actions/workflow/status/LK-Studio1128/gmx-cuda12-builder/build-gromacs.yml?branch=main)](https://github.com/LK-Studio1128/gmx-cuda12-builder/actions)
[![GROMACS](https://img.shields.io/badge/GROMACS-2026.3-00b4d8)](https://www.gromacs.org)
[![CUDA](https://img.shields.io/badge/CUDA-12.8.0-76b900)](https://developer.nvidia.com/cuda-toolkit-archive)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

Automated, reproducible **Windows x64 binary builds of GROMACS 2026.3 compiled against the CUDA 12.x toolkit**, driven entirely by a GitHub Actions workflow. The workflow installs the CUDA 12.8.0 toolkit, fetches the official GROMACS source, configures with CMake + Ninja + MSVC (`cl`), and uploads a self-contained engine package as a build artifact — **no local build machine required**.

---

## Why this project exists

Most community-provided **Windows prebuilt packages of GROMACS 2026.x are compiled with CUDA 13.0** (the newest toolkit at the time), yet ship with — and link against — the CUDA 12-series `cufft64_12.dll`. This hybrid combination has caused real-world problems on NVIDIA GPUs:

- `CUDA error #717` / `#719` device resets and NPT-stage segmentation faults observed with **Ampere/Blackwell GPUs + GROMACS 2026.x compiled with CUDA 13**.
- CUDA 13.0's cuFFT release notes list **unresolved correctness issues** for half/bfloat R2C transforms.
- Blackwell (sm_120) requires **CUDA ≥ 12.8**; anything older fails with PTX JIT errors on 50-series cards.

By contrast, the official ecosystem (conda-forge, EESSI) builds 2026.x with **CUDA 12.9** and covers Blackwell `sm_120` — and on a real **RTX 5090** the CUDA 12.x combination runs the full GPU-resident pipeline **stable at 952–1302 ns/day** (see [Validation](#validation)).

**This repository provides the missing piece: an easy, auditable way to produce a *clean CUDA 12.x* Windows build of modern GROMACS — with no local Windows toolchain needed.**

> GROMACS officially requires only **CUDA toolkit ≥ 12.1** — CUDA 13 is *not* required for 2026.x.

---

## What the workflow produces

A self-contained artifact `gromacs-2026.3-cuda12.8-win64` (~230 MB zip) with:

```
├── GMXRC.bat              # environment launcher (GMXLIB/GMXBIN/PATH)
├── bin/
│   ├── gmx.exe            # ~100 MB fat binary — 11 CUDA architectures + PTX
│   ├── cufft64_11.dll     # cuFFT 12.8 runtime — statically embeds the CUDA
│   │                      # runtime (imports only KERNEL32 → no system CUDA
│   │                      # runtime install required)
│   ├── fftw3f.dll         # FFTW3 (vcpkg build, single precision)
│   └── fftw3.dll, fftw3l.dll, GMXRC, demux.pl, xplor2gmx.pl, ...
├── include/               # development headers
└── share/gromacs/top/     # 21 force-field directories incl. amber14sb_OL15/OL3,
                           # charmm36(-jul2022), amber99sb-ildn, oplsaa, ...
```

Key build attributes (verified from `gmx --version`):

| Attribute      | Value                                                    |
|----------------|----------------------------------------------------------|
| GROMACS        | 2026.3 (mixed precision)                                  |
| CUDA compiler  | nvcc **12.8.61** (NVIDIA 12.8.0 toolkit)                 |
| CUDA targets   | `sm_50;52;61;70;75;80;86;89;90;100;120` + PTX `compute_120` (Maxwell → Blackwell) |
| Compiler       | MSVC 19.44 (Visual Studio 2022)                          |
| Parallelism    | **OpenMP enabled** (`GMX_OPENMP=ON`) — mdrun threads via `-ntomp N` on a single MPI rank; thread-MPI also present |
| SIMD           | AVX2_256                                                 |
| FFT            | fftw3 (CPU) / cuFFT (GPU)                                |

### Supported NVIDIA GPUs — by compute capability (CUDA architecture)

Because the toolkit is **CUDA 12.8** (which still supports Maxwell/Pascal/Volta), this
build covers **every CUDA-capable NVIDIA GPU from GTX 9-series (2014) through Blackwell
(2026)**. CUDA 13-based engines cannot do this — CUDA 13 removed Maxwell/Pascal/Volta
support entirely, so a GTX 10-series or V100 can **never** run a CUDA 13 build.

| Compute cap | Architecture | Example GPUs              | Supported | PTX fallback |
|:-----------:|:------------:|---------------------------|:---------:|:------------:|
| `sm_50`     | Maxwell 1st  | GTX 750 / GTX 960         | ✅        | — |
| `sm_52`     | Maxwell 2nd  | GTX 980 / GTX 970 / GTX 950 | ✅      | — |
| `sm_61`     | Pascal       | **GTX 1060 / 1070 / 1080**, GTX 1080 Ti | ✅ | — |
| `sm_70`     | Volta        | **V100**, TITAN V          | ✅        | — |
| `sm_75`     | Turing       | RTX 20-series, GTX 16-series | ✅      | — |
| `sm_80`     | Ampere       | RTX 30-series (A100)       | ✅        | — |
| `sm_86`     | Ampere       | RTX 30-series (consumer)   | ✅        | — |
| `sm_89`     | Ada Lovelace | RTX 40-series              | ✅        | — |
| `sm_90`     | Hopper       | H100 / H200                | ✅        | — |
| `sm_100`    | Blackwell    | B100 / B200                | ✅        | — |
| `sm_120`    | Blackwell    | RTX 50-series              | ✅ native | `compute_120` PTX → future cards |

* `120-virtual` = Blackwell PTX included, so a hypothetical future `sm_121+` device can JIT-compile it.
* Mixed precision / mixed `sm_50`+`sm_61` cards are all handled by the same fat binary.

> **CUDA-13 engines cannot cover the first four rows** (`sm_50/52/61/70`): NVIDIA dropped
> Maxwell/Pascal/Volta from the CUDA 13 toolchain. If your machine has a GTX 10-series or
> a V100, this **CUDA 12.8 all-arch build is the one that works** — that is the whole point
> of producing it.

### Self-contained runtime

`gmx.exe` imports only `cufft64_11.dll`, `fftw3f.dll` and the MSVC runtime. The bundled `cufft64_11.dll` is the CUDA **12.8** cuFFT library (verified via embedded `release 12.8` version string) that **statically embeds the CUDA runtime** — its import table lists only `KERNEL32.dll`. The engine therefore runs on machines **without any system CUDA runtime installed** (verified: `gmx --version` exits 0 on a bare Windows Server with no CUDA on `PATH`). This makes it strictly more portable than typical prebuilts that depend on a system `cudart64_12.dll`.

> **Note on the file name:** the CUDA 12.8.0 toolkit as staged by the `Jimver/cuda-toolkit` action places cuFFT under the name `cufft64_11.dll`. The toolchain links against that name, so the artifact keeps it. The embedded version string confirms it is genuine CUDA 12.8 cuFFT. If you rebuild with a differently-staged toolkit the DLL name may differ — match the artifact to whatever `gmx.exe` imports.

---

## Pipeline overview

| Step | What happens |
|------|--------------|
| 1 | `Jimver/cuda-toolkit@v0.2.21` installs **CUDA 12.8.0** (~13 min) |
| 2 | Download + extract GROMACS 2026.3 tarball from `ftp.gromacs.org` |
| 3 | **Patch OpenMP `find_package`** → `COMPONENTS C CXX` (CMake 3.31 `OpenMP_CUDA` bug; needed because we build `GMX_OPENMP=ON`) |
| 4 | `vcpkg install fftw3:x64-windows` (port name is `fftw3`, not `fftw3f`) |
| 5 | Restore build cache (re-runs skip straight to packaging) |
| 6 | CMake configure with **Ninja** + `vcvars64` + explicit `-DCMAKE_(C|CXX)_COMPILER=cl` |
| 7 | Ninja release build, 11 CUDA architectures + PTX (~4–5 h on the 2-core runner) |
| 8 | `cmake --install`, copy FFTW + cuFFT runtime DLLs, `gmx --version` verify (incl. **OpenMP enabled** check) |
| 9 | Upload artifact (14-day retention) |

### Configure flags used

```bash
cmake -S gromacs-2026.3 -B build -G Ninja ^
  -DCMAKE_C_COMPILER=cl -DCMAKE_CXX_COMPILER=cl ^
  -DGMX_GPU=CUDA ^
  -DGMX_FFT_LIBRARY=fftw3 ^
  -DCMAKE_PREFIX_PATH="C:\vcpkg\installed\x64-windows" ^
  -DGMX_MPI=OFF -DGMX_OPENMP=ON ^
  -DGMX_DOUBLE=OFF -DGMX_HWLOC=OFF ^
  -DCMAKE_BUILD_TYPE=Release ^
  -DCUDAToolkit_ROOT="C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v12.8" ^
  -DCMAKE_CUDA_ARCHITECTURES="50;52;61;70;75;80;86;89;90;100;120;120-virtual" ^
  -DCMAKE_INSTALL_PREFIX="C:\gmxout"
```

> **Note:** GROMACS **2026.3 dropped `GMX_CUDA_TARGET_SM` / `GMX_CUDA_TARGET_COMPUTE`** — CMake warns they are unused. You **must** use `-DCMAKE_CUDA_ARCHITECTURES` instead (e.g. `"60;75;86"`). Use a plain `list` of SM codes to compile those archs natively, or append `-virtual` to a code to add its PTX for JIT forward-compatibility (here `120-virtual` makes the binary tolerant of future Blackwell+ cards).

> **OpenMP is ON and required.** `-DGMX_OPENMP=ON` because the launcher drives `mdrun` as a single MPI rank with `-ntomp N` OpenMP threads. GROMACS's own `gmx --version` is checked during packaging to confirm `OpenMP support: enabled` (the build aborts if it is off). Enabling OpenMP alongside the CUDA language trips a CMake ≥3.31 `OpenMP_CUDA` probe that fails (`missing LIB_NAMES`), so the workflow first patches every `find_package(OpenMP...)` in the GROMACS tree to `COMPONENTS C CXX` (see the *Patch OpenMP find_package* step) — this keeps host OpenMP while skipping the broken CUDA probe.

---

## Validation

- **Windows (bare, no CUDA on PATH):** `gmx --version` → GROMACS 2026.3 / CUDA runtime 12.80 / **exit 0**. The engine loads its cuFFT at startup (a truncated `cufft64_11.dll` produces an immediate `0xC0000403` fail-fast — proof that the DLL chain is genuinely exercised).
- **Linux + RTX 5090 (sm_120), same CUDA-12.x combo** (2026.3, CUDA 12.9): full GPU-resident pipeline stable across EM (l-bfgs), NVT/NPT with position restraints, and 3000-step production runs at **952–1302 ns/day**; CUDA Graphs adds ~10%. The CUDA-13 combinations that crash (`#717`/`#719`, NPT SIGSEGV) were **not** reproducible with CUDA 12.x.
- Force-field/data parity: the artifact's `share/` tree is file-for-file identical to a reference 2026.2 Windows distribution (21 force fields).

---

## Usage

1. **Get the artifact** — open the [Actions tab](https://github.com/LK-Studio1128/gmx-cuda12-builder/actions), pick the latest green run, download `gromacs-2026.3-cuda12.8-win64`.
2. **Use the engine** — either replace an existing bundled `Gromacs*/bin/gmx.exe` tree, or point your application's GROMACS selector at `...\bin\gmx.exe` of the extracted folder.
3. **Sanity check**
   ```bat
   bin\gmx.exe --version
   ```
   Confirm `CUDA compiler: ... (NVIDIA 12.8.61)`.

Because the engine reports a CUDA-12.x runtime, CUDA-runtime-aware launcher logic (e.g. auto-upgrade to full GPU offload, CUDA Graphs on Blackwell) now enables the fast path instead of conservatively degrading.

---

## Customization

Edit `.github/workflows/build-gromacs.yml`:

- **GROMACS version** — change the download URL in *Download GROMACS source*.
- **CUDA toolkit** — change `cuda: '12.8.0'` (Jimver supports 12.8.0; 12.8.1/12.9 are not yet in its version table) and the `CUDAToolkit_ROOT` path.
- **Architectures** — edit `CMAKE_CUDA_ARCHITECTURES` (e.g. drop `50;52;61;70` for a leaner binary if you do not need GTX-9/10-series or V100).
- **More parallelism** — the free runner has only 2 cores; on a self-hosted runner lower nothing and just raise `--parallel`.
- The **build cache** key includes the version — bump it when you change inputs to force a clean rebuild.

---

## Troubleshooting — pitfalls we hit (and fixed)

| Symptom | Root cause | Fix in this workflow |
|---|---|---|
| `nvcc: Host compiler targets unsupported OS` | CMake picked MinGW `gcc` instead of MSVC | `call vcvars64.bat` + explicit `-DCMAKE_(C\|CXX)_COMPILER=cl` |
| vcvars not found | GitHub runners install VS in `Enterprise`, not `Community` | hardcode the `Enterprise` path |
| `GMX_BUILD_OWN_FFTW` rejected | GROMACS forbids auto-FFTW builds under MSVC | install FFTW via `vcpkg install fftw3:x64-windows` |
| CMake 3.31 `OpenMP_CUDA` detection failure (`missing LIB_NAMES`) | With CUDA enabled, CMake ≥3.31 probes `OpenMP_CUDA` during `find_package(OpenMP)` and fails | **patch every `find_package(OpenMP...)` to `COMPONENTS C CXX`** (keeps host OpenMP, skips the CUDA probe) — do **not** just set `GMX_OPENMP=OFF`, because the launcher needs OpenMP threading |
| "All requested archs compiled but run still produced old 7-arch binary" | GROMACS 2026.3 ignores `GMX_CUDA_TARGET_SM`; cache restored a stale tree | use `CMAKE_CUDA_ARCHITECTURES` **and** bump the cache key / drop restore-keys so the arch change forces a full rebuild |
| Official FFTW `dll64` .lib link test fails | Old FFTW import libs are not MSVC-friendly | use vcpkg `fftw3` instead |
| CUDA action `12.8.1`/`12.9` "version not available" | Jimver version table lags | use `12.8.0` |
| `git push` 502s from behind a proxy | flaky git-over-HTTPS | update files via the `gh api` contents endpoint |
| Full rebuild on every run | 2-core runner, ~2.5 h CUDA build (7-arch) / ~4–5 h (all-arch) | `actions/cache` on the `build/` dir (re-runs: ~15 min) |
| Artifact can't run | missing runtime DLLs | copy FFTW + cuFFT DLLs into `bin/`; cuFFT is statically-runtimed |

---

## License

[MIT](LICENSE) — the workflow, not GROMACS itself (GROMACS is LGPL-2.1; CUDA toolkit is NVIDIA EULA).
