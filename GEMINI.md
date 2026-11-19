# Project Instructions & Quota Guardrails

This repository is a pure native Metal static recompilation port of *Mirror's Edge (2008, GOG PC)* on macOS ARM64 (`RunnerMetal`).

## Hard Constraints
- **100% Native Metal Rendering (`RunnerMetal`):** Zero Vulkan, DXVK, MoltenVK, Wine, or Rosetta 2 at runtime. `IDirect3D9` / `IDirect3DDevice9` must talk directly to `Metal.framework`.
- **Zero Copyrighted Assets in Git:** `work/`, `_donor/`, `src/lifted/`, `build/`, and GOG installer assets are excluded via `.gitignore`.

## Quota & Context Hygiene Rules
1. **Never** run broad text searches (`grep_search`), recursive listings, or wildcard reads inside `work/`, `_donor/`, `src/lifted/`, or `build/`.
2. Read `_donor/` files **only** via targeted paths when implementing a specific subsystem; read `src/lifted/` **only** around an exact function symbol or LLDB/Clang crash address.
3. **Cap autonomous build-fix loops at 3 consecutive attempts:** If a CMake/Clang error is not resolved after 3 tries, stop and summarize the root error for the user instead of burning tokens on blind retries.
