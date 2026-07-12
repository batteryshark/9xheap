# 9xheap

A 32-bit Windows compatibility shim that restores Windows 9x local/global heap
behavior for legacy applications running on modern Windows (XP through 11) and
Wine/Crossover.

Some Win9x-era applications depend on memory-manager quirks that later Windows
versions changed or dropped: `LocalAlloc`/`GlobalAlloc` flag semantics, heap
fragmentation patterns, and process-heap assumptions. 9xheap loads a small DLL
into the target process, hotpatches the relevant heap APIs, and routes emulated
allocations through a ported Win9x heap implementation. Calls that belong to
other heaps fall through to the original system functions unchanged.

The build targets 32-bit x86 applications. The hotpatch stub and pointer
handling are not x64-safe.

## Layout

```
core/    Win9x heap implementation, built as a static library (no CRT, no hooks)
  9xheap_core_api.h     Public core API
  9xheap_core.c         Core facade over the emulated heap
  emulateheap_*.c/.h    Ported Win9x heap and local/global memory code
shim/    Hooking DLL that patches Win32/NT heap APIs
  shim_entry.c          DllMain entry point
  shim_minimal.c        Hook orchestration and API resolution
  binding_helpers.c     Real-process-heap capture and validation
  bindings_kernel32.c   kernel32 heap hooks
  bindings_ntdll.c      ntdll heap hooks
  mem.c                 32-bit x86 hotpatch engine
  x86_ldasm_precompiled.h  Instruction-length disassembler (Apache-2.0)
tests/   test_static.c  Static-import load smoke test
```

The core carries no dependency on the hook mechanism, so the heap emulation can
be reused with a different hooking front end. See [`ARCHITECTURE.md`](ARCHITECTURE.md)
for the execution path, component responsibilities, and the high-risk areas
(DllMain hooking, hotpatch installation, PEB fallbacks).

## Building

Requires MinGW-w64 (`i686-w64-mingw32-gcc`) and make.

Ubuntu/Debian (including WSL):

```bash
sudo apt install gcc-mingw-w64-i686 make
make
```

MSYS2 `mingw32` shell (tools are unprefixed there):

```bash
mingw32-make CROSS_PREFIX=
mingw32-make CROSS_PREFIX= test
```

Build output (git-ignored):

- `dist/9xheap.dll` — the shim DLL
- `build/lib9xheap_core.a` — core static library
- `build/lib9xheap.dll.a` — import library

For debug logging, rebuild with `make CFLAGS="-DDEBUG -g"`.

## Usage

The DLL exports one function, `fix_9xheap()`. Hooks install on process attach,
so the export mainly serves as a load trigger for static import.

Static import (link against `lib9xheap.dll.a`):

```c
__declspec(dllimport) void fix_9xheap(void);

int main(void) {
    fix_9xheap();  // loads the DLL and installs hooks
    HLOCAL mem = LocalAlloc(LMEM_FIXED, 1024);  // now uses 9x behavior
    // ...
}
```

Dynamic load:

```c
HMODULE h9x = LoadLibrary("9xheap.dll");
void (*fix_fn)(void) = (void*)GetProcAddress(h9x, "fix_9xheap");
if (fix_fn) fix_fn();
```

Or inject `9xheap.dll` into the target process by any means.

## Hooked functions

- kernel32: `LocalAlloc`, `LocalFree`, `LocalReAlloc`, `GlobalAlloc`,
  `GlobalFree`, `GlobalReAlloc`, `GetProcessHeap`
- ntdll: `RtlAllocateHeap`, `RtlFreeHeap`

APIs are resolved through API-set names first, then `kernelbase.dll`, then
`kernel32.dll`, so the shim works on modern API-forwarding builds while keeping
`kernel32.dll` as the floor for older systems.

## How it works

`InitializeRealProcessHeap()` captures the real process heap before
`GetProcessHeap` is patched, then validates candidates with `HeapValidate()`. If
the direct call is unavailable it falls back to `ProcessEnvironmentBlock->Reserved4`
slots, validating each.

`mem.c` uses `x86_ldasm` to measure whole instructions in a function prologue so
the trampoline never splits an instruction, then overwrites the prologue with a
`push address; ret` stub. It flushes the instruction cache and restores page
protection afterward. This path is 32-bit x86-specific.

Each hook checks whether an allocation belongs to the emulated heap and, if not,
delegates to the saved original function pointer.

## Testing

```bash
make test
```

The test loads the DLL through static import, confirms `fix_9xheap()` is
callable, and checks that a `LocalAlloc`/`LocalFree` sequence survives hook
installation.

## Troubleshooting

- **DLL fails to load** — confirm the target process is 32-bit and that
  `kernel32.dll` and `ntdll.dll` resolve.
- **Hooks have no effect** — confirm `fix_9xheap()` ran and that the application
  calls the hooked APIs rather than a statically linked CRT allocator.
- **Memory corruption** — usually mixed heap ownership (9x-heap pointer freed
  through a system path). Confirm the real process heap was captured correctly.

## License

The tree combines code under different terms:

- `core/emulateheap_*` — Windows 95 heap code, `(C) Microsoft Corp.` (see file
  headers).
- `shim/x86_ldasm_precompiled.h` — Apache License 2.0.
- The remaining shim and build glue — project-specific.

Check individual source headers for the terms that apply to each file.
