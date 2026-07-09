# Windows Setup (No Admin Rights)

This is not an officially supported or CI-tested platform — release artifacts are macOS/Linux
only, and CI runs on `ubuntu-latest` exclusively. This doc records a setup that has been verified
working end to end (build, `split`/`preprocess`/`vectorize`/`ingest`, and query) without any
source code changes required for the Windows build itself, and without administrator rights,
WSL2, or Docker.

If you have admin rights, WSL2 is simpler and matches the tested Linux target — use that instead
and skip this doc. This is for the case where you don't have admin access and can't install WSL2
or Docker Desktop (both require admin-gated Windows features).

## Rust toolchain (GNU target, no Visual Studio)

The default Rust toolchain on Windows uses MSVC, which needs Visual Studio Build Tools — typically
an admin-rights install. The GNU toolchain avoids this entirely.

1. Get a portable MinGW-w64 GCC build (no installer required) — e.g. from
   [winlibs.com](https://winlibs.com), a zip you extract anywhere and add to `PATH`.
2. Install `rustup` targeting the GNU host. The default `rustup-init` install itself does not
   need admin rights:
   ```bash
   rustup-init.exe -y --default-host x86_64-pc-windows-gnu --no-modify-path
   ```
   `--no-modify-path` keeps it from touching your permanent `PATH`; add the mingw and cargo `bin`
   directories to `PATH` for the session(s) where you build/run ragmail instead.
3. `cargo build --release` against the unmodified `rust/` workspace — no source patches are
   needed for this to work.

## Python side

No compiler needed — all dependencies (`torch`, `lancedb`, `sentence-transformers`, `pyarrow`,
etc.) publish prebuilt Windows wheels. `uv` is not required either:

```bash
python -m venv venv
venv\Scripts\pip install -e python
```

## Wiring the Rust CLI to a non-default venv location

`ragmail`'s Rust binary looks for the Python bridge at (in order): `RAGMAIL_PY_BRIDGE_BIN`, a
sibling `ragmail-py` next to the `ragmail` binary, then `.venv/bin/ragmail-py` /
`python/.venv/bin/ragmail-py` relative to the repo root (see `docs/developers.md`). If your venv
isn't at one of those default locations (e.g. you didn't use `just bootstrap`), set it explicitly:

```bash
set RAGMAIL_PY_BRIDGE_BIN=C:\path\to\venv\Scripts\ragmail-py.exe
```

## Skill install: symlinks don't work without Developer Mode

`.claude/skills/ragmail` and `.codex/skills/ragmail` are real symlinks in this repo, pointing at
`.agents/skills/ragmail/`. Windows only supports symlinks for non-admin users when Developer Mode
is enabled; otherwise a plain `git clone`/checkout on Windows silently turns them into a small
text file containing the link target path instead of an actual symlink, and the skill will not be
discovered.

Either enable Developer Mode, or just copy the directory instead of relying on the symlink:

```bash
cp -r .agents/skills/ragmail ~/.claude/skills/ragmail   # or wherever your client looks for skills
```

If you copy rather than symlink, remember the copy will drift from `.agents/skills/ragmail/` on
future updates — resync manually, or fix the symlink once Developer Mode is on.

## lancedb version is pinned deliberately

`python/pyproject.toml` pins `lancedb==0.30.0` rather than a floor (`>=0.20.0`). This isn't
Windows-specific — it affects any fresh install, any OS. Versions after ~0.30.x removed the
`use_tantivy=True` multi-column FTS API in favor of native FTS, which only supports one column per
index. This repo's schema builds one combined FTS index across multiple columns
(`subject`, `body_plain`, `from_name`, etc.) — re-architecting that to per-column native indices is
a real design change, not a compatibility shim, so the pin is the practical fix until/unless that
migration happens deliberately. Don't `pip install -U lancedb` or loosen this constraint without
also updating `python/lib/ragmail/storage/repository.py`'s FTS index creation and query logic.

If FTS queries ever fail with "Cannot perform full text search unless an INVERTED index has been
created" or a Tantivy-removal `ValueError`, the index is missing, stale, or was built against a
different lancedb version than is now installed — rebuild it:

```python
from ragmail.storage import Database, EmailRepository
db = Database("workspaces/<name>/db/email_search.lancedb")
EmailRepository(db, dimension=768).create_fts_index(force=True)
```
