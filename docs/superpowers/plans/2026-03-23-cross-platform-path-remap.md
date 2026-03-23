# Cross-Platform Path Remapping Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `JCODEMUNCH_PATH_MAP` env var so an index built on one machine (e.g. Linux `/home/ridge/Nextcloud`) can be reused on another (e.g. Windows `D:\Nextcloud`) without re-indexing.

**Architecture:** A new `path_map.py` module parses `JCODEMUNCH_PATH_MAP` and provides `parse_path_map()` / `remap()`. Reverse remap is applied before repo-name hash derivation in `index_folder.py` and `watcher.py` (so the existing index is found). Forward remap is applied to `source_root` when assembling `list_repos` responses in both storage backends.

**Tech Stack:** Python stdlib only (`os`, `pathlib`). pytest + monkeypatch for tests. No new dependencies.

**Spec:** `docs/superpowers/specs/2026-03-23-cross-platform-path-remap-design.md`

---

## File Map

| File | Status | Responsibility |
|---|---|---|
| `src/jcodemunch_mcp/path_map.py` | **create** | `ENV_VAR`, `parse_path_map`, `remap` |
| `tests/test_path_map.py` | **create** | all unit + integration tests |
| `src/jcodemunch_mcp/tools/index_folder.py` | modify | reverse remap at lines 446 and 698 |
| `src/jcodemunch_mcp/watcher.py` | modify | reverse remap at lines 272 and 748 |
| `src/jcodemunch_mcp/storage/sqlite_store.py` | modify | forward remap `source_root` at line 853 |
| `src/jcodemunch_mcp/storage/index_store.py` | modify | forward remap `source_root` at line 641 |
| `src/jcodemunch_mcp/server.py` | modify | add `JCODEMUNCH_PATH_MAP` to `_run_config` Core section |
| `CLAUDE.md` | modify | add `JCODEMUNCH_PATH_MAP` to env vars table |

---

## Task 1: `parse_path_map` — parse the env var

**Files:**
- Create: `src/jcodemunch_mcp/path_map.py`
- Create: `tests/test_path_map.py`

- [ ] **Step 1.1: Write the failing tests**

Create `tests/test_path_map.py`:

```python
"""Tests for JCODEMUNCH_PATH_MAP env var parsing and path remapping."""

import logging
import pytest

from jcodemunch_mcp.path_map import parse_path_map, ENV_VAR


def test_parse_unset(monkeypatch):
    monkeypatch.delenv(ENV_VAR, raising=False)
    assert parse_path_map() == []


def test_parse_whitespace_only(monkeypatch):
    monkeypatch.setenv(ENV_VAR, "   ")
    assert parse_path_map() == []


def test_parse_single_pair(monkeypatch):
    monkeypatch.setenv(ENV_VAR, "/home/user=/mnt/user")
    assert parse_path_map() == [("/home/user", "/mnt/user")]


def test_parse_multiple_pairs(monkeypatch):
    monkeypatch.setenv(ENV_VAR, "/a=/b,/c=/d")
    assert parse_path_map() == [("/a", "/b"), ("/c", "/d")]


def test_parse_equals_in_path(monkeypatch):
    """First = is the separator; later = chars belong to the value."""
    monkeypatch.setenv(ENV_VAR, "/home/user/a=b=/new/path")
    assert parse_path_map() == [("/home/user/a=b", "/new/path")]


def test_parse_malformed_no_equals_skipped(monkeypatch, caplog):
    monkeypatch.setenv(ENV_VAR, "/valid=/ok,noequalssign,/also=/fine")
    with caplog.at_level(logging.WARNING):
        result = parse_path_map()
    assert result == [("/valid", "/ok"), ("/also", "/fine")]
    assert any("noequalssign" in r.message for r in caplog.records)


def test_parse_empty_orig_skipped(monkeypatch, caplog):
    monkeypatch.setenv(ENV_VAR, "=/new/path")
    with caplog.at_level(logging.WARNING):
        result = parse_path_map()
    assert result == []


def test_parse_empty_new_skipped(monkeypatch, caplog):
    monkeypatch.setenv(ENV_VAR, "/old/path=")
    with caplog.at_level(logging.WARNING):
        result = parse_path_map()
    assert result == []


def test_parse_whitespace_stripped(monkeypatch):
    """Leading/trailing whitespace in tokens is stripped."""
    monkeypatch.setenv(ENV_VAR, " /home/user = /mnt/user ")
    assert parse_path_map() == [("/home/user", "/mnt/user")]
```

