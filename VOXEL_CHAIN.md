# The `main-behavior1k` branch

The nvblox_torch that `StanfordVL/curobo@78612f45` calls into, plus the one edit needed to
configure it on a modern CUDA toolchain. Branched 2026-09-01 from `NVlabs/nvblox_torch` `main`
@ `e121121`.

## Where the code comes from

cuRobo's dockerfiles clone this repo with no `-b` and no sha, so the "pin" is whatever `main` held
at build time. It has been `e121121` since 2024-07-10, so in practice that is the version.

cuRobo calls `Mapper(voxel_sizes=, integrator_types=, free_on_destruction=, cuda_device_id=)`,
`update_hashmaps`, `update_mesh`, `get_mesh`, `decay_occupancy`, `query_sphere_sdf_cost` and
`query_sphere_trajectory_sdf_cost`. All of those exist here. None of them exists in the in-tree
`nvblox_torch` of `nvidia-isaac/nvblox` `v0.0.9`, which renamed or dropped every one — later nvblox
releases are not a drop-in for cuRobo, even though they build on modern CUDA more easily.

The matching nvblox is `chahyon-ku/nvblox`, branch `v0.0.5-behavior1k`.

## What this branch changes on top of `e121121`

One file, `src/nvblox_torch/cpp/CMakeLists.txt`, build system only. CUDA 12.8 dropped the
`CUDA::nvToolsExt` target that torch's own `Caffe2/public/cuda.cmake` still asks for during
`find_package(Torch)`, so the configure fails before it starts. nvtx3 is header only, so declaring
the target as a header-only interface pointing at `NVTX_INCLUDE_DIR` is all torch needs to find.
It has to precede `find_package(Torch)`.

## Building

The 2024 code does not configure against system CUDA 13.0, and torch is cu128, so the build uses a
`cu128` conda env (nvcc 12.8, cmake 3.26, gcc 11). With `CONDA=~/miniforge3/envs/cu128`, `PREFIX`
the nvblox install prefix, and
`NVTX_INC=$CONDA/nsight-compute-2025.1.1/host/target-linux-x64/nvtx/include/nvtx3`:

* `-DCMAKE_PREFIX_PATH` must lead with torch's own, from
  `python -c 'import torch.utils; print(torch.utils.cmake_prefix_path)'`, then `$PREFIX`, then
  `$CONDA` and `$CONDA/targets/x86_64-linux`.
* `-DCMAKE_CUDA_ARCHITECTURES=<arch>`. This CMakeLists hardcodes 70.
* `-DNVTX_INCLUDE_DIR=$NVTX_INC`, which is what the interface target above reads, plus
  `-I$NVTX_INC` on the compile flags.
* `-I$PREFIX/include -I$PREFIX/include/eigen3 -I$CONDA/include`. Do **not** add
  `-I$PREFIX/include/stdgpu`: `stdgpu/limits.h` then shadows `<limits.h>` and even cmake's
  compiler check fails.
* `-L` and `-Wl,-rpath-link` for `$CONDA/lib`, `$CONDA/targets/x86_64-linux/lib` and
  `$PREFIX/lib`, with
  `-DCMAKE_INSTALL_RPATH="$CONDA/lib;$CONDA/targets/x86_64-linux/lib;$PREFIX/lib"` and
  `-DCMAKE_BUILD_WITH_INSTALL_RPATH=ON`, or nothing loads outside the build tree.

`PRE_CXX11_ABI_LINKABLE` is defined unconditionally here, and nvblox is built with
`-DPRE_CXX11_ABI_LINKABLE=ON` to match. The pair still links against a torch reporting
`_GLIBCXX_USE_CXX11_ABI = True`.

`make` produces `libpy_nvblox.so`, which belongs in `src/nvblox_torch/bin/`, where
`util_file.get_binary_path()` looks for it. That directory is covered by this repo's `*.so`
ignore rule, so the binary is never committed. `pip install -e .` then puts the package in the
venv as `nvblox_torch-0.0.post1.dev1+dirty`.

## Sanity check

Integrate a flat 2 m depth image, render from the same pose, read 2.0 m back. A ray that hits
nothing returns **-1.0**, not 0.
