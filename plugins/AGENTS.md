# plugins/ — Plugin Ecosystem

61 user-facing plugins + 6 internal helpers. All are executable scripts communicating with nnn via environment variables and pipes.

## Plugin Contract

### Environment Variables (set by nnn before execution)

| Variable | Purpose |
|---|---|
| `NNN_PIPE` | Write pipe. Plugin writes commands to request nnn actions. **Always set.** |
| `NNN_SEL` | Path to selection file (null-separated file paths). Default: `$XDG_CONFIG_HOME/nnn/.selection` |
| `NNN_FIFO` | Path to FIFO for preview notifications. Only set when preview enabled. |
| `NNN_ARCHMNT` | Mount point for archives |
| `NNN_OPENER` | File opener command |
| `NNN_BATTHEME` | bat theme name |
| `NNN_FCOLORS` | File-type color codes |
| `NNN_MTYPE` | MIME type of hovered file |
| `NNN_HELP` | Show help flag |
| `NNN_LVL` | Nesting level (0 = top level) |
| `NNN_ORDER` | Sort order flag |
| `NNN_SCRIPT` | Script to run on a key |
| `NNN_TRASH` | Trash command |
| `NNN_HIDDEN` | Show hidden files flag |
| `NNN_IDLE` | Idle timeout seconds |
| `NNN_LOCKER` | Lock command |
| `NNN_PISTOL` | Use pistol for preview |

Additional exported from `setexports()` in nnn.c: `CURDIR`, `NNN轻量`, `NNN_SSHFS`, `NNN_RCLONE`, `NNN_CMPRS`, plus standard `EDITOR`, `PAGER`, `SHELL`, `TERM`.

### Pipe Protocol

Write to `$NNN_PIPE` to request nnn actions:
- `{context}c{path}` — cd to path in context (0 = current context). E.g., `printf "0c/tmp" > "$NNN_PIPE"`
- Plugin output on stdout is shown as messages in nnn

### Helper Library (`.nnn-plugin-helper`)

Source at the top of any plugin to use:
```sh
. "$(dirname "$0")"/.nnn-plugin-helper
```

Functions provided:
- `nnn_cd(dir, [context])` — request nnn change directory via pipe
- `cmd_exists(cmd)` — returns 0 if command available, non-zero otherwise
- `nnn_use_selection()` — returns 0 if user chose selection, 1 if current dir

Sets `$selection` to `$NNN_SEL` path.

## Plugin Categories

### File Openers & Previewers
| Plugin | Lines | Description |
|---|---|---|
| `nuke` | 557 | Primary opener. Routes files by extension → MIME → fallback. Set `GUI=1` for GUI apps. |
| `preview-tui` | 553 | tmux/kitty/wezterm/zellij/standalone previewer. Most complex plugin. |
| `tnp` | — | Open files in a Tmux Neovim pane |
| `preview-tabbed` | — | Preview using tabbed/xembed for GUI apps |

### Fuzzy Finding (fzf-based)
| Plugin | Description |
|---|---|
| `fzopen` | Open file via fzf |
| `fzcd` | Cd to directory via fzf |
| `fzhist` | Search dir history via fzf |
| `fzplug` | Fuzzy-select and run another plugin |

### File Operations
| Plugin | Description |
|---|---|
| `renamer` | Batch rename via qmv/vidir |
| `diffs` | Diff selected files via vimdiff |
| `rsynccp` | Copy with rsync + progress |
| `splitjoin` | Split/join files |
| `chksum` | Create/verify checksums |
| `fixname` | Clean filenames (bash) |
| `bulknew` | Create multiple files at once |
| `openall` | Open all selected files |
| `togglex` | Toggle executable mode |

### GPG
| Plugin | Description |
|---|---|
| `gpge` | Encrypt file |
| `gpgs` | Sign file |
| `gpgv` | Verify signature |
| `gpgd` | Decrypt file |

### Media
| Plugin | Description |
|---|---|
| `imgview` | Image viewer integration |
| `imgresize` | Resize images |
| `imgur` | Upload to imgur |
| `mp3conv` | Extract audio via ffmpeg |
| `ringtone` | Create ringtone via ffmpeg |
| `mocq` / `cmusq` | Music player (mocp/cmus) queue |
| `moclyrics` | Lyrics for current track |

### Navigation & Search
| Plugin | Description |
|---|---|
| `autojump` | Navigate via jump/autojump/zoxide |
| `gitroot` | Cd to git repo root |
| `cdpath` | CDPATH navigation |
| `finder` | Custom find command |
| `dups` | Find duplicate files |
| `oldbigfile` | Find large/old files |
| `mimelist` | List files by MIME type |

### Mount & Network
| Plugin | Description |
|---|---|
| `nmount` | Mount devices interactively |
| `mtpmount` | Mount MTP devices |
| `umounttree` | Unmount recursively |
| `upload` | Upload to Firefox Send/ix.io/file.io |
| `kdeconnect` / `gsconnect` | Send files to phone |

### System
| Plugin | Description |
|---|---|
| `pskill` | Kill processes via fzf |
| `launch` | GUI app launcher |
| `suedit` | Edit file with sudo |
| `nbak` | Backup nnn config |
| `getplugs` | Update plugins to match nnn version |
| `wallpaper` | Set wallpaper/colors |

### Reading
| Plugin | Description |
|---|---|
| `pdfread` | Read PDF aloud |
| `gutenread` | Browse/read Project Gutenberg |

### Clipboard (macOS)
| Plugin | Description |
|---|---|
| `cbcopy-mac` | Copy to macOS clipboard (AppleScript) |
| `cbpaste-mac` | Paste from macOS clipboard (AppleScript) |

### Internal (dot-prefixed, not user-facing)
| File | Purpose |
|---|---|
| `.nnn-plugin-helper` | Shared helper: `nnn_cd()`, `cmd_exists()`, selection utilities |
| `.iconlookup` | Icon lookup for directory previews |
| `.npreview` | Internal preview support |
| `.cbcp` | Internal clipboard copy |
| .`nmv` | Internal move helper |
| `.ntfy` | Internal notification |

## Writing a New Plugin

1. Create executable script in `plugins/`. POSIX sh preferred, bash acceptable.
2. Source the helper: `. "$(dirname "$0")"/.nnn-plugin-helper`
3. Read `$NNN_SEL` for selection, `$NNN_FIFO` for preview integration
4. Write to `$NNN_PIPE` for nnn cd: `printf "0c%s\n" "$target_dir" > "$NNN_PIPE"`
5. Output to stdout = shown as nnn status message
6. Exit 0 on success, non-zero on failure
7. Header convention: first comment block has `Description:`, `Shell:`, `Author:`, `Dependencies:` lines

## Pitfalls

- Plugins run in a child process. They cannot modify nnn's internal state directly — use the pipe protocol.
- `$NNN_PIPE` may not exist if FIFO is disabled (`O_NOFIFO=1`). Always check.
- Selection file uses null-separated paths, not newline-separated.
- Shell compatibility: prefer POSIX sh. If using bash, note it in header (`Shell: bash`).
- `nuke` and `preview-tui` are the most complex plugins (~550 lines each). Study them as reference.
- Plugin key is `;` in nnn. Plugin can also be run as command via `=` (launcher) or prompt `]`.
