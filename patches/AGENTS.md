# patches/ — User Patch Framework

Optional diff patches applied/reversed automatically around compilation. Each patch modifies `src/nnn.c` at build time without permanent source changes.

## Current Patches

| Patch | Make Variable | Description |
|---|---|---|
| `colemak/` | `O_COLEMAK=1` | Remaps key bindings for Colemak keyboard layout |
| `gitstatus/` | `O_GITSTATUS=1` | Adds git status column to detail view. Adds `-G` CLI flag. |
| `namefirst/` | `O_NAMEFIRST=1` | Prints filename first in detail view. Adds uid/gid columns. |
| `restorepreview/` | `O_RESTOREPREVIEW=1` | Adds pipe to close/restore preview-tui for internal edits |

Each directory contains `mainline.diff` against current master. `gitstatus/` also has `namefirst.diff` for inter-patch compatibility.

## How Patches Are Applied

Build flow in `Makefile`:
1. `make prepatch` — applies enabled patches in alphabetical order via `patch -p1`
2. Compilation of `src/nnn.c`
3. `make postpatch` — reverses patches to restore clean source

Multiple patches can be combined: `make O_GITSTATUS=1 O_NAMEFIRST=1`

## Applying Patches

```sh
make O_NAMEFIRST=1                    # single patch
make O_GITSTATUS=1 O_NAMEFIRST=1      # combined
PATCH_OPTS="--merge" make O_RESTOREPREVIEW=1  # merge-mode (conflict markers)
```

## Resolving Patch Conflicts

Run `make checkpatches` to test all 2^4 = 16 combinations. Uses `check-patches.sh`.

To resolve a conflict:
1. `PATCH_OPTS="--merge" make O_<PATCH>=1` — generates conflict markers in `src/nnn.c`
2. Edit `src/nnn.c` to resolve conflicts
3. `git diff > patch.diff && sed -i -e "/^$/{r patch.diff" -e "q;}" patches/<name>/mainline.diff`

## Adding a New Patch

1. Create `patches/<name>/` directory with `mainline.diff` against `src/nnn.c`
2. Add `O_<NAME>` variable to `Makefile` (follow existing pattern)
3. Add patch path to `Makefile` variables section
4. Add to `prepatch`/`postpatch` targets
5. Add Make variable to `patches/check-patches.sh` array
6. If it conflicts with existing patches, provide compatibility diff(s)

## Pitfalls

- Patches are applied in alphabetical order. Later patches may conflict with earlier ones.
- Always run `make checkpatches` after modifying any patch.
- Patches target `src/nnn.c` line numbers which shift between versions. Patches are adapted on each release.
- Do not modify source and leave patches applied — always build clean or run `postpatch`.
