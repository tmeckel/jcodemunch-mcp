# Cross-Platform Path Remapping — Design Spec

**Date:** 2026-03-23
**Feature:** `JCODEMUNCH_PATH_MAP` environment variable
**Branch:** `feat/cross-platform-path-remap`

---

## Motivation

jCodemunch derives each local repo's identity from a SHA-1 hash of the absolute folder
path (`_local_repo_name`). When a user builds an index on Linux
(`/home/ridge/Nextcloud/Dev/myproject`) and syncs `~/.code-index/` to Windows
(`D:\Nextcloud\Dev\myproject`), two problems arise:

1. **Broken repo lookup** — the hash of the Windows path differs from the stored hash,
   so `index_folder` starts a fresh index instead of reusing the existing one.
2. **Wrong `source_root` display** — `list_repos` returns the original Linux path,
   which is meaningless (and potentially broken) on Windows.

`JCODEMUNCH_PATH_MAP` solves both without requiring a re-index.

---

## Environment Variable

```
JCODEMUNCH_PATH_MAP=<orig1>=<new1>,<orig2>=<new2>,...
```

- **`orig`** — the path prefix as stored in the index (the machine that built it).
- **`new`** — the corresponding prefix on the current machine.
- Multiple mappings are comma-separated.
- The `=` separator splits on the **first `=` only** (`split("=", 1)`) so POSIX paths
  containing `=` are handled correctly.
- Separators are normalised to `/` for all internal comparisons. Output uses the OS
  native separator (`os.sep`).
- Malformed entries (no `=`, empty `orig`, empty `new`) are skipped with a `WARNING`
  log; the rest of the list still applies.

### Example

```bash
# Linux index reused on Windows
JCODEMUNCH_PATH_MAP=/home/ridge/Nextcloud=D:\Nextcloud,/home/ridge/work=C:\work
```

---

## Architecture

### New module: `src/jcodemunch_mcp/path_map.py`

Single source of truth for parsing and applying the mapping.

```python
ENV_VAR = "JCODEMUNCH_PATH_MAP"

def parse_path_map() -> list[tuple[str, str]]:
    """Parse JCODEMUNCH_PATH_MAP into (original, replacement) pairs.

    Returns [] when the env var is unset or empty.
    Malformed entries are skipped with WARNING log.
    """

def remap(path: str, pairs: list[tuple[str, str]], reverse: bool = False) -> str:
    """Apply path prefix substitution with separator normalisation.

    Forward (reverse=False): orig → new   (stored index → current machine)
    Reverse (reverse=True):  new  → orig  (current machine → stored index)

    Tries pairs in order; returns on first match.
    Always outputs using os.sep.

    Note: remap() is not a pure no-op when pairs is empty — it still
    normalises separators to os.sep. Callers that compare the return value
    to the original input must account for this.
    """

def remap_symbol_paths(symbol: dict, pairs: list[tuple[str, str]]) -> dict:
    """Return a copy of symbol with all path fields forward-remapped.

    Remaps the 'file' key (and 'source_root' if present).
    Safe to call with an empty pairs list (normalises separators only).
    Used by get_symbol, search_symbols, and other tools that return
    symbol dicts directly.
    """
```

### Separator normalisation detail

All prefix comparisons are done after replacing `\` with `/` on both sides.
Output is reconstructed by replacing `/` with `os.sep`, so Windows users see
`D:\Nextcloud\Dev\foo` and POSIX users see `/home/ridge/Nextcloud/Dev/foo`.

---

## Touch Points

### Rule

> **Remap whenever a path is read from the stored index and returned to the caller.
> Do not remap paths that come from user input.**

| File | Location | Direction | Reason |
|---|---|---|---|
| `tools/index_folder.py:446` | watcher fast path, before `_local_repo_name` | reverse | match stored hash |
| `tools/index_folder.py:698` | standard path, before `_local_repo_name` | reverse | match stored hash |
| `storage/sqlite_store.py:853` | `_list_repo_from_db`, `source_root` field | forward | from stored index |
| `storage/index_store.py:641` | `_repo_entry_from_data`, `source_root` field (legacy JSON) | forward | from stored index |
| `server.py:~1432` | `_run_config` Core section | — | config display |

**Not remapped (confirmed):**

- `index_folder` return value `_folder_display` — derived from user-supplied `folder_path`, already correct for the current machine.
- `find_references`, `find_importers`, `check_references` — import graph keys are relative paths (e.g. `src/main.py`), not absolute.
- `get_repo_outline` — does not include `source_root` in its response payload.
- `_index_to_dict` — internal serialisation helper, never returned to an MCP caller.

### Remap call pattern (forward, e.g. `_list_repo_from_db`)

```python
from ..path_map import parse_path_map, remap

