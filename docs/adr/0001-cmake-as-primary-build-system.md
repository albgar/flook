# CMake as primary build system

The existing smeka/GNU Make build system requires manual compiler configuration via `setup.make` and has no standard discovery mechanism for downstream consumers (`find_package`, IDE integration, etc.). We are replacing it with CMake (minimum 3.14) as the primary build system. The smeka Makefiles are retained in-tree but no longer the recommended path. CMake was chosen over alternatives (Meson, waf) because it is the de facto standard in the HPC/scientific Fortran ecosystem and has the widest toolchain and IDE support.

## Considered options

- **Keep smeka** — no migration cost, but blocks CI standardisation and downstream CMake consumers.
- **Meson** — good Fortran support but low adoption in the target (HPC) user base.
- **waf** — already used by aotus upstream, but Python-based, non-standard, and hard to compose with downstream projects.