- [ ] **Step 1.2: Run tests to verify they fail**

```bash
cd ~/Nextcloud/Dev/jcodemunch-mcp
python -m pytest tests/test_path_map.py -v 2>&1 | head -20
```

Expected: `ModuleNotFoundError: No module named 'jcodemunch_mcp.path_map'`

- [ ] **Step 1.3: Implement `parse_path_map`**

Create `src/jcodemunch_mcp/path_map.py`:

```python
"""Cross-platform path prefix remapping via JCODEMUNCH_PATH_MAP."""

import logging
import os

logger = logging.getLogger(__name__)

ENV_VAR = "JCODEMUNCH_PATH_MAP"


def parse_path_map() -> list[tuple[str, str]]:
    """Parse JCODEMUNCH_PATH_MAP into (original, replacement) pairs.

    Format: orig1=new1,orig2=new2,...
    Splits on the first '=' only so POSIX paths containing '=' work correctly.
    Returns [] when the env var is unset or empty.
    Malformed entries (no '=', empty orig, empty new) are skipped with a WARNING.
    """
    raw = os.environ.get(ENV_VAR, "").strip()
    if not raw:
        return []

    pairs: list[tuple[str, str]] = []
    for entry in raw.split(","):
        entry = entry.strip()
        if not entry:
            continue
        if "=" not in entry:
            logger.warning("JCODEMUNCH_PATH_MAP: skipping malformed entry (no '='): %r", entry)
            continue
        orig, new = entry.split("=", 1)
        orig = orig.strip()
        new = new.strip()
        if not orig:
            logger.warning("JCODEMUNCH_PATH_MAP: skipping entry with empty original prefix: %r", entry)
            continue
        if not new:
            logger.warning("JCODEMUNCH_PATH_MAP: skipping entry with empty replacement prefix: %r", entry)
            continue
        pairs.append((orig, new))
    return pairs
```

- [ ] **Step 1.4: Run tests to verify they pass**

```bash
python -m pytest tests/test_path_map.py -v -k "parse"
```

Expected: all `test_parse_*` tests PASS.

- [ ] **Step 1.5: Commit**

```bash
git add src/jcodemunch_mcp/path_map.py tests/test_path_map.py
git commit -m "feat: add path_map.py with parse_path_map (JCODEMUNCH_PATH_MAP)"
```

---

## Task 2: `remap` — apply prefix substitution with separator normalisation

**Files:**
- Modify: `src/jcodemunch_mcp/path_map.py`
- Modify: `tests/test_path_map.py`

- [ ] **Step 2.1: Write the failing tests**

Append to `tests/test_path_map.py`:

