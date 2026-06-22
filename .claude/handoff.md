# Handoff: flook CMake refactoring

**Repo:** `https://github.com/ElectronicStructureLibrary/flook`
**Local paths:** `/Users/albertog/code/GITLAB/flook` (primary) · `/Users/albertog/G/flook` (symlink — never copy between them)
**Active branch:** `main--ag-test` (base: `main`)
**Date:** 2026-06-22

---

## What was done this session

### 1. Domain model (`CONTEXT.md`)
Ran `/domain-modeling`. Created `CONTEXT.md` at the repo root with the canonical vocabulary for the project. Key terms: `luaState`, `luaTbl`, `Lua environment`, `channel`, `script load`, `registered function`, and the four implementation layers. Committed in `0537c16`.

### 2. Code review of CMake build
Ran `/code-review high` over the full `main--ag-test` branch diff. Eight finder angles, ten candidates surfaced and verified. See the session transcript for the full JSON output. Three confirmed high/medium issues were fixed immediately; five were deferred (tracked in memory — see `.claude/projects/.../memory/project-cmake-review-open.md`).

### 3. Code-review fixes (`38ca7fe`)
Three confirmed bugs fixed in `CMakeLists.txt` and `flook.pc.in`:

| Fix | Location |
|-----|----------|
| `LUA_ANSI` → `LUA_USE_C89` (renamed in Lua 5.2; was a no-op) | `CMakeLists.txt` line 80 |
| Add `PATTERN "tests" EXCLUDE` to module install rule (prevented private test `.mod` files like `m_array` from being installed into system include dir) | `CMakeLists.txt` line 235 |
| Revert `flook.pc.in` to prefix-relative paths (`${prefix}/@CMAKE_INSTALL_INCLUDEDIR@`) — fixes smeka sed substitution breakage and `--define-prefix` relocation | `flook.pc.in` lines 2–3 |

Build verified: 5/5 tests pass (default build).

### 4. `FLOOK_OO=ON` build fixes (`9329c94`)
Three pre-existing bugs found and fixed when smoke-testing the F03 variant:

| Fix | File | Detail |
|-----|------|--------|
| Dummy argument name mismatch | `src/flook_f03.F90:867` | `tbl_init_` declared arg `state` but body used `lua`; renamed arg |
| Literal integer to `intent(inout)` dummy | `src/test/tst_tbl_f03.f90:79,110` | `tbl%close(lvls=2)` → use local variable |
| Procedural tests guard | `CMakeLists.txt:259` | Wrapped non-OO tests in `if(NOT FLOOK_OO)` — `flook_f03.F90` only exports type-bound procedures, not standalone generics |

Build verified: 3/3 f03 tests pass (`FLOOK_OO=ON`); 5/5 procedural tests still pass (`FLOOK_OO=OFF`).

All commits pushed to `ag/main--ag-test`.

---

## Known gaps / next tasks

1. **Quad / extended-double precision in aotus** — `CMakeLists.txt` uses dummy stub modules. The real `aot_quadruple_*` and `aot_extdouble_*` sources require Fortran `try_compile` checks (analogous to aotus's `wscript`). See comment in `CMakeLists.txt` at the `aotus_objs` target.

2. **CI** — `.travis.yml` exists but is stale. A GitHub Actions workflow using the new CMake build is the natural next step.

3. **Install smoke-test** — `cmake --install build --prefix /tmp/flook-install` has not been verified yet.

4. **`BUILD_SHARED_LIBS=ON` smoke-test** — PIC path is plumbed but was not exercised.

5. **Deferred code-review findings** — five low-severity items remain open (see memory `project-cmake-review-open.md`). Most notable: `SameMajorVersion` for a 0.x library (issue 5), and the `mkstemp` check missing `<unistd.h>` (issue 6).

6. **PR to `master`** — the branch is ready for review; a `/code-review` pass was completed and fixes applied. Consider opening a PR.

---

## Important non-obvious facts

- **`FLOOK_OO` API split**: `flook_f03.F90` only exports `luaState`, `luaTbl`, and `len` as standalone publics — all other procedures are type-bound. The procedural API (`lua_run`, `lua_table`, etc.) does not exist in the OO module. Any new test or consumer code must use the OO style (`lua%run(...)`) when built with `FLOOK_OO=ON`. See memory `project-flook-oo-api-split.md`.

- **smeka is a parallel build path**: the GNU Make / smeka build is still present and functional alongside CMake. Changes to `flook.pc.in` must be compatible with smeka's sed-based substitution in `smeka/Makefile.pkgconfig` (uses short-form `@CMAKE_INSTALL_INCLUDEDIR@`, not `FULL_` variants).

- **Repo paths**: `/Users/albertog/G/flook` is a symlink to `/Users/albertog/code/GITLAB/flook`. Never copy files between them.

---

## Repo context

- **flook** is a Fortran library embedding Lua 5.3.5 via the `aotus` Fortran-Lua binding layer (git submodule at `aotus/`).
- Dependency chain: `lua-5.3.5` (C) → `LuaFortran` / `flu_binding` (Fortran ISO_C_BINDING) → `aotus` (Fortran) → `flook` (Fortran).
- Domain vocabulary: `CONTEXT.md` at repo root. ADRs 0001–0003 at `docs/adr/`.
- `docs/` is gitignored except `docs/adr/`. Generated Doxygen output goes to `docs/` and must stay ignored.

---

## Suggested skills

- **`/tdd`** — if adding quad/extdouble Fortran feature-detection, write the `try_compile` checks test-first.
- **`/diagnosing-bugs`** — if the install or shared-library smoke-tests surface issues.
- **`/code-review`** — run before opening the PR to `master`.
