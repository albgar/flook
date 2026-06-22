# flook

A Fortran library that embeds a Lua interpreter inside a Fortran application, enabling Lua scripts to read and modify Fortran data at designated points in the program's execution.

## Language

### User API

**luaState**:
The Fortran handle for a single embedded Lua environment. Created with `lua_init`, closed with `lua_close`. Multiple luaStates can coexist independently.
_Avoid_: Lua instance, Lua handle, state

**Lua environment**:
The running embedded Lua interpreter owned by a luaState. Script globals and function definitions persist across channels within the same environment.
_Avoid_: Lua shell, Lua VM, Lua instance

**luaTbl**:
A Fortran handle for a currently-open Lua table. Tracks nesting depth as a linked list, allowing traversal of arbitrarily deep table hierarchies via dot-separated names.
_Avoid_: table handle, table reference

**channel**:
A designated point in the Fortran program's execution where `lua_run` is called to invoke a specific Lua function, temporarily handing control to a Lua script. A single luaState can serve many channels.
_Avoid_: hook point, call site, hook

**script load**:
A `lua_run` call that executes a Lua source file to define functions and globals in the Lua environment. Happens once after `lua_init`, before any channels are opened.
_Avoid_: initialization run, file load

**registered function**:
A Fortran subroutine with C interop (`bind(c)`) exposed to a Lua environment via `lua_register`, making it callable by name from within Lua scripts.
_Avoid_: Lua callback, Fortran hook, C-bound procedure

### Implementation layers

The flook library is composed of four layers, bottom to top: bundled Lua → LuaFortran → aotus → flook layer.

**bundled Lua**:
The C Lua 5.3.5 library compiled from `aotus/external/lua-5.3.5/`. Always compiled from source; no system Lua is used.
_Avoid_: Lua (when referring specifically to the C library artifact)

**LuaFortran**:
The lowest Fortran layer (module `flu_binding`); thin wrappers around the C Lua API. Users of flook never interact with this directly.
_Avoid_: flu_binding (use LuaFortran when naming the layer)

**aotus**:
The middle Fortran-Lua binding layer providing higher-level table open/close/get/set operations. Lives in the `aotus/` git submodule.

**flook layer**:
The topmost implementation layer: the `flook` Fortran module that provides `luaState`, `luaTbl`, and all `lua_*` procedures. The name "flook" refers both to this layer and to the user-facing library as a whole.
_Avoid_: flook module (ambiguous; prefer "flook layer" or "flook library" to make the scope clear)
