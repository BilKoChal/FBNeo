# Auto-load save state — how this fork was modified (FBNeo)

This fork of `FBNeo` adds a feature: when a ROM is loaded, the core reads a
small config file next to its own `.dll` and, if enabled, automatically loads a
save state for that ROM. This document explains exactly what was changed so you
can **re-apply it yourself after an upstream update**.

All changes are in a single file:

```
src/burner/libretro/libretro.cpp
```

This is **C++** (compiled as `-std=gnu++98`), unlike the C cores. Everything is
wrapped in `#ifdef _WIN32`, so non-Windows builds are unaffected. See the
"Other platforms" note at the end (this is also why it does **not** run on
RetroArch Android, which uses ARM `.so` cores).

---

## 1. What it does at runtime

1. ROM loads → the core remembers the ROM path and sets a one-shot flag
   (only if the load succeeded).
2. On the **first emulated frame**, the core:
   - finds the folder its own `.dll` lives in,
   - reads `fbneo_libretro.cfg` from that folder (same base name as the core),
   - if `enabled = 1`, builds `<save_path>\<rom-name-without-extension>.<state_ext>`,
   - reads that file, unwraps RetroArch's **RASTATE** container if present,
   - and calls `retro_unserialize()` with the result.
3. Every step is written to `autoload.log` next to the `.dll` for debugging.
   This log is **shared by every core in that folder** (each run line is
   tagged `core=<name>` and timestamped), and it is **trimmed to the latest
   25 runs** at the start of each run so it never grows without bound.

Config file (`fbneo_libretro.cfg`, placed beside the `.dll`):

```ini
enabled   = 1
save_path = G:\RetroBat\saves\fbneo
state_ext = state.auto
```

FBNeo arcade ROMs are zips, so for `mslug.zip` the core loads
`<save_path>\mslug.state.auto`.

---

## 1b. Multiple save folders (multi-system cores)

You can list **several `save_path` lines** in the config. The core tries
them top-to-bottom and loads the **first folder that contains a matching
state file**. This is for multi-system cores that keep states in per-system
folders, e.g.:

```ini
enabled   = 1
save_path = G:\RetroBat\saves\mastersystem\libretro.genesis_plus_gx
save_path = G:\RetroBat\saves\megadrive\libretro.genesis_plus_gx
state_ext = state.auto
```

In `autoload.log` each attempt shows as a `trying :` line, and the one that
matched shows as `found in :`. Up to 16 paths are supported. In the code this
is the `save_paths[]` array in `autoload_config` plus the search loop in
`autoload_try_load_state()` - both are inside the self-contained block, so
copying the block verbatim brings the feature along.

## 2. The three code changes

There are exactly three edits. Two are tiny "hook" snippets; one is the big
self-contained helper block.

### Edit A — the helper block

Paste the entire block delimited by
`/* ---- Config-driven auto-load save state (Windows) ... */` down to its
closing `#endif`. You can copy it verbatim from this fork's `libretro.cpp`
(it currently sits just before `void retro_run()`).

> **CRITICAL placement rule:** the block must appear **before the first** of
> `retro_load_game()` and `retro_run()`, because both use the globals declared
> at the top of the block. In FBNeo, `retro_run()` comes **first**, so the block
> goes right before `retro_run()`. (Wrong placement → `'autoload_state_pending'
> undeclared`.)

#### FBNeo-specific details baked into the block

- **No `<windows.h>`.** Including it in this big C++ file risks clashing with
  FBNeo's own types/macros, so the block **declares the two Win32 functions it
  needs directly** (`GetModuleHandleExA`, `GetModuleFileNameA`) as
  `extern "C" __declspec(dllimport) ... __stdcall`. The `__stdcall` matters for
  the 32-bit build; don't drop it. `HMODULE` is just `void*` here.
- **`log_cb` is a bare function pointer** (`retro_log_printf_t log_cb`), so the
  block calls `log_cb(RETRO_LOG_INFO, ...)` (NOT `log_cb.log(...)`).
- **`retro_serialize_size()` / `retro_unserialize()`** are implemented in
  `retro_memory.cpp`, but they are declared by `<libretro.h>` so they are
  callable from `libretro.cpp` with no extra includes.
- The block includes `<stdio.h>`, `<stdarg.h>`, `<string.h>` and guards
  `MAX_PATH` with `#ifndef`.

### Edit B — trigger on the first frame (inside `retro_run`)

`retro_run()` opens with three bool declarations. Add the one-shot check right
after them:

```cpp
void retro_run()
{
	bool bEnableVideo  = true;
	bool bEmulateAudio = true;
	bool bPresentAudio = true;

#ifdef _WIN32
	if (autoload_state_pending)
	{
		autoload_state_pending = false;
		autoload_try_load_state();
	}
#endif
```

### Edit C — capture the ROM path (end of `retro_load_game`)

