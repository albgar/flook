cmake_minimum_required(VERSION 3.14)

project(flook
    VERSION 0.8.4
    LANGUAGES C Fortran
)

include(GNUInstallDirs)
include(CheckCSourceCompiles)
include(CMakePackageConfigHelpers)

# ---------------------------------------------------------------------------
# Options
# ---------------------------------------------------------------------------
option(FLOOK_OO          "Build the Fortran 2003 object-oriented API" OFF)
option(FLOOK_BUILD_TESTS "Build and register tests with CTest"        ON)
# BUILD_SHARED_LIBS is a standard CMake variable; default is OFF (static).

# ---------------------------------------------------------------------------
# Source tree roots
# ---------------------------------------------------------------------------
set(_lua_src    "${CMAKE_CURRENT_SOURCE_DIR}/aotus/external/lua-5.3.5/src")
set(_flu_dir    "${CMAKE_CURRENT_SOURCE_DIR}/aotus/LuaFortran")
set(_aotus_src  "${CMAKE_CURRENT_SOURCE_DIR}/aotus/source")
set(_flook_src  "${CMAKE_CURRENT_SOURCE_DIR}/src")

# All Fortran object libraries write .mod files to this single directory so
# that inter-library USE statements resolve without extra include flags.
set(CMAKE_Fortran_MODULE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/modules")

# ---------------------------------------------------------------------------
# POSIX feature detection — controls LUA_USE_POSIX compile definition
# ---------------------------------------------------------------------------
check_c_source_compiles([=[
    #include <stdlib.h>
    #include <unistd.h>
    #include <stdio.h>
    #include <math.h>
    int main(void) { return 0; }
]=] _lua_posix_headers)

if(_lua_posix_headers)
    check_c_source_compiles([=[
        #include <stdlib.h>
        int main(void) { char t[] = "XXXXXX"; mkstemp(t); return 0; }
    ]=] _lua_have_mkstemp)
    check_c_source_compiles([=[
        #include <stdio.h>
        int main(void) { FILE *f = popen("ls","r"); return 0; }
    ]=] _lua_have_popen)
    check_c_source_compiles([=[
        #include <stdlib.h>
        int main(void) { srandom(42); return 0; }
    ]=] _lua_have_srandom)
endif()

if(_lua_have_mkstemp AND _lua_have_popen AND _lua_have_srandom)
    set(_lua_use_posix ON)
endif()

# ---------------------------------------------------------------------------
# Lua 5.3.5 — bundled C sources
# lua.c and luac.c are the standalone interpreter/compiler; excluded here.
# ---------------------------------------------------------------------------
add_library(lua_objs OBJECT
    ${_lua_src}/lapi.c    ${_lua_src}/lcode.c   ${_lua_src}/lctype.c
    ${_lua_src}/ldebug.c  ${_lua_src}/ldo.c     ${_lua_src}/ldump.c
    ${_lua_src}/lfunc.c   ${_lua_src}/lgc.c     ${_lua_src}/llex.c
    ${_lua_src}/lmem.c    ${_lua_src}/lobject.c ${_lua_src}/lopcodes.c
    ${_lua_src}/lparser.c ${_lua_src}/lstate.c  ${_lua_src}/lstring.c
    ${_lua_src}/ltable.c  ${_lua_src}/ltm.c     ${_lua_src}/lundump.c
    ${_lua_src}/lvm.c     ${_lua_src}/lzio.c
    ${_lua_src}/lauxlib.c  ${_lua_src}/lbaselib.c ${_lua_src}/lbitlib.c
    ${_lua_src}/lcorolib.c ${_lua_src}/ldblib.c  ${_lua_src}/liolib.c
    ${_lua_src}/lmathlib.c ${_lua_src}/loslib.c  ${_lua_src}/ltablib.c
    ${_lua_src}/lstrlib.c  ${_lua_src}/lutf8lib.c ${_lua_src}/loadlib.c
    ${_lua_src}/linit.c
)
target_compile_definitions(lua_objs PRIVATE
    LUA_ANSI
    $<$<BOOL:${_lua_use_posix}>:LUA_USE_POSIX>
)
target_include_directories(lua_objs PUBLIC $<BUILD_INTERFACE:${_lua_src}>)

# ---------------------------------------------------------------------------
# LuaFortran — C wrapper + ISO_C_BINDING Fortran layer
# ---------------------------------------------------------------------------
add_library(luawrap_objs OBJECT ${_flu_dir}/wrap_lua_dump.c)
target_include_directories(luawrap_objs PRIVATE ${_lua_src})

add_library(flu_objs OBJECT
    ${_flu_dir}/lua_fif.f90
    ${_flu_dir}/dump_lua_fif_module.f90
    ${_flu_dir}/lua_parameters.f90
    ${_flu_dir}/flu_kinds_module.f90
    ${_flu_dir}/flu_binding.f90
)

# ---------------------------------------------------------------------------
# aotus — high-level Fortran/Lua interface
# Quad- and extended-double precision use dummy stub modules; enabling the
# real implementations requires Fortran try_compile feature checks (future).
# ---------------------------------------------------------------------------
add_library(aotus_objs OBJECT
    ${_aotus_src}/aotus_module.f90
    ${_aotus_src}/aot_err_module.f90
    ${_aotus_src}/aot_fun_module.f90
    ${_aotus_src}/aot_fun_declaration_module.f90
    ${_aotus_src}/aot_table_module.f90
    ${_aotus_src}/aot_table_ops_module.f90
    ${_aotus_src}/aot_top_module.f90
    ${_aotus_src}/aot_out_module.f90
    ${_aotus_src}/aot_out_general_module.f90
    ${_aotus_src}/aot_path_module.f90
    ${_aotus_src}/aot_references_module.f90
    ${_aotus_src}/aot_vector_module.f90
    ${_aotus_src}/quadruple/dummy_quadruple_fun_module.f90
    ${_aotus_src}/quadruple/dummy_quadruple_table_module.f90
    ${_aotus_src}/quadruple/dummy_quadruple_top_module.f90
    ${_aotus_src}/quadruple/dummy_quadruple_out_module.f90
    ${_aotus_src}/quadruple/dummy_quadruple_vector_module.f90
    ${_aotus_src}/extdouble/dummy_extdouble_fun_module.f90
    ${_aotus_src}/extdouble/dummy_extdouble_table_module.f90
    ${_aotus_src}/extdouble/dummy_extdouble_top_module.f90
    ${_aotus_src}/extdouble/dummy_extdouble_out_module.f90
    ${_aotus_src}/extdouble/dummy_extdouble_vector_module.f90
)
# Explicit ordering so CMake resolves Fortran module dependencies across
# OBJECT library boundaries during parallel builds.
add_dependencies(aotus_objs flu_objs)

# ---------------------------------------------------------------------------
# flook — one source file, selected by FLOOK_OO
# ---------------------------------------------------------------------------
if(FLOOK_OO)
    set(_flook_main "${_flook_src}/flook_f03.F90")
else()
    set(_flook_main "${_flook_src}/flook.F90")
endif()

configure_file(flook.inc.in "${CMAKE_CURRENT_BINARY_DIR}/flook.inc" @ONLY)

add_library(flook_objs OBJECT ${_flook_main})
target_include_directories(flook_objs PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
add_dependencies(flook_objs aotus_objs)

# ---------------------------------------------------------------------------
# PIC — required for all object libraries when building a shared flook
# ---------------------------------------------------------------------------
if(BUILD_SHARED_LIBS)
    set_target_properties(lua_objs luawrap_objs flu_objs aotus_objs flook_objs
        PROPERTIES POSITION_INDEPENDENT_CODE ON)
endif()

# ---------------------------------------------------------------------------
# libflook — combines all object layers into one archive / DSO
# ---------------------------------------------------------------------------
add_library(flook
    $<TARGET_OBJECTS:lua_objs>
    $<TARGET_OBJECTS:luawrap_objs>
    $<TARGET_OBJECTS:flu_objs>
    $<TARGET_OBJECTS:aotus_objs>
    $<TARGET_OBJECTS:flook_objs>
)
target_include_directories(flook
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_Fortran_MODULE_DIRECTORY}>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)
target_link_libraries(flook PUBLIC ${CMAKE_DL_LIBS})

# ---------------------------------------------------------------------------
# libflookall — backward-compatible combined archive
# Identical content to libflook.a; produced as a copy to preserve the name
# expected by existing downstream Makefiles and pkg-config consumers.
# ---------------------------------------------------------------------------
if(NOT BUILD_SHARED_LIBS)
    add_custom_command(
        OUTPUT  "${CMAKE_CURRENT_BINARY_DIR}/libflookall.a"
        COMMAND "${CMAKE_COMMAND}" -E copy
                "$<TARGET_FILE:flook>"
                "${CMAKE_CURRENT_BINARY_DIR}/libflookall.a"
        DEPENDS flook
        COMMENT "Creating libflookall.a"
    )
    add_custom_target(flookall ALL
        DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/libflookall.a"
    )
endif()

# ---------------------------------------------------------------------------
# pkg-config
# ---------------------------------------------------------------------------
if(CMAKE_DL_LIBS)
    set(_pc_libs "-L\${libdir} -lflook -l${CMAKE_DL_LIBS}")
else()
    set(_pc_libs "-L\${libdir} -lflook")
endif()
configure_file(flook.pc.in "${CMAKE_CURRENT_BINARY_DIR}/flook.pc" @ONLY)

# ---------------------------------------------------------------------------
# CMake package config (find_package support)
# ---------------------------------------------------------------------------
configure_package_config_file(
    cmake/flook-config.cmake.in
    "${CMAKE_CURRENT_BINARY_DIR}/cmake/flook-config.cmake"
    INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/flook"
)
write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/cmake/flook-config-version.cmake"
    VERSION     ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

# ---------------------------------------------------------------------------
# Install
# ---------------------------------------------------------------------------
install(TARGETS flook
    EXPORT  flook-targets
    ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
)
if(NOT BUILD_SHARED_LIBS)
    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/libflookall.a"
        DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    )
