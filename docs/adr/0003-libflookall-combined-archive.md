# Keep libflookall combined static archive

Many downstream consumers of flook (Fortran simulation codes) link via raw Makefiles or autotools and rely on a single `-lflookall` to pull in lua + aotus + flook without managing transitive dependencies. Dropping `libflookall.a` would break these workflows. We therefore keep it as an opt-in CMake target (`flookall`) built by merging the individual archives with `ar`, and update `flook.pc` to list the individual libraries (`-lflook -laotus -llua -ldl`) so non-CMake consumers that do not use `libflookall` have a correct link line.

CMake consumers get proper target propagation via `target_link_libraries` and do not need `libflookall.a` at all.
