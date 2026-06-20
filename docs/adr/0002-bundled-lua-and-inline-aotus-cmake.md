# Bundle Lua 5.3.5 and describe aotus sources inline in CMake

The Fortran bindings in `aotus/LuaFortran/` target the Lua 5.3 C API specifically. Allowing a system Lua introduces a version-check burden and breaks on HPC clusters where Lua is absent or mismatched. We therefore always build Lua from the vendored source at `aotus/external/lua-5.3.5/`. A `FLOOK_USE_SYSTEM_LUA` escape hatch can be added later if needed.

`aotus` is a git submodule whose upstream build system is waf. Rather than invoking waf from CMake (fragile, requires Python) or modifying the submodule to add its own CMakeLists.txt (couples us to upstream), we describe aotus's sources directly in flook's own CMake tree. This is safe because flook pins the submodule and treats aotus as an internal dependency, not a separately versioned package.

## Considered options

- **System Lua** — rejected: version fragility, absent on most HPC targets.
- **ExternalProject_Add wrapping waf** — rejected: requires Python, produces opaque imported targets, breaks parallel builds.
- **CMakeLists.txt inside the aotus submodule** — rejected: couples us to an upstream project we do not maintain; submodule changes would need to be upstreamed separately.
