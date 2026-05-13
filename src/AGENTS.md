# src/ — Core Source Code

Single-compilation-unit C11 file manager. Everything compiles from `nnn.c` (10,798 lines, ~230 static functions).

## File Map

| File | Lines | Role |
|---|---|---|
| `nnn.c` | 10,798 | Entire file manager. Single translation unit. |
| `nnn.h` | 293 | Key bindings: `enum action` + `struct key bindings[]` table |
| `dbg.h` | 91 | Debug macros (`DPRINTF_*`). Active only with `O_DEBUG=1`. |
| `qsort.h` | 186 | Tourbin optimized QSORT. Active only with `O_QSORT=1`. |
| `icons.h` | 450 | Icon lookup infrastructure (extension → icon string). |
| `icons-in-terminal.h` | 3,736 | Pre-generated icon glyph table for icons-in-terminal system. |
| `icons-hash.c` | 292 | Golden-ratio hash for icon lookups. Generates hash table when `ICONS_GENERATE` defined. |

## nnn.c Structure by Line Range

### Includes & Macros (1–910)
- Platform detection: `LINUX_INOTIFY`, `BSD_KQUEUE`, `HAIKU_NM`
- Optional libs: readline (`NORL`), PCRE2 (`PCRE2`), icons (`ICONS_IN_TERM`/`NERD`/`EMOJI`)
- Key constants: `ENTRY_INCR=64`, `CTX_MAX=8`, `SCROLLOFF=3`, `ONSCREEN=xlines-4`
- Spawn flags: `F_NONE`..`F_WINDOW` (line 266–278) — control `spawn()` behavior

### Data Structures (312–434)
| Struct | Line | Purpose |
|---|---|---|
| `entry` | 312 | Directory entry: name, time, size, blocks, nlen, flags, uid/gid. Bitfields for compact storage. |
| `settings` | 353 | Per-context config bitfield: sort order, hidden files, preview, filter mode, etc. |
| `runstate` | 386 | Transient program state: picker mode, selection mode, plugin state, etc. |
| `context` | 417 | Workspace: path, last dir, cursor name, filter, settings, color. Array of 8 (`g_ctx[CTX_MAX]`). |

Entry flags (line 254): `DIR_OR_DIRLNK`, `HARD_LINK`, `SYM_ORPHAN`, `FILE_MISSING`, `FILE_SELECTED`, `FILE_SCANNED`, `FILE_YOUNG`.

### Globals (436–860)
- `cfg` (settings) — global defaults, `.ctxactive=1`, `.autoenter=1`, `.timetype=2` (mtime), `.rollover=1`
- `g_ctx[CTX_MAX]` — 8 context workspaces
- `pdents` — pointer to directory entry array
- `ndents`, `cur`, `curscroll` — entry count, cursor position, scroll offset
- `pselbuf`, `selbufpos`, `selbuflen` — selection buffer and state
- `selpath` — path to selection file
- `pluginstr`, `bmstr`, `orderstr` — parsed plugin/bookmark/order kv arrays
- Threading globals (511–560): DU thread pool, mutexes, condition vars, task queue
- Event system (865–883): inotify (Linux) / kqueue (BSD/macOS) / BNodeMonitor (Haiku)

### Static Lookup Tables (611–860)
- `utils[]` — external tool names for availability checks
- `messages[]` — user-facing status strings indexed by enum
- `env_cfg[]` — `NNN_*` env var names for config parsing
- `envs[]` — env vars exported to plugins
- `archive_cmd[]` — archive commands (atool, bsdtar, zip, tar)
- `gcolors[]` — default color scheme string

### Function Groups

**Signal handlers** (933–945): `sigint_handler` sets `g_state.interrupt`, `clean_exit_sighandler` calls `exit()` (triggers `atexit` → `cleanup()`).

**Utility functions** (947–1397):
- `xitoa()` — fast int-to-string
- `xstrsncpy()`, `xstrdup()`, `is_suffix()`, `is_prefix()` — string ops
- `mkpath()` — join dir + name into path buffer
- `abspath()` — resolve to absolute path
- `shell_escape()` — escape for shell execution
- `xdirname()`, `xbasename()`, `xextension()` — path decomposition
- `getpwname()`, `getgrname()` — uid/gid to name

**Selection management** (1781–2383):
- `writesel()` — write selection to file
- `addtoselbuf()` / `rmfromselbuf()` — add/remove entries from selection buffer
- `clearselection()` — free selection buffer
- `editselection()` — open selection in editor for manual editing
- `invertselbuf()` — invert selection in current dir
- `scanselforpath()` — check if path is in selection
- `startselection()` / `endselection()` — begin/end selection mode

**Curses/UI init** (2384–2562):
- `initcurses()` — ncurses setup, color pairs, mouse, key definitions
- `init_fcolors()` — file-type color initialization
- `parseargs()` — parse CLI args into commands

**Process management** (2563–2736):
- `xfork()` — fork with flag-based behavior (piped stdout, detached, etc.)
- `join()` — wait for child, handle exit codes
- `spawn()` — full process spawn: fork → exec → join. Accepts up to 3 args + flags.
- `plugscript()` — execute plugin script