pairs = parse_path_map()
"source_root": remap(meta.get("source_root", ""), pairs),
```

### Remap call pattern (reverse, e.g. `index_folder`)

```python
from .path_map import parse_path_map, remap

pairs = parse_path_map()
lookup_path = Path(remap(str(folder_path), pairs, reverse=True))
repo_name = _local_repo_name(lookup_path)
# folder_path (unchanged) is still used for the actual file walk
```

---

## Config Display (`_run_config`)

Added to the **Core** section, importing `ENV_VAR` from `path_map.py`:

```
JCODEMUNCH_PATH_MAP          (none)     ← default when unset
JCODEMUNCH_PATH_MAP          /home/ridge/Nextcloud=D:\Nextcloud
```

`ENV_VAR` is imported as a constant so the string is never duplicated.

---

## Tests

New file: `tests/test_path_map.py`

Modelled on `tests/test_extra_extensions.py` (monkeypatch + autouse fixture style).

### `parse_path_map` cases
- Unset env var → empty list
- Whitespace-only → empty list
- Single valid pair
- Multiple valid pairs (comma-separated)
- Path with `=` in it (split on first `=` only)
- Malformed entry (no `=`) mixed with valid entries → valid entries apply, warning logged
- Empty `orig` or empty `new` → entry skipped, warning logged

### `remap` cases
- No pairs, POSIX path → returned with POSIX separators
- No pairs, Windows path → separators normalised to `os.sep`
- Forward remap: Linux prefix replaced with Windows prefix
- Reverse remap: Windows prefix replaced back to Linux prefix
- No matching prefix → path returned unchanged (but separator-normalised)
- First matching pair wins (pair order matters)
- Mixed separators in input (`D:/Nextcloud/Dev`) match prefix `D:\Nextcloud`

### `remap_symbol_paths` cases
- Symbol with only `file` key → `file` remapped
- Symbol with `file` and `source_root` → both remapped
- Empty pairs → separators normalised, values otherwise unchanged

### Integration: `list_repos` with remap
- Index a temp folder, set `JCODEMUNCH_PATH_MAP` to remap that path, call `list_repos`,
  verify `source_root` in the response contains the remapped prefix.

### Integration: `index_folder` reverse lookup
- Index a temp folder, set `JCODEMUNCH_PATH_MAP` to remap the path to a fake "other"
  prefix, call `index_folder` with the fake prefix, verify it detects "no changes"
  (i.e., found the existing index via the reverse remap).

---

## Error Handling

| Situation | Behaviour |
|---|---|
| `JCODEMUNCH_PATH_MAP` unset | `parse_path_map()` returns `[]`; `remap()` is separator-normalise only |
| Malformed entry (no `=`) | Skip entry, emit `logging.WARNING` |
| Empty `orig` or `new` after split | Skip entry, emit `logging.WARNING` |
| Path does not match any pair | Return path with `os.sep` normalisation, no error |

---

## Files Changed

| File | Change |
|---|---|
| `src/jcodemunch_mcp/path_map.py` | **new** — `ENV_VAR`, `parse_path_map`, `remap`, `remap_symbol_paths` |
| `src/jcodemunch_mcp/tools/index_folder.py` | reverse remap at lines 446 and 698 |
| `src/jcodemunch_mcp/storage/sqlite_store.py` | forward remap at line 853 |
| `src/jcodemunch_mcp/storage/index_store.py` | forward remap at line 641 |
| `src/jcodemunch_mcp/server.py` | add `JCODEMUNCH_PATH_MAP` to `_run_config` Core section |
| `tests/test_path_map.py` | **new** — unit + integration tests |