endif()
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/flook.inc"
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)
install(DIRECTORY "${CMAKE_Fortran_MODULE_DIRECTORY}/"
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
    FILES_MATCHING PATTERN "*.mod" PATTERN "*.MOD"
)
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/flook.pc"
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig"
)
install(EXPORT flook-targets
    FILE        flook-targets.cmake
    NAMESPACE   flook::
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/flook"
)
install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/cmake/flook-config.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/cmake/flook-config-version.cmake"
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/flook"
)

# ---------------------------------------------------------------------------
# Tests
# ---------------------------------------------------------------------------
if(FLOOK_BUILD_TESTS)
    enable_testing()

    set(_tdir "${CMAKE_CURRENT_SOURCE_DIR}/src/test")

    foreach(_t tst_passreturn tst_tbl tst_tbl_ptr tst_exp_flook tst_aot_passreturn)
        add_executable(${_t} ${_tdir}/${_t}.f90)
        # Each test gets its own module output dir so locally-defined modules
        # (e.g. m_array defined in both tst_passreturn and tst_aot_passreturn)
        # don't race to write the same .mod file under parallel builds.
        set_target_properties(${_t} PROPERTIES
            Fortran_MODULE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/modules/tests/${_t}"
        )
        target_link_libraries(${_t} PRIVATE flook)
        add_test(NAME ${_t} COMMAND ${_t} WORKING_DIRECTORY "${_tdir}")
    endforeach()

    if(FLOOK_OO)
        foreach(_t tst_passreturn_f03 tst_tbl_f03 tst_exp_flook_f03)
            add_executable(${_t} ${_tdir}/${_t}.f90)
            set_target_properties(${_t} PROPERTIES
                Fortran_MODULE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/modules/tests/${_t}"
            )
            target_link_libraries(${_t} PRIVATE flook)
            add_test(NAME ${_t} COMMAND ${_t} WORKING_DIRECTORY "${_tdir}")
        endforeach()
    endif()
endif()