`retro_load_game` ends with a single line: `return retro_load_game_common();`.
Replace that line with a version that records `info->path` first and arms the
flag only if the load succeeded:

```cpp
#ifdef _WIN32
	autoload_rom_path[0] = '\0';
	if (info && info->path)
	{
		strncpy(autoload_rom_path, info->path, sizeof(autoload_rom_path) - 1);
		autoload_rom_path[sizeof(autoload_rom_path) - 1] = '\0';
	}
	{
		bool autoload_loaded_ok = retro_load_game_common();
		if (autoload_loaded_ok)
			autoload_state_pending = true;
		return autoload_loaded_ok;
	}
#else
	return retro_load_game_common();
#endif
```

(The actual emulator state is set up inside `retro_load_game_common()`, which
doesn't receive `info`, so the path must be captured here in `retro_load_game`.)

---

## 3. Why RASTATE unwrapping is needed (extra important for FBNeo)

RetroArch wraps the raw core state in a container that begins with the ASCII
text `RASTATE`; the real serialized data is inside the `MEM ` block. The helper
`autoload_extract_rastate()` extracts that block and feeds only its bytes to
`retro_unserialize()`.

FBNeo's `retro_unserialize()` does **not** check the size against
`retro_serialize_size()` — it reads the buffer directly. So if you fed it the
whole file *with* the `RASTATE` header, FBNeo would read the header bytes as
state data and corrupt the load. Unwrapping is therefore mandatory, not just a
size fix.

**Compression:** if RetroArch's *Save State Compression* is ON, the file is
additionally compressed and this code can't read it. Keep that setting **OFF**
for the states you point the feature at, then re-save.

---

## 4. Re-applying after an upstream update

1. Pull/merge the new upstream code.
2. Re-do the three edits. The big block (Edit A) is self-contained and never
   needs changing — paste it back before `retro_run()`. Edits B and C are a few
   lines each.
3. Verify just this file compiles before a full build (fast):

```bash
sudo apt-get update && sudo apt-get install -y mingw-w64 make
cd src/burner/libretro
# capture the exact compile command the Makefile would use, then run it:
make -f Makefile platform=win CC=x86_64-w64-mingw32-gcc CXX=x86_64-w64-mingw32-g++ -n \
  | grep -m1 'libretro.cpp' | sh
# success = libretro.o is produced with no errors
```

4. Then do the full build (next section) or just push and let CI build it.

---

## 5. Building

### A) GitHub Actions (recommended — FBNeo is big)

This repo has `.github/workflows/build.yml`. Push your changes (or run it from
the Actions tab). A full FBNeo build takes roughly **45–90 minutes**, which is
fine for CI. Download the DLL from the run's **Artifacts**
(`fbneo_libretro_x86_64`). A tag like `v1.0` also creates a Release.

### B) Locally with mingw-w64 (slow — needs a fast machine and patience)

```bash
sudo apt-get update && sudo apt-get install -y mingw-w64 make
cd src/burner/libretro
make -f Makefile platform=win \
     CC=x86_64-w64-mingw32-gcc CXX=x86_64-w64-mingw32-g++ -j"$(nproc)"
# produces src/burner/libretro/fbneo_libretro.dll
```

The Windows DLL is built by the Makefile's catch-all (`platform=win`) branch.
Expect thousands of object files and a multi-step link (`SPLIT_UP_LINK`).

---

## 6. Installing & using

1. Put `fbneo_libretro.dll` in RetroArch's `cores` folder.
2. Put `fbneo_libretro.cfg` (edited for your paths) in the **same** folder.
3. Make sure a matching state exists at
   `<save_path>\<rom-name-without-extension>.<state_ext>`.
4. Launch a ROM. Check `autoload.log` (next to the DLL):

```
looking for   : G:\RetroBat\saves\fbneo\mslug.state.auto
RASTATE       : container detected, MEM block = NNNN bytes
RESULT        : state loaded OK
```

Common `RESULT` lines:
- `config file not found` → `.cfg` missing / misnamed / not beside the DLL.
- `disabled in config (enabled=0)` → set `enabled = 1`.
- `state file not found or unreadable` → the built path doesn't exist; compare
  the `looking for` line against your actual file.
- `size mismatch` warning → wrong ROM, or Save State Compression is ON.

---

## 7. Other platforms (Linux / macOS / Android)

The feature is Windows-only as written (`#ifdef _WIN32`, using
`GetModuleHandleExA`/`GetModuleFileNameA` to locate the core's own folder).

- **Android RetroArch** loads ARM `.so` cores built with the NDK; a Windows
  `.dll` will not work there, and the `#ifdef _WIN32` code compiles out anyway.
- To support Linux/macOS/Android, swap the "find my own directory" helper for a
  POSIX version using `dladdr()`, wrap it in the matching `#else`, and
  cross-build for the target. That's a separate task.