```python
import os
from jcodemunch_mcp.path_map import remap


def test_remap_empty_pairs_normalises_sep():
    """No mapping set: path returned with os.sep normalisation."""
    result = remap("/home/user/project/file.py", [])
    assert result == str(os.path.join("/home", "user", "project", "file.py"))


def test_remap_forward_replaces_prefix():
    pairs = [("/home/user", "C:\\Users\\user")]
    result = remap("/home/user/project/file.py", pairs)
    # Normalised with os.sep — on POSIX this is forward slash
    assert result.replace("\\", "/") == "C:/Users/user/project/file.py"


def test_remap_reverse_replaces_prefix():
    pairs = [("/home/user", "C:\\Users\\user")]
    result = remap("C:\\Users\\user\\project\\file.py", pairs, reverse=True)
    assert result.replace("\\", "/") == "/home/user/project/file.py"


def test_remap_no_match_returns_normalised():
    pairs = [("/home/other", "/mnt/other")]
    result = remap("/home/user/project", pairs)
    assert result.replace("\\", "/") == "/home/user/project"


def test_remap_first_pair_wins():
    pairs = [("/home/user", "/first"), ("/home/user", "/second")]
    result = remap("/home/user/file.py", pairs)
    assert result.replace("\\", "/") == "/first/file.py"


def test_remap_mixed_separators_in_input_match():
    """D:/Users/user (forward slashes) matches stored prefix D:\\Users\\user."""
    pairs = [("/home/user", "D:\\Users\\user")]
    result = remap("D:/Users/user/project", pairs, reverse=True)
    assert result.replace("\\", "/") == "/home/user/project"


def test_remap_multiple_pairs_correct_one_matches():
    pairs = [("/home/alice", "/mnt/alice"), ("/home/bob", "/mnt/bob")]
    result = remap("/home/bob/work/file.py", pairs)
    assert result.replace("\\", "/") == "/mnt/bob/work/file.py"
```

- [ ] **Step 2.2: Run tests to verify they fail**

```bash
python -m pytest tests/test_path_map.py -v -k "remap"
```

Expected: `ImportError` (remap not yet defined).

- [ ] **Step 2.3: Implement `remap`**

Append to `src/jcodemunch_mcp/path_map.py`:

```python
def remap(path: str, pairs: list[tuple[str, str]], reverse: bool = False) -> str:
    """Apply path prefix substitution with OS separator normalisation.

    Forward (reverse=False): replaces original → replacement.
                             Use when reading stored paths for display.
    Reverse (reverse=True):  replaces replacement → original.
                             Use before hashing a user-supplied path to look
                             up an index that was built on a different machine.

    Tries pairs in order; applies the first match.
    Always outputs using os.sep.

    Note: not a pure no-op when pairs is empty — separator normalisation
    still applies. Callers that compare the return value to the original
    input must account for this.
    """
    # Normalise input separators to '/' for comparison
    path_norm = path.replace("\\", "/")

    for orig, new in pairs:
        if reverse:
            src = new.replace("\\", "/")
            dst = orig.replace("\\", "/")
        else:
            src = orig.replace("\\", "/")
            dst = new.replace("\\", "/")

        # Ensure prefix comparison works at directory boundaries
        src_prefix = src.rstrip("/")
        if path_norm == src_prefix or path_norm.startswith(src_prefix + "/"):
            remainder = path_norm[len(src_prefix):]
            remapped = dst.rstrip("/") + remainder
            # Output using OS native separator
            return remapped.replace("/", os.sep)

    # No match — return with OS native separator
    return path_norm.replace("/", os.sep)
```

- [ ] **Step 2.4: Run all path_map tests**

```bash
python -m pytest tests/test_path_map.py -v
```

Expected: all tests PASS.

- [ ] **Step 2.5: Commit**

```bash
git add src/jcodemunch_mcp/path_map.py tests/test_path_map.py
git commit -m "feat: add remap() to path_map.py with separator normalisation"
```

---

## Task 3: Forward remap `source_root` in storage layer

Apply remap when `source_root` is assembled into `list_repos` responses. Two places: `sqlite_store.py` (SQLite backend) and `index_store.py` (legacy JSON backend).

**Files:**
- Modify: `src/jcodemunch_mcp/storage/sqlite_store.py` (line 853)
- Modify: `src/jcodemunch_mcp/storage/index_store.py` (line 641)
- Modify: `tests/test_path_map.py`

- [ ] **Step 3.1: Write the failing integration test**

Append to `tests/test_path_map.py`:

```python
from pathlib import Path
from jcodemunch_mcp.tools.index_folder import index_folder
from jcodemunch_mcp.tools.list_repos import list_repos


def test_list_repos_source_root_remapped(tmp_path, monkeypatch):
    """source_root in list_repos response uses the remapped prefix."""
    # Create a minimal source file so index_folder has something to index
    (tmp_path / "hello.py").write_text("def hello(): pass\n")

    # Index the folder with AI summaries disabled
    monkeypatch.setenv("JCODEMUNCH_USE_AI_SUMMARIES", "false")
    storage = str(tmp_path / ".index")
    result = index_folder(
        path=str(tmp_path),
        storage_path=storage,
        use_ai_summaries=False,
    )
    assert result.get("success"), result

    # Set up a remap: tmp_path prefix → /remapped/prefix
    orig_prefix = str(tmp_path)
    new_prefix = "/remapped/prefix"
    monkeypatch.setenv(ENV_VAR, f"{orig_prefix}={new_prefix}")

    repos = list_repos(storage_path=storage)
    assert repos["count"] == 1
    source_root = repos["repos"][0]["source_root"]
    assert source_root.replace("\\", "/").startswith("/remapped/prefix"), (
        f"Expected remapped prefix, got: {source_root}"
    )
```

- [ ] **Step 3.2: Run the test to verify it fails**

```bash
python -m pytest tests/test_path_map.py::test_list_repos_source_root_remapped -v
```

Expected: FAIL — `source_root` still shows the original `tmp_path` prefix.

- [ ] **Step 3.3: Implement forward remap in `sqlite_store.py`**

In `src/jcodemunch_mcp/storage/sqlite_store.py`, add the import near the top of the file (alongside other imports):

```python
from ..path_map import parse_path_map, remap
```

Then in `_list_repo_from_db` change line 853:

Before:
```python
            "source_root": meta.get("source_root", ""),
```

After:
```python
            "source_root": remap(meta.get("source_root", ""), parse_path_map()),
```

- [ ] **Step 3.4: Run the test to verify it passes**

```bash
python -m pytest tests/test_path_map.py::test_list_repos_source_root_remapped -v
```

Expected: PASS.

- [ ] **Step 3.5: Write a unit test for the legacy JSON path (`_repo_entry_from_data`)**

Append to `tests/test_path_map.py`:

```python
def test_repo_entry_from_data_source_root_remapped(monkeypatch):
    """_repo_entry_from_data remaps source_root (legacy JSON index path)."""
    from jcodemunch_mcp.storage.index_store import IndexStore

    monkeypatch.setenv(ENV_VAR, "/old/root=/new/root")
    store = IndexStore()

    data = {
        "repo": "local/myproject-abc12345",
        "indexed_at": "2026-01-01T00:00:00",
        "symbol_count": 10,
        "file_count": 5,
        "languages": {"python": 5},
        "index_version": 5,
        "source_root": "/old/root/myproject",
    }
    entry = store._repo_entry_from_data(data)
    assert entry is not None
    assert entry["source_root"].replace("\\", "/") == "/new/root/myproject"
```

- [ ] **Step 3.6: Run the test to verify it fails**

```bash
python -m pytest tests/test_path_map.py::test_repo_entry_from_data_source_root_remapped -v
```

Expected: FAIL — `source_root` is `/old/root/myproject`.

- [ ] **Step 3.7: Implement forward remap in `index_store.py`**

In `src/jcodemunch_mcp/storage/index_store.py`, add the import alongside existing imports at the top (`index_store.py` lives in `storage/`, so two dots are needed):

```python
from ..path_map import parse_path_map, remap
```

Then in `_repo_entry_from_data`, change lines 640–644:

Before:
```python
        if data.get("source_root"):
            if os.environ.get("JCODEMUNCH_REDACT_SOURCE_ROOT", "") == "1":
                repo_entry["source_root"] = data.get("display_name", "") or ""
            else:
                repo_entry["source_root"] = data["source_root"]
```

After:
```python
        if data.get("source_root"):
            if os.environ.get("JCODEMUNCH_REDACT_SOURCE_ROOT", "") == "1":
                repo_entry["source_root"] = data.get("display_name", "") or ""
            else:
                repo_entry["source_root"] = remap(data["source_root"], parse_path_map())
```

