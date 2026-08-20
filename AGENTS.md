# AGENTS.md

This file provides guidance to agentic AI tools when working on the **ExolangTk** project.

## Project Overview

**ExolangTk** is a collection of C99 **header-only** libraries for building compilers and interpreters that need to interoperate with native code. It is split into two toolkits with a strict architectural boundary:

- **InteropTk** — the static "building-block" layer. Models C types, ABI record layout, calling-convention metadata, symbol mangling, declaration parsing, value marshalling, and C-consumable API macros. Contains **no** runtime code-generation or dynamic loading.
- **FFItk** — the dynamic FFI layer. Loads shared libraries, describes and performs foreign calls, generates trampolines, and exposes host-callable closures. **Builds on** InteropTk and must not re-declare type/layout/convention facts already owned by it.

Dependency direction is one-way: `FFItk → InteropTk`. Never introduce an edge from InteropTk back to FFItk.

## Directory Layout

.
├── docs                     # Doxygen configuration and manual sources
│   ├── CMakeLists.txt
│   ├── Doxyfile.in          # Configured at build time (@VAR@ substitution)
│   ├── FrontPage.md         # Doxygen main page
│   ├── layout.xml           # Doxygen navigation layout
│   └── manual               # Long-form manual pages
├── include
│   ├── FFItk                # FFItk module headers (ffi_*.h)
│   ├── FFItk.h              # Umbrella header: includes all of FFItk
│   ├── InteropTk            # InteropTk module headers (itk_*.h)
│   └── InteropTk.h          # Umbrella header: includes all of InteropTk
└── manifests
    ├── FFItk-Modules.yaml    # Source of truth for FFItk modules
    └── InteropTk-Modules.yaml# Source of truth for InteropTk modules


- **The `manifests/*.yaml` files are the source of truth.** The set of modules, their headers, their `provides` symbols, their dependencies, and their stability all originate there. When implementing, read the relevant manifest first and keep the header in sync with it.
- Each module `foo` in InteropTk maps to `include/InteropTk/itk_foo.h`; each module `bar` in FFItk maps to `include/FFItk/ffi_bar.h`.
- Umbrella headers (`InteropTk.h`, `FFItk.h`) include every module header of their toolkit in dependency order.

## Namespaces & Naming

| Toolkit    | Symbol prefix | Macro prefix | Header prefix |
|------------|---------------|--------------|---------------|
| InteropTk  | `itk_`        | `ITK_`       | `itk_`        |
| FFItk      | `ffi_`        | `FFI_`       | `ffi_`        |

- Public functions, types, and enum members use the lowercase prefix (`itk_layout_of`, `ffi_cif`).
- Public macros and include guards use the uppercase prefix (`ITK_LAYOUT_H`, `FFI_CIF_H`).
- Internal helpers not meant for consumers use a double-underscore suffix on the prefix: `itk__` / `ffi__`. Never document these as public API.
- Enum members are prefixed with their type context, e.g. `FFI_ABI_SYSV`, `ITK_TYPE_INT32`.

## Header-Only Rules (STRICT)

These libraries are header-only. Adhere to the following invariants:

1. **No non-inline definitions with external linkage by default.** Every function definition in a header must be either `static inline` or guarded behind an implementation macro (see below).
2. **Single-header implementation pattern.** Support the common opt-in pattern:
   ```c
   #define ITK_LAYOUT_IMPLEMENTATION
   #include <InteropTk/itk_layout.h>
   ```
   - When the `*_IMPLEMENTATION` macro is defined in exactly one translation unit, non-inline function bodies are emitted there.
   - Otherwise only declarations (and `static inline` helpers) are visible.
   - Use `ITK_DEF` / `FFI_DEF` function-qualifier macros that expand appropriately (e.g. `static inline` vs. extern-with-definition) so the same source serves both modes.
3. **Idempotent inclusion.** Every header has an include guard and must be safe to include multiple times and in any order that respects declared dependencies.
4. **No global mutable state** in headers. Any required state must live in caller-provided structs or explicit contexts (e.g. `ffi_library`, `itk_arena`).
5. **No dependency on a build system for correctness.** A consumer copying `include/` into their tree must be able to compile against the headers with only a C99 compiler.
6. **C99 only.** No C11/C++ features. No compiler-specific extensions unless wrapped in a `platform`-module feature macro with a portable fallback.
7. **Freestanding-friendly.** Prefer `<stddef.h>`, `<stdint.h>`, `<stdbool.h>`. Isolate any `<stdlib.h>`/`<string.h>` usage so hosts can override allocation and memory routines (via the `alloc` module).

## Dependency Discipline

- Consult the `depends-on` block in the manifest before adding an `#include`. A module may only include headers of modules listed as its dependencies (internal or cross-toolkit).
- FFItk modules include InteropTk headers via `<InteropTk/itk_*.h>`; the reverse is forbidden.
- Do not create circular includes. If two modules seem mutually dependent, the shared declarations belong in a lower-level module.

## Coding Conventions