**File operations** (2737–3063):
- `cpmvrm_selection()` — copy/move/trash/delete selected files
- `cpmv_rename()` — single-file copy/move with rename
- `batch_rename()` — rename multiple files via editor (disabled with `NOBATCH`)
- `archive_selection()` — create archive from selection
- `write_lastdir()` — write last dir for cd-on-quit

**Bookmark/Session system** (5075–5250):
- `save_session()` / `load_session()` — persist/restore context states
- `set_smart_ctx()` — intelligently assign context
- `handle_bookmark()` / `add_bookmark()` — bookmark navigation/storage

**Plugin system** (6383–6729):
- `setexports()` — export env vars for plugins
- `plctrl_init()` — initialize plugin control (parse kv pairs)
- `run_plugin()` — load and execute a plugin, handle its output
- `run_cmd_as_plugin()` — treat a command string as a plugin
- `readpipe()` — read from `NNN_PIPE` (plugin→nnn communication)
- `prompt_run()` — run command from prompt

**Disk usage (DU) engine** (6779–7084):
- `du_queue_task()` — enqueue a directory for DU calculation
- `du_walk_dir()` — recursive directory walk counting blocks/files
- `du_worker_loop()` — pthread worker: takes tasks from queue, walks dirs
- `dirwalk()` — top-level DU orchestrator, manages thread pool
- `prep_threads()` — allocate thread pool and task queue
- Hardlink dedup via bitmap hash (`test_set_bit()`, `ihashbmp`)

**Directory listing** (7089–7404):
- `dentfill()` — read directory entries via `fts_open`, stat each, apply filters, sort. This is the main directory loading function.
- `populate()` — wrapper: `dentfill()` → `dentfree()` → `redraw()`
- `dentfree()` — free directory entry memory

**UI rendering** (7457–8400):
- `move_cursor()` — move cursor, handle scroll offset
- `statusbar()` — draw bottom status bar (path, sort, filter, selection count)
- `draw_line()` — render a single file entry line
- `redraw()` — full screen redraw: clear, draw entries, statusbar, cursor
- `preview_pane()` — handle file preview in side pane
- `adjust_cols()` — calculate column widths

**Main browse loop** (8414–9792):
- `browse()` — THE central event loop. ~1,380 lines. Switch on `nextsel()` → dispatches to all subsystems. This is where key actions are handled.
- Contains inline handling for: navigation, selection, file ops, filtering, sorting, contexts, plugins, sessions, help, lock screen

**Startup/teardown** (9793–end):
- `make_tmp_tree()` — create temporary directory tree
- `load_input()` — load file for picker mode
- `check_key_collision()` — verify no duplicate key bindings
- `usage()` — CLI help text
- `setup_config()` — parse env vars, init paths, setup bookmarks/plugins/orders
- `set_tmp_path()` — create temp dir for nnn runtime files
- `cleanup()` — atexit handler: free memory, restore terminal, remove FIFO
- `main()` (10232) — parse args → `setup_config()` → `initcurses()` → `browse()` → exit

## Key Patterns

- **Entry array**: `pdents` (array of `struct entry`), grown in chunks of `ENTRY_INCR`. Index `cur` = selected entry, `ndents` = total visible entries.
- **Context switching**: `g_ctx[0..7]` stores per-workspace state. `cfg.curctx` = active context.
- **Selection buffer**: `pselbuf` holds null-separated file paths. Written to `selpath` file for external access.
- **Plugin communication**: Plugin writes `{context}c{path}` to `NNN_PIPE` to request nnn cd. nnn reads via `readpipe()`.
- **DU threading**: Directory tasks queued, worker threads pick them up. Uses `pthread_cond` signaling.
- **File monitoring**: inotify/kqueue watches current dir. On change, sets `watch=TRUE` → `populate()` in browse loop.
- **Forward declarations** (918–929): Only 7 functions need forward decls due to mutual recursion.

## Anti-patterns & Pitfalls

- Do not split `nnn.c` into multiple compilation units. The project explicitly uses single-file architecture.
- `pdents` entries use bitfields and are packed — do not add fields without checking alignment/size impact.
- `browse()` is a 1,380-line switch — edits here require understanding the full loop flow.
- Selection buffer uses raw memory with manual bookkeeping (`selbufpos`, `selbuflen`) — no null-termination of the buffer itself.
- `spawn()` flags are bitmask-combined; `F_CLI = F_NORMAL | F_MULTI`, `F_SILENT = F_CLI | F_NOTRACE`.
- Color pairs are pre-allocated in `initcurses()` — index defined by `gcolors[]` string and `fcolors[]` array.
- When adding compile-time flags, update `Makefile` (flag → `CPPFLAGS`) and add to `setup_config()` if runtime-configurable.
- `xstrsncpy()` always null-terminates and returns `strlen(src)` — different from `strlcpy` semantics.

## Adding New Actions

1. Add enum value to `enum action` in `nnn.h` (before `SEL_MAX`)
2. Add key binding to `bindings[]` in `nnn.h`
3. Add case in `browse()` switch statement in `nnn.c`
4. If configurable via env: add to `env_cfg[]`, parse in `setup_config()`