- [ ] **Step 3.8: Run all storage tests**

```bash
python -m pytest tests/test_path_map.py::test_repo_entry_from_data_source_root_remapped tests/test_path_map.py::test_list_repos_source_root_remapped tests/test_storage.py tests/test_sqlite_store.py -v
```

Expected: all PASS.

- [ ] **Step 3.9: Commit**

```bash
git add src/jcodemunch_mcp/storage/sqlite_store.py \
        src/jcodemunch_mcp/storage/index_store.py \
        tests/test_path_map.py
git commit -m "feat: forward-remap source_root in list_repos responses"
```

---

## Task 4: Reverse remap in `index_folder.py`

Remap the caller-supplied `folder_path` back to the stored path before computing `_local_repo_name`, so `index_folder` finds the existing index when called with a remapped path.

The `folder_path` variable is used unchanged for the actual file walk — only the hash derivation uses the remapped version.

**Files:**
- Modify: `src/jcodemunch_mcp/tools/index_folder.py` (lines 446, 698)
- Modify: `tests/test_path_map.py`

- [ ] **Step 4.1: Write the failing integration test**

Append to `tests/test_path_map.py`:

```python
def test_index_folder_reverse_remap_finds_existing_index(tmp_path, monkeypatch):
    """index_folder with a remapped path finds the existing index (no re-index)."""
    (tmp_path / "hello.py").write_text("def hello(): pass\n")
    monkeypatch.setenv("JCODEMUNCH_USE_AI_SUMMARIES", "false")
    storage = str(tmp_path / ".index")

    # Index using the real path
    result = index_folder(
        path=str(tmp_path),
        storage_path=storage,
        use_ai_summaries=False,
    )
    assert result.get("success"), result

    # Pretend the folder lives at a different prefix on the current machine
    fake_prefix = str(tmp_path.parent / "remapped_root")
    fake_path = fake_prefix + "/" + tmp_path.name

    # Tell path_map: fake_prefix → real parent prefix
    monkeypatch.setenv(ENV_VAR, f"{fake_prefix}={str(tmp_path.parent)}")

    # Call index_folder with the fake path — should detect "no changes"
    result2 = index_folder(
        path=fake_path,
        storage_path=storage,
        use_ai_summaries=False,
        incremental=True,
    )
    assert result2.get("success"), result2
    assert result2.get("message") == "No changes detected" or result2.get("changed", 0) == 0, (
        f"Expected no changes, got: {result2}"
    )
```

- [ ] **Step 4.2: Run the test to verify it fails**

```bash
python -m pytest tests/test_path_map.py::test_index_folder_reverse_remap_finds_existing_index -v
```

Expected: FAIL — `index_folder` starts a fresh index instead of finding the existing one.

- [ ] **Step 4.3: Add import to `index_folder.py`**

In `src/jcodemunch_mcp/tools/index_folder.py`, add alongside existing imports from `..security`:

```python
from ..path_map import parse_path_map, remap
```

- [ ] **Step 4.4: Patch line 446 (watcher fast path)**

Before (line 446):
```python
        if changed_paths and incremental:
            repo_name = _local_repo_name(folder_path)
            owner = "local"
```

After:
```python
        if changed_paths and incremental:
            _pairs = parse_path_map()
            repo_name = _local_repo_name(Path(remap(str(folder_path), _pairs, reverse=True)))
            owner = "local"
```

- [ ] **Step 4.5: Patch line 698 (standard path)**

Before (line 698, in the standard discovery section):
```python
        # Create repo identifier from folder path
        repo_name = _local_repo_name(folder_path)
        owner = "local"
```

After:
```python
        # Create repo identifier from folder path
        _pairs = parse_path_map()
        repo_name = _local_repo_name(Path(remap(str(folder_path), _pairs, reverse=True)))
        owner = "local"
```

- [ ] **Step 4.6: Run the integration test**

```bash
python -m pytest tests/test_path_map.py::test_index_folder_reverse_remap_finds_existing_index -v
```

Expected: PASS.

