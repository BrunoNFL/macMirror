---
trigger: always_on
---

# Quota & Context Hygiene Rules

1. **Never** run broad text searches (`grep_search`), recursive listings (`find`, `ls -R`), or wildcard reads inside `work/`, `_donor/`, `src/lifted/`, or `build/`.
2. Read `_donor/` files **only** via targeted paths when implementing or auditing a specific subsystem; read `src/lifted/` **only** around an exact function symbol or LLDB/Clang crash address.
3. **Cap autonomous build-fix loops at 3 consecutive attempts:** If a CMake/Clang error is not resolved after 3 tries, stop and summarize the root error for the user instead of burning tokens on blind retries.
4. Keep `work/`, `_donor/`, `src/lifted/`, `build/`, and GOG installer binaries/assets out of git at all times.
