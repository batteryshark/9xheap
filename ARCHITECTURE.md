# 9xheap Architecture

9xheap is a 32-bit Windows compatibility shim for applications that depend on Windows 9x heap and local/global memory behavior. It is best understood as two layers:

- `core/`: a mostly self-contained Win9x heap implementation adapted for MinGW.
- `shim/`: a DLL that hotpatches selected Win32 and NT heap APIs so legacy callers reach the core when they are using emulated heap objects.

The current project name is historically useful because it matches the exported DLL and original subject matter. If this is renamed for public or long-term use, prefer a descriptive repository name such as `legacy-heap-compat` while keeping the binary/export names stable unless every consumer is updated.

## Smallest Execution Path

1. A process loads `9xheap.dll`, either by static import, `LoadLibrary`, or injection.
2. `shim/shim_entry.c` receives `DLL_PROCESS_ATTACH` and calls `init_shim_minimal()`.
3. `shim/shim_minimal.c` captures the real process heap before hook installation, initializes the core heap system, resolves target APIs, and hotpatches them.
4. `shim/mem.c` uses `x86_ldasm_precompiled.h` to copy whole x86 instructions into a trampoline before overwriting the target prologue with a `push address; ret` stub.
5. Hook handlers in `shim/bindings_kernel32.c` and `shim/bindings_ntdll.c` decide whether each call belongs to the emulated heap. Calls for other heaps fall back to the original function pointers.
6. The emulated behavior ultimately lives in `core/emulateheap_heap.c`, `core/emulateheap_lmem.c`, and `core/emulateheap_support.c`, exposed to the shim through `core/emulateheap_kernel32.h` and wrapped by `core/9xheap_core_api.h`.

## Component Responsibilities

`core/9xheap_core_api.h` and `core/9xheap_core.c`

Public core facade. This separates reusable heap behavior from the DLL patching mechanism. Some API-like operations are explicit stubs that return `ERROR_CALL_NOT_IMPLEMENTED`; that is preferable to pretending unsupported heap operations are implemented.

`core/emulateheap_*.c`

Ported compatibility heap implementation. These files carry historical Windows memory-manager assumptions and should be changed conservatively.

`shim/shim_entry.c`

Minimal DLL entrypoint. Hook installation happens on process attach. The exported `fix_9xheap` function is intentionally a no-op because static import is used as a load trigger.

`shim/shim_minimal.c`

Hook orchestration. Despite the "minimal" name, this file intentionally resolves modern API-set DLL names first and falls back through `kernelbase.dll` and `kernel32.dll`. This mirrors the old working shim behavior while still allowing older systems to resolve the real kernel32 exports.

`shim/binding_helpers.c`

Heap ownership and safety helpers. `InitializeRealProcessHeap()` is load-bearing: it captures the real process heap before `GetProcessHeap` is replaced and uses PEB fallbacks only if needed.

`shim/bindings_kernel32.c` and `shim/bindings_ntdll.c`

Hook handlers. These route emulated allocations to the core and preserve normal process behavior by delegating non-emulated heaps to the original trampoline functions.

`shim/mem.c`

32-bit hotpatch engine. It depends on instruction-length disassembly so the trampoline does not split an x86 instruction. This project targets 32-bit applications; the stub and pointer assumptions are not x64-safe.

`tests/test_static.c`

Smoke test for the static-import load path. It verifies that the DLL loads, `fix_9xheap()` is callable, and a basic `LocalAlloc`/`LocalFree` sequence survives after hook installation.

## Compatibility Boundaries

- The shim is for 32-bit Windows processes. The Makefile builds with `i686-w64-mingw32-gcc`, `-m32`, and an x86 hotpatch stub.
- The core is intended to stay independent of the hook mechanism.
- API-set names in `shim_minimal.c` are not accidental clutter. They are part of the compatibility strategy for modern Windows, with fallback resolution for older targets.
- Heap routing relies on preserving the real process heap and validating non-emulated pointers before falling back to NT heap calls.
- Hooking from `DllMain` is simple for consumers but raises loader-lock and partial-initialization risk. Treat changes around `DllMain`, `LoadLibraryW`, and hotpatch installation as high-risk.
- Individual hook failures are currently tolerated because `shim_minimal.c` does not check every `hotpatch_function()` result. Partial installation may be intentional compatibility behavior, but it should be made observable before being made stricter.
- `shim/mem.c` patches executable code, flushes the instruction cache, and restores page protection. Treat this path as compatibility-critical because the trampoline and patch stub are 32-bit x86-specific.
- `GetAppCompatFlags()` currently returns `0` even though the local-memory code checks appcompat-style heap slack flags. Re-enabling those flags would be a behavioral change.
- Several inherited include names in the MinGW patch use different casing than the checked-in lowercase files. Keep case sensitivity in mind if moving the project to Linux, WSL, or a case-sensitive checkout.

## Build and Verification

Use MSYS2 or another MinGW-w64 environment with `i686-w64-mingw32-gcc`, `i686-w64-mingw32-ar`, `i686-w64-mingw32-strip`, and make-compatible tooling.

```sh
make
make test
```

Inside the MSYS2 `mingw32` shell, the tools may be available as unprefixed `gcc`, `ar`, and `strip`. Use:

```sh
make CROSS_PREFIX=
make CROSS_PREFIX= test
```

Build output is intentionally ignored:

- `dist/9xheap.dll`
- `build/lib9xheap_core.a`
- `build/lib9xheap.dll.a`
- `build/test_static.exe`

## Current Comprehension Notes

- High confidence: the repo is organized around a core library plus a hooking DLL, and `fix_9xheap` exists to force/load-trigger the DLL.
- High confidence: the hotpatch engine is x86-only.
- Requires author confirmation: whether hook installation from `DllMain` is an intentional constraint for static-import ergonomics or just inherited from the old shim.