- [ ] **Step 4.7: Run broader index_folder tests to check for regressions**

```bash
python -m pytest tests/test_incremental.py tests/test_index_file.py tests/test_local_integration.py -v
```

Expected: all PASS.

- [ ] **Step 4.8: Commit**

```bash
git add src/jcodemunch_mcp/tools/index_folder.py tests/test_path_map.py
git commit -m "feat: reverse-remap folder_path before repo hash in index_folder"
```

---

## Task 5: Reverse remap in `watcher.py`

Same reverse-remap treatment for `_local_repo_id` in the watcher (two call sites: watcher startup at line 272, cleanup at line 748).

**Files:**
- Modify: `src/jcodemunch_mcp/watcher.py` (lines 272, 748)
- Modify: `tests/test_path_map.py`

- [ ] **Step 5.1: Write a unit test**

Append to `tests/test_path_map.py`:

```python
from unittest.mock import patch


def test_watcher_local_repo_id_uses_reverse_remap(monkeypatch):
    """_watch_single calls _local_repo_id with the remapped (stored) path."""
    from jcodemunch_mcp import watcher as watcher_mod

    monkeypatch.setenv(ENV_VAR, "/remapped=/real")

    captured = {}

    original_fn = watcher_mod._local_repo_id

    def capturing_local_repo_id(path):
        captured["path"] = path
        return original_fn(path)

    with patch.object(watcher_mod, "_local_repo_id", side_effect=capturing_local_repo_id):
        # Drive just enough of _watch_single to hit the _local_repo_id call.
        # We stop immediately by raising inside the store constructor.
        try:
            import asyncio
            from jcodemunch_mcp.watcher import _watch_single

            async def run():
                # _watch_single loops forever; we interrupt on first store access
                with patch("jcodemunch_mcp.watcher.IndexStore", side_effect=RuntimeError("stop")):
                    try:
                        await _watch_single(folder_path="/remapped/myproject", storage_path=None)
                    except RuntimeError:
                        pass

            asyncio.run(run())
        except Exception:
            pass

    if captured:
        assert captured["path"].replace("\\", "/") == "/real/myproject", captured["path"]
```

> Note: this test uses a controlled interruption to exercise only the path-derivation part of `_watch_single`. If the watcher's internals make this too brittle, use `test_watch_claude.py` as a model for watcher unit-test patterns instead.

- [ ] **Step 5.2: Run the test to verify it fails**

```bash
python -m pytest tests/test_path_map.py::test_watcher_local_repo_id_uses_reverse_remap -v
```

Expected: FAIL (captured path is still `/remapped/myproject`).

- [ ] **Step 5.3: Add import to `watcher.py`**

In `src/jcodemunch_mcp/watcher.py`, add alongside existing imports:

```python
from .path_map import parse_path_map, remap
```

- [ ] **Step 5.4: Patch line 272 (`_watch_single` startup)**

Before:
```python
    repo_id = _local_repo_id(folder_path)
    _repo_owner, _repo_store_name = repo_id.split("/", 1)
```

After:
```python
    _pairs = parse_path_map()
    repo_id = _local_repo_id(remap(folder_path, _pairs, reverse=True))
    _repo_owner, _repo_store_name = repo_id.split("/", 1)
```

- [ ] **Step 5.5: Patch line 748 (cleanup branch)**

Before:
```python
        repo_id = _local_repo_id(folder)
        try:
            result = await asyncio.to_thread(
```

After:
```python
        _pairs = parse_path_map()
        repo_id = _local_repo_id(remap(folder, _pairs, reverse=True))
        try:
            result = await asyncio.to_thread(
```

- [ ] **Step 5.6: Run watcher test**

```bash
python -m pytest tests/test_path_map.py::test_watcher_local_repo_id_uses_reverse_remap -v
```

Expected: PASS (or skip gracefully if the watcher internals prevent the interruption pattern — in that case, manual inspection of the code change is sufficient).

- [ ] **Step 5.7: Run broader watcher tests to check for regressions**

