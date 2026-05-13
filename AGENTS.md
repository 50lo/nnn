# AGENTS.md — nnn (n³) Terminal File Manager

## What

nnn is a monolithic C11 terminal file manager (~10,800 lines in a single source file) with a plugin-based extension model. Version 5.3 (In progress). BSD 2-Clause license.

## Architecture

```
src/nnn.c          →  Entire file manager (single compilation unit)
src/nnn.h          →  Key bindings (enum action + bindings[] table)
src/dbg.h          →  Debug logging (O_DEBUG=1 only)
plugins/*          →  61 executable scripts (sh/bash/python) + 6 internal helpers
patches/*          →  4 optional compile-time patches applied/reversed around build
misc/test/         →  PTY-based test runner + benchmarks (no unit test framework)
misc/quitcd/       →  Shell wrappers for cd-on-quit
misc/haiku/        →  Haiku OS port (C++ node monitor)
misc/macos-legacy/ →  macOS <10.12 clock_gettime polyfill
```

## Build System

**Compiler:** C11, `-O3` by default. **Binary:** `nnn`.

```sh
make                                    # default build (no readline, O_NORL=1)
make O_NORL=0                           # with readline support
make O_PCRE2=1 O_NERD=1                # with PCRE2 + Nerd Font icons
make O_GITSTATUS=1 O_NAMEFIRST=1       # with patches
make debug                              # debug build (O_DEBUG=1)
make static                             # static binary
make checkpatches                       # test all 16 patch combinations
make clean
```

**Key compile-time flags** (set in `Makefile`):

| Flag | Default | Effect |
|---|---|---|
| `O_NORL` | 1 | Disable readline (`-DNORL`) |
| `O_PCRE2` | 0 | Use PCRE2 instead of POSIX regex |
| `O_ICONS` / `O_NERD` / `O_EMOJI` | 0 | Icon support (mutually exclusive) |
| `O_NOBATCH` | 0 | Disable built-in batch renamer |
| `O_NOFIFO` | 0 | Disable FIFO previewer |
| `O_NOSSN` | 0 | Disable session support |
| `O_NOMOUSE` | 0 | Disable mouse support |
| `O_QSORT` | 0 | Use Tourbin optimized QSORT |
| `O_DEBUG` | 0 | Enable debug logging to `/tmp/nnndbg` |
| `O_BENCH` | 0 | Benchmark mode (exit on first input) |
| `O_DIMFILTERED` | 1 | Dim characters matching filter |

**Linked libraries:** ncursesw (or ncurses), pthread. Optional: readline, pcre2.

## Navigation Map

- **[`src/AGENTS.md`](src/AGENTS.md)** — Core source code: data structures, function map by line range, patterns, entry point, how to add new actions
- **[`plugins/AGENTS.md`](plugins/AGENTS.md)** — Plugin contract (env vars, pipe protocol, helper library), all 61 plugins categorized, how to write new plugins
- **[`patches/AGENTS.md`](patches/AGENTS.md)** — Patch framework: current patches, apply/resolve workflow, how to add patches
- **[`misc/AGENTS.md`](misc/AGENTS.md)** — Test runner, cross-platform support (Haiku, macOS legacy), shell completions, cd-on-quit wrappers

## Key Conventions

- **Single-file architecture.** All logic in `src/nnn.c`. Do not split into multiple compilation units.
- **No dynamic allocation without `xrealloc`.** Entry array grown in chunks. Manual memory management throughout.
- **8 contexts.** `g_ctx[0..7]`, each with independent path, filter, sort, cursor.
- **Selection is a flat buffer.** `pselbuf` holds null-separated paths. No structured list.
- **Plugins communicate via env vars and pipes.** Write `{ctx}c{path}` to `$NNN_PIPE` to request cd.
- **File monitoring is platform-specific.** inotify (Linux), kqueue (BSD/macOS), BNodeMonitor (Haiku).
- **No unit tests.** PTY-based integration testing via `misc/test/nnn_runner.sh`.

## CI

- **GitHub Actions** (`.github/workflows/`): macOS gcc, macOS clang+clang-tidy, Ubuntu patches+cppcheck
- **CircleCI** (`.circleci/`): gcc 9–14, clang 14–18, clang-tidy-18, shellcheck. Nightly Saturday builds. Tag-triggered releases.
- **CodeQL**: Weekly C++ and Python security analysis.

## Quick Reference for Common Tasks

| Task | Where to Look |
|---|---|
| Add a new keybinding | `src/nnn.h` (enum + bindings[]) → `src/nnn.c` browse() switch |
| Add a compile-time option | `Makefile` flag → `src/nnn.c` `#ifdef` → `setup_config()` |
| Add a new plugin | `plugins/<name>` — see `plugins/AGENTS.md` |
| Fix a file operation bug | `src/nnn.c` file ops section (lines 2737–3063) |
| Fix a UI rendering bug | `src/nnn.c` redraw/statusbar/draw_line (lines 7457–8400) |
| Fix DU calculation | `src/nnn.c` DU engine (lines 6779–7084) |
| Fix plugin communication | `src/nnn.c` plugin section (lines 6383–6729) + `plugins/.nnn-plugin-helper` |
| Add a test | `misc/test/` — write input file for `nnn_runner.sh` |
| Update shell completions | `misc/auto-completion/` (bash, fish, zsh) |
