# Storage — Architecture Reference

> Shared reference. Loaded when storage layout is relevant.
> Never scan this directory — it holds large volumes of user-uploaded and generated data.

---

## Overview

`storage/` is the runtime data directory. It holds user-uploaded files,
generated thumbnails, rendered pages, and caches. It is not a source
package and contains no PHP, templates, or configuration.

---

## Structure

| Path | Contents |
|------|----------|
| `storage/users/` | Per-user uploaded assets (photos, originals, etc.) |
| `storage/*/users/` | Per-user assets in multi-tenant or named-site variants |
| `storage/<other>/` | Generated files: rendered PDFs, thumbnails, page previews, caches |

---

## Search / Grep Rules

Never descend into `storage/` or any of its subdirectories. The directory
may contain millions of files. Recursive scans cause severe CPU and disk-I/O
spikes on the server.

Always exclude it explicitly:

| Tool | Flag |
|------|------|
| `grep` | `--exclude-dir=storage` |
| `rg` / `ugrep` | `-g '!storage'` or `--ignore-dir=storage` |
| `find` | `-not -path '*/storage/*'` |

To locate a specific generated file, derive its path from the storage
naming convention rather than scanning.