```bash
python -m pytest tests/test_watcher_serve.py tests/test_watch_claude.py tests/test_watcher_lock.py -v
```

Expected: all PASS.

- [ ] **Step 5.8: Commit**

```bash
git add src/jcodemunch_mcp/watcher.py tests/test_path_map.py
git commit -m "feat: reverse-remap folder_path before repo hash in watcher"
```

---

## Task 6: Config display and CLAUDE.md

Show `JCODEMUNCH_PATH_MAP` in `jcodemunch-mcp config` output and document it in CLAUDE.md.

**Files:**
- Modify: `src/jcodemunch_mcp/server.py` (~line 1432)
- Modify: `CLAUDE.md`

- [ ] **Step 6.1: Add import of `ENV_VAR` to `server.py`**

Near the top of `server.py` where other module imports live, add:

```python
from .path_map import ENV_VAR as _PATH_MAP_ENV_VAR
```

- [ ] **Step 6.2: Add row to `_run_config` Core section**

In `_run_config`, immediately after the `JCODEMUNCH_EXTRA_IGNORE_PATTERNS` row:

Before:
```python
    extra = os.environ.get("JCODEMUNCH_EXTRA_IGNORE_PATTERNS", "")
    row("JCODEMUNCH_EXTRA_IGNORE_PATTERNS", extra if extra else dim("(none)"), not extra)
```

After:
```python
    extra = os.environ.get("JCODEMUNCH_EXTRA_IGNORE_PATTERNS", "")
    row("JCODEMUNCH_EXTRA_IGNORE_PATTERNS", extra if extra else dim("(none)"), not extra)
    path_map_val = os.environ.get(_PATH_MAP_ENV_VAR, "")
    row(_PATH_MAP_ENV_VAR, path_map_val if path_map_val else dim("(none)"), not path_map_val)
```

- [ ] **Step 6.3: Verify config output**

```bash
python -m jcodemunch_mcp.server config 2>/dev/null | grep PATH_MAP || \
  python -c "from jcodemunch_mcp.server import _run_config; _run_config()"
```

Expected: a line like `JCODEMUNCH_PATH_MAP          (none)` appears in the Core section.

- [ ] **Step 6.4: Run server tests to check for regressions**

```bash
python -m pytest tests/test_server.py tests/test_cli.py -v
```

Expected: all PASS.

- [ ] **Step 6.5: Update CLAUDE.md env vars table**

In `CLAUDE.md`, in the `## Env Vars` table, add after the `JCODEMUNCH_EXTRA_IGNORE_PATTERNS` row:

```markdown
| `JCODEMUNCH_PATH_MAP` | — | Remap stored path prefixes at retrieval time; format: `orig1=new1,orig2=new2`. Allows an index built on one machine (e.g. Linux `/home/user`) to be used on another (e.g. Windows `D:\Users\user`) without re-indexing |
```

- [ ] **Step 6.6: Commit**

```bash
git add src/jcodemunch_mcp/server.py CLAUDE.md
git commit -m "feat: add JCODEMUNCH_PATH_MAP to config display and CLAUDE.md docs"
```

---

## Task 7: Full test suite and branch cleanup

- [ ] **Step 7.1: Run the full test suite**

```bash
python -m pytest --tb=short -q
```

Expected: existing tests all PASS (885+); new `test_path_map.py` tests all PASS; no new failures.

- [ ] **Step 7.2: Verify the branch is on `feat/cross-platform-path-remap`**

```bash
git branch --show-current
```

Expected: `feat/cross-platform-path-remap`

If still on `main`, create the branch now (before the PR step):

```bash
git checkout -b feat/cross-platform-path-remap
```

---

## After implementation

Once all tasks are done:
1. Use `superpowers:finishing-a-development-branch` to handle the PR, issue, and merge options.
2. The GitHub issue should be filed on `jgravelle/jcodemunch-mcp` (upstream) before opening the PR on `iEdgir01/jcodemunch-mcp` (fork) so the PR can reference the issue.
