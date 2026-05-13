# misc/ — Tests, Cross-platform Support, Auxiliary Tools

## Directory Map

| Path | Purpose |
|---|---|
| `test/` | Test harness and benchmarking |
| `auto-completion/` | Shell completions for bash, fish, zsh |
| `haiku/` | Haiku OS port (C++ node monitor) |
| `macos-legacy/` | macOS <10.12 `clock_gettime` polyfill |
| `quitcd/` | Shell wrapper functions for cd-on-quit |
| `natool/` | Python3 wrapper for patool (archive operations) |
| `musl/` | Script to build musl-linked static binary |
| `packagecore/` | Package build configs (CentOS, Debian, Fedora, Ubuntu) |
| `desktop/` | XDG `.desktop` file for GUI integration |
| `logo/` | Logo assets (SVG, PNG) |

## Testing

No unit test framework. Testing uses scripted PTY interaction and manual verification scripts.

### Test Runner: `test/nnn_runner.sh` (426 lines)

Runs nnn inside a PTY via `script(1)`, feeds keystrokes from an input file, captures output for assertion checking.

**Usage:** `./misc/test/nnn_runner.sh [OPTIONS] <input-file> [nnn-args]`

Key options: `-e <binary>`, `-d <workdir>`, `-o <output-dir>`, `-t <delay_ms>`, `-v` (verbose)

**Input file format:**
- One keystroke per line. Blank lines and `#` comments are skipped.
- Key encoding: `Enter`, `Escape`, `Tab`, `Space`, `Up/Down/Left/Right`, `BSpace`, `C-<x>` (Ctrl+x), or literal characters
- Directives: `@sleep <ms>`, `@snapshot <label>`, `@assert <string>`, `@refute <string>`, `@mark`

**Exit codes:** 0 = pass, 1 = usage error, 2 = assertion failure

### Other Test Scripts

| Script | Purpose |
|---|---|
| `genfiles.sh` | Generate 100,000 temp files for stress testing |
| `mktest.sh` | Create test fixtures (symlinks, fifos, broken links) |
| `benchmark.sh` | Run timing benchmarks with configurable sample count |
| `verify-du.sh` | Verify nnn disk usage against system `du` |
| `plot-bench.py` | Matplotlib violin plots of benchmark data |
| `test-76ee5803.sh` | Regression test for shell-escape quoting fix |

### Test Workflow

1. `mktest.sh` — create fixture directory with various file types
2. `nnn_runner.sh` — drive nnn through scripted interactions, capture snapshots, assert output
3. `benchmark.sh` — time nnn operations over multiple runs
4. `verify-du.sh` — compare DU calculations against system `du`

## Cross-Platform Support

### Haiku (`haiku/`)
- `nm.cpp` (C++): BNodeMonitor wrapper for file system watching
- `haiku_interop.h`: Interface header included by `nnn.c` when `__HAIKU__` defined
- `Makefile`: Haiku-specific build (245 lines), compiles both C and C++ sources

### macOS Legacy (`macos-legacy/`)
- `mach_gettime.c/h`: `clock_gettime`/`CLOCK_REALTIME`/`CLOCK_MONOTONIC` polyfill using `mach_absolute_time`
- Auto-included by `Makefile` when macOS < 10.12 detected
- Uses `MACOS_BELOW_1012` preprocessor flag

## Shell Completions (`auto-completion/`)

| Shell | File |
|---|---|
| Bash | `bash/nnn-completion.bash` |
| Fish | `fish/nnn.fish` |
| Zsh | `zsh/_nnn` |

## cd-on-quit (`quitcd/`)

Shell functions that wrap nnn so the parent shell's working directory changes to nnn's last directory on exit.

| Shell | File |
|---|---|
| Bash/Zsh/Sh | `quitcd.bash_sh_zsh` — defines `n()` function |
| Fish | `quitcd.fish` |
| Csh | `quitcd.csh` |
| Elvish | `quitcd.elv` |
| Nushell | `quitcd.nu` |
| Rc | `quitcd.rc` |

Works by writing last dir to a temp file on quit, then the wrapper function `cd`s to that path.

## Pitfalls

- Tests require `script` command (util-linux or POSIX) and `mkfifo`.
- `nnn_runner.sh` delay between keystrokes defaults to 200ms — may need tuning for slow systems.
- Haiku build requires both C and C++ compiler.
- Shell completions must be updated when CLI flags change.