- Indentation: 4 spaces, no tabs. Braces on the same line for functions and control flow (K&R).
- Return values: functions that can fail return an `itk_status` / `ffi_status` code; out-parameters carry results. Use the `error` module's conventions in InteropTk.
- Ownership: document ownership transfer explicitly in Doxygen (`@note Ownership ...`). Allocation goes through the `alloc` module's vtables, never raw `malloc` in library code paths that hosts may want to control.
- All public identifiers must appear in the module's `provides` list in the manifest. If you add a new public symbol, update the manifest in the same change.
- Keep executable/generated-memory concerns (W^X, cache flushing) confined to FFItk's `trampoline` module.

## Documentation Requirements (Doxygen)

**Every public declaration must carry a rich Doxygen docstring.** Documentation is not optional and is checked as part of review. Use Javadoc-style `/** ... */` blocks.

Required tags, where applicable:

- `@file` at the top of every header, with `@brief` and a one-paragraph description of the module's role.
- `@defgroup` / `@ingroup` so each header contributes to a module group that matches the manifest module name.
- `@brief` for every function, type, macro, and enum.
- `@param[in]`, `@param[out]`, `@param[in,out]` with direction annotations for every parameter.
- `@retval` for each meaningful status/return value, or `@return` for value-returning functions.
- `@pre` / `@post` for contracts (non-null requirements, initialization state, alignment).
- `@note`, `@warning` for ownership, lifetime, thread-safety, and platform caveats.
- `@sa` cross-references to related symbols (e.g. link `ffi_call` to `ffi_cif_prepare`).
- `@par Example` with a compilable snippet for each primary entry point.
- `@since` referencing the toolkit version (currently `0.1.0`).

### File header template

```c
/**
 * @file itk_layout.h
 * @ingroup itk_layout
 * @brief ABI record layout: size, alignment, and field offset computation.
 *
 * This header models how aggregate types are laid out in memory for a given
 * target ABI. Given a sequence of member types described by the @ref itk_ctypes
 * module, it computes total size, alignment, per-field byte offsets, and any
 * required tail padding, honoring packing directives and natural alignment.
 *
 * @note This is a static building-block module of InteropTk. It performs no
 *       allocation of its own beyond what the caller-supplied @ref itk_arena
 *       provides, and holds no global state.
 * @sa itk_ctypes.h
 * @since 0.1.0
 */
#ifndef ITK_LAYOUT_H
#define ITK_LAYOUT_H
/* ... */
#endif /* ITK_LAYOUT_H */

### Function docstring template

c
/**
 * @brief Compute the ABI layout of an aggregate type.
 *
 * Walks @p members in declaration order, assigning each field a byte offset
 * that satisfies its natural alignment (or the record's packing override),
 * and accumulates the record's overall size and alignment. Tail padding is
 * added so the total size is a multiple of the record alignment.
 *
 * @param[in]  members   Array of @p count member type descriptors. Must be
 *                       non-NULL when @p count is greater than zero.
 * @param[in]  count     Number of members in @p members.
 * @param[in]  packing   Packing directive (0 selects natural alignment).
 * @param[out] out       Receives the computed layout. Must be non-NULL.
 *
 * @retval ITK_OK              Layout computed successfully.
 * @retval ITK_ERR_INVALID     A NULL pointer or malformed member was supplied.
 * @retval ITK_ERR_OVERFLOW    The accumulated size exceeded the addressable range.
 *
 * @pre  Every entry in @p members has been initialized via the @ref itk_ctypes API.
 * @post On success, @p out->offsets contains @p count valid byte offsets.
 *
 * @note This routine is pure and thread-safe: it reads @p members and writes
 *       only through @p out.
 * @warning The lifetime of @p out->offsets follows @p out; it is not heap-owned.
 *
 * @par Example
 * @code
 * itk_type members[] = { ITK_TYPE_INT32, ITK_TYPE_PTR };
 * itk_layout lo;
 * if (itk_layout_of(members, 2, 0, &lo) == ITK_OK) {
 *     printf("size=%zu align=%zu\n", lo.size, lo.align);
 * }
 * @endcode
 *
 * @sa itk_layout, itk_ctypes
 * @since 0.1.0
 */
ITK_DEF itk_status itk_layout_of(const itk_type *members, size_t count,
                                 unsigned packing, itk_layout *out);

Apply the same richness to FFItk (`@ingroup ffi_cif`, etc.), and note in docstrings when a symbol wraps or relies on InteropTk (`@sa itk_callconv`).

## Implementation Workflow for Agents

1. **Read the manifest** for the toolkit you are editing (`manifests/*.yaml`). Treat its `modules`, `provides`, and `depends-on` as authoritative.
2. **Implement lowest-dependency modules first.** Suggested order:
   - InteropTk: `platform` → `ctypes` → `layout` → `callconv` → `mangle` → `cdecl` → `marshal` → `cstring` → `error` → `alloc` → `export`.
   - FFItk: `loader` / `trampoline` → `cif` → `frame` → `call` → `closure` → `library`.
3. **Match header ↔ manifest.** Ensure each `provides` symbol exists with the documented signature, and every new public symbol is added back to the manifest.
4. **Respect stability flags.** For modules marked `experimental`, mark the corresponding symbols with `@warning This API is experimental and may change before 1.0.` in their docstrings.
5. **Keeps build/installation simple.** Ensure files under `include/` are self-contained and ready to compile when standard headers and dependencies are visible in the search paths.
