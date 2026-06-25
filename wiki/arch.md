# wiki — Architecture Reference

> Shared reference. Update here when conventions or gotchas are confirmed.
> Upstream repo: https://github.com/bitweaver/wiki

---

## Overview

Wiki page package. Provides editable pages with version history, wiki
syntax parsing, and inter-page linking. Extends LibertyContent.
Content type GUID: `wiki_page`.

---

## Key Files

| File | Purpose |
|------|---------|
| `help/WikiPage.php` | Main page content class |
| `help/view_wiki.php` | Page view controller |
| `help/edit_wiki.php` | Page edit controller |
| `help/templates/` | Smarty templates |

---

## Data Model

| Table | Purpose |
|-------|---------|
| `wiki_pages` | Page content and metadata, keyed by `content_id` |

---

## Conventions

*To be documented.*

---

## Known Quirks & Gotchas

*To be documented.*
