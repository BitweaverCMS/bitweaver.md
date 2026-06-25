# CLAUDE.md — Bitweaver Development Guide

> Generic session instructions for developing on [Bitweaver CMS](https://www.bitweaver.org/wiki/Bitweaver+Overview).
> **No paths are hardcoded here.** All paths are injected by the developer
> bootstrap `CLAUDE.md` in your personal workspace before this file is read.
>
> This file is suitable for publication and is intended to be shared as a
> starting point for any team developing on Bitweaver with Claude Code.
>
> Site-specific overlays (proprietary packages, environment rules) belong in
> a separate architecture repo that loads this file as its base layer.

---

## Path Variables

All path references in this file use these variables. They must be set by
the developer bootstrap before this file is read.

| Variable | Purpose |
|----------|---------|
| `$BW_ROOT` | This repo — generic Bitweaver arch docs and this CLAUDE.md |
| `$ARCH_ROOT` | Site-specific architecture repo (proprietary packages, env rules). May be the same as `$BW_ROOT` for vanilla Bitweaver installs. |
| `$DEV_ROOT` | This developer's personal workspace (plans, notes, files) |
| `$DEPLOY_BASE` | Parent directory of all deployment clones |

If `$DEV_ROOT` is not set, Claude is in **admin mode** — see that section below.

---

## ⚠️ CRITICAL OPERATING RULES

Non-negotiable. Apply in every session.

0. **Session startup runs first, always.** Before responding to any first
   message — regardless of what it says — Claude must complete the Session
   Startup protocol below. No source file may be read until `$WORK_ROOT` is
   confirmed and architecture docs are loaded.

1. **No automatic changes.** Claude must never write, edit, delete, or rename
   any file without explicit user confirmation for that specific change.

2. **No automatic commits.** Claude must never run `git commit`, `git push`,
   `git merge`, `git rebase`, or any destructive git operation. Only the user
   may commit.

3. **Analyse first, act second.** For every problem:
   - Identify root cause and affected files/packages.
   - Propose a clearly described solution.
   - Wait for the user to say "implement" (or equivalent) before touching any file.

4. **Show diffs before applying.** Display the exact diff and ask for final
   confirmation before writing any change.

5. **One change at a time.** Each logical change is confirmed individually
   unless the user explicitly approves a batch.

6. **No package manager side-effects.** Do not run `composer install/update`,
   `npm install`, or equivalent without confirmation.

7. **Architectural discoveries go to `$ARCH_ROOT`.** When Claude confirms a
   convention, data model detail, gotcha, or any fact that all developers
   should know, it proposes an addition to `$ARCH_ROOT/<package>/arch.md`.
   These belong in the shared repo, not in a developer-local `memory/` file.

8. **Never search the `storage` module.** Do not run `ugrep`/`grep`/`rg`/
   `find`/glob (or any recursive scan) inside the **storage** package
   directory (`storage/`, e.g. `$WORK_ROOT/storage/`), or any directory named
   `storage`. It holds millions of generated/user files (rendered pages,
   thumbnails, uploads, caches); `storage/users/` and `storage/*/users/`
   specifically contain large volumes of user-uploaded assets. Crawling it
   causes **severe CPU and disk-I/O spikes** on the server. Always exclude it: use `--exclude-dir=storage`
   (grep), `-g '!storage'`/`--ignore-dir=storage` (rg/ugrep), or
   `-not -path '*/storage/*'` (find). To reach a specific generated file,
   **derive its path** from the storage-branch convention instead of scanning.

9. **Never delete files in `storage/`.** Files under any `storage/` directory
   must never be deleted by Claude under any circumstances. The user must
   delete storage files manually.

10. **Never delete any file without explicit user approval.** Propose the
    deletion and wait for the user to confirm before running any `rm` or
    equivalent command.

11. **Never run `sudo` without explicit per-command user confirmation.** Before
    issuing any `sudo` command, state the exact command and its effect, and
    wait for the user to approve that specific invocation. This applies even
    when a task obviously requires elevated privileges — ask first, every time.

---

## Project Overview

Bitweaver is a modular, object-oriented PHP CMS using **Smarty templates** and
**ADOdb** for database abstraction. It supports MySQL, PostgreSQL, SQLite,
Firebird, Oracle, MS SQL Server, and Sybase.

The upstream repository at `https://github.com/BitweaverCMS/bitweaver` is a
**git supermodule**; each package is a git submodule tracked independently
under `https://github.com/bitweaver/<package>`.

---

## Core Packages (always in scope)

| Package | Role |
|---------|------|
| **kernel** | Database initialisation, package configuration, bootstrap |
| **liberty** | Base content classes — access control, history, formatting, wiki parsing, HTML scrubbing |
| **users** | User data, authentication, session management |
| **themes** | Site theming, Smarty template resolution |
| **languages** | Internationalisation (i18n) |
| **util** | Shared utility functions used across packages |

Reference: https://www.bitweaver.org/wiki/Bitweaver+Framework

---

## All Upstream Packages

```
blogs       bnspell     calendar    downloads (treasury)
feed        forums (boards)         gatekeeper  kernel
languages   liberty     messages    newsletters
nexus       photos (fisheye)        pigeonholes quicktags
quota       rss         search (ilike)           sharethis
smileys     stars       stats       tags
themes      users       util        wiki
```

Each lives in its own subdirectory and git submodule. When working on an
optional package, always check for dependencies on core packages.

---

## Creating a New Package

Each package is its **own git repository**, wired into a deployment as a **git
submodule** (the deployment checkout is the supermodule; the full set is recorded
in the deployment's `.gitmodules`). A package's directory name may differ from its
package name — the site overlay keeps that mapping.

To scaffold a brand-new package — directory layout, the required
`includes/bit_setup_inc.php` registration, schema/permissions, the `index.php`
view controller + `.htaccess` routing, dependency reuse, and the submodule +
activation steps — follow the dedicated reference:

> **`$BW_ROOT/core/new-package.md`** — Creating a New Bitweaver Package (patterns + checklist).

This is distinct from *Initialising a new package **workspace*** (below), which
only creates this developer's `plans/notes/files/memory` scratch dirs for an
**existing** package.

---

## Session Startup

**Complete this protocol before responding to any first message.**

### Step 0 — Pre-fill detection

Check whether the first user message is in launcher format (sent by the
`bwdev` shell function):

- **`source=<name>`** — treat `<name>` as the pre-selected deployment; skip
  Steps 1–2 and go directly to Step 2b validation.
- **`source=<name> package=<name>`** — additionally pre-select the package;
  skip the package prompt in Step 4b.

If neither pattern is present, proceed with interactive Steps 1–2.

### Step 0b — Admin mode detection

If `$DEV_ROOT` is not set (no developer bootstrap was loaded), enter
**admin mode**. Skip Steps 1–4. See the Admin Mode section.

### Step 1 — Discover available deployments

Run:

```bash
ls $DEPLOY_BASE/*/kernel/includes/setup_inc.php 2>/dev/null \
  | sed "s|$DEPLOY_BASE/||; s|/kernel/includes/setup_inc.php||" | sort
```

Present the list to the user and ask which deployment to use.

### Step 2 — Confirm selection

> "Which deployment will we be working in?
> Discovered: **[list]**"

### Step 2b — Validate

```bash
ls $DEPLOY_BASE/<name>/kernel/includes/setup_inc.php
```

If absent, report the error and re-prompt. Do not proceed with an invalid
deployment.

If the site-specific overlay (`$ARCH_ROOT/CLAUDE.md`) defines any
environment-specific confirmation rules for the selected deployment (e.g.
requiring explicit confirmation for a production environment), apply them
now before setting `$WORK_ROOT`.

### Step 3 — Confirm working root

Set `$WORK_ROOT` = `$DEPLOY_BASE/<chosen>/` and echo:

> "Working root confirmed: `$DEPLOY_BASE/<chosen>/`"

Do not read or modify files outside `$WORK_ROOT`, `$DEV_ROOT`, `$ARCH_ROOT`,
or `$BW_ROOT`.

### Step 4 — Load core architecture

Read `$BW_ROOT/core/arch.md`. This documents the framework globals,
inheritance chain, LibertyContent polymorphism, BitPermUser, BitSystem,
and template resolution. Required every session regardless of package.

### Step 4b — Select package and load package architecture

Ask: *"Which package will we be focusing on this session?"*
(Skip if package was pre-filled in Step 0.)

Once confirmed:

1. Read `$BW_ROOT/<package>/arch.md` if it exists.
2. Read `$ARCH_ROOT/<package>/arch.md` if it exists (site-specific overlay;
   skip if `$ARCH_ROOT` == `$BW_ROOT`).
3. Check `$DEV_ROOT/<package>/plans/` for active plans and report what's found.

Confirm readiness:

> "Ready. Loaded: `$WORK_ROOT` | package: `<package>` | arch: [files loaded] | plans: [N found]
> What are we working on?"

---

## Session Workflow

```
1.  Session startup completes (Steps 0–4b above).
2.  Claude enters Plan Mode — confirm scope before any code work.
3.  Claude reads only the source files necessary for the task.
4.  Claude analyses and proposes a solution in plain language.
5.  User approves or redirects.
6.  Claude shows the exact diff.
7.  User confirms.
8.  Claude applies the change.
9.  Claude updates the plan file in $DEV_ROOT/<package>/plans/.
10. If the change surfaces a shared architectural fact, Claude proposes an
    addition to $ARCH_ROOT/<package>/arch.md for the user to commit.
11. User tests, then commits manually.
```

---

## Plan Mode

**Claude must be in Plan Mode before beginning any feature work or bug fix.**
A named, active plan file must exist before any code is read or changed.

### Entering Plan Mode

1. Check `$DEV_ROOT/<package>/plans/` for an existing matching plan.
2. If found:
   > "Found existing plan: `<filename>` — scope: <summary>.
   > Is this the plan we are working from, or do you want a new one?"
3. If not found:
   > "No existing plan for this. Describe the full scope so I can draft
   > one for your review before we begin."
4. Draft presented; no code touched until user approves.
5. On approval:
   > "Plan Mode active — working from: `<filename>`"

### Ambiguous scope

> "Is this a standalone fix, part of a larger feature, or an ad-hoc change?
> Should I create a plan, add to an existing one, or proceed without one?"

Trivial one-liners may proceed without a plan file if Claude states this
explicitly and the user agrees.

---

## Directory Structure

### Architecture repos

```
$BW_ROOT/                      Generic Bitweaver arch (this repo)
  CLAUDE.md                    This file
  core/
    arch.md                    Framework reference — loaded every session
  <package>/
    arch.md                    Upstream package arch — loaded when active
  templates/
    developer-CLAUDE.md        Stub for new developer workspace creation

$ARCH_ROOT/                    Site-specific arch (may equal $BW_ROOT)
  CLAUDE.md                    Site overlay — loads this file, adds overrides
  <package>/
    arch.md                    Proprietary package arch
```

### Developer workspace

```
$DEV_ROOT/                     Personal workspace — never deployed
  CLAUDE.md                    Bootstrap (sets all $variables, loads $ARCH_ROOT/CLAUDE.md)
  core/
    plans/  notes/  files/  memory/
  <package>/
    plans/                     One file per feature or fix
    notes/                     Scratch: research, traces, ideas
    files/                     Artefacts: SQL snippets, diffs, sample data
    memory/                    Local-only ephemeral context
```

### What goes where

| Location | Purpose |
|----------|---------|
| `$BW_ROOT/<pkg>/arch.md` | Upstream Bitweaver package facts |
| `$ARCH_ROOT/<pkg>/arch.md` | Site-specific/proprietary package facts |
| `$DEV_ROOT/<pkg>/plans/` | This developer's active and archived plans |
| `$DEV_ROOT/<pkg>/notes/` | Scratch research, error traces, ideas |
| `$DEV_ROOT/<pkg>/files/` | SQL snippets, example data, diffs |
| `$DEV_ROOT/<pkg>/memory/` | Developer-local context only — not shared |

### Initialising a new package workspace

> This creates only the developer's scratch dirs for an **existing** package. To
> create a new Bitweaver **code package** (its own submodule, `bit_setup_inc.php`,
> schema, controllers), see `$BW_ROOT/core/new-package.md`.

If `$DEV_ROOT/<package>/` does not exist, propose:

```bash
mkdir -p $DEV_ROOT/<package>/{plans,notes,files,memory}
```

Show the command; wait for confirmation before running.

---

## Admin Mode

Active when `$DEV_ROOT` is not set (Claude invoked directly from an arch repo
directory, no developer bootstrap). In admin mode Claude can browse and update
arch docs, and initialise new developer workspaces. Package workspace
operations are not available.

### Creating a new developer workspace

When asked to create a workspace (e.g. *"create developer workspace named
spider.md"*):

1. Verify the target directory does not already exist.
2. Present this batch and ask for **one confirmation**:
   ```
   mkdir <target>
   git init <target>
   cp $BW_ROOT/templates/developer-CLAUDE.md → <target>/CLAUDE.md
       (fill {{DEVNAME}}, {{BW_ROOT}}, {{ARCH_ROOT}}, {{DEV_ROOT}}, {{DEPLOY_BASE}})
   mkdir -p <target>/core/{plans,notes,files,memory}
   ```
3. Execute on confirmation.
4. Report: *"Workspace ready at `<target>`. Developer runs `claude` from that directory."*

---

## Plan File Format

```markdown
# <Package> — <Feature or Fix Title>

## Status
<!-- draft | active | blocked | complete -->

## Scope
<!-- what this covers and explicitly what it does NOT cover -->

## Dependencies
<!-- other packages or plans this depends on -->

## Approach
<!-- numbered steps -->

## Open Questions
<!-- needs user input or research before proceeding -->

## Decisions Log
### YYYY-MM-DD — <short title>
<rationale>

## Change History
### YYYY-MM-DD — <short title>
Files: <list>
Summary: <what changed and why>
```

Rules:
- Update plan files only after a change is applied and confirmed.
- Never update speculatively.
- Name descriptively: `feature-wiki-edit-permissions.md`, not `plan1.md`.

---

## Code Conventions

- **PHP style**: follow existing file conventions (procedural + OOP hybrid;
  classes in `<Package>Lib.php`).
- **Templates**: Smarty `.tpl` files in each package's `templates/`; never
  embed logic in templates.
- **Database**: always use the ADOdb abstraction layer (`$gBitDb`); never
  write raw PDO or mysqli calls.
- **Permissions**: use the Liberty/kernel permission API; never bypass ACL.
- **i18n**: all user-facing strings wrapped in `tra()`.
- **HTML**: templates must be valid HTML5.

---

## Git Safety Rules

```bash
# NEVER run automatically:
git commit
git push
git merge
git rebase
git reset --hard
git clean -fd

# Safe read-only operations:
git status
git diff
git log --oneline -n 20
git show <sha>
git branch -a
git submodule status
```

Commits are the user's sole responsibility. Provide a draft commit message
only if the user explicitly asks for one.

---

## Analysis Output Format

```
### Problem
<plain-language description>

### Root Cause
<file(s), function(s), query responsible — and why>

### Affected Packages
<list — always include core packages if impacted>

### Proposed Fix
<what changes and why it is safe>

### Files to Change
- path/to/file.php  — <one-line reason>

### Diff Preview
<shown after user says "show diff" or "proceed">
```

---

## What Claude Will NOT Do

- Modify any file without explicit per-change confirmation.
- Run git commit, push, or any history-altering command.
- Install or upgrade dependencies without confirmation.
- Assume the live database schema — ask the user for `SHOW TABLES` / `\d`
  output if schema knowledge is needed.
- Skip error handling or ACL checks in proposed code.
- Access or modify files outside `$WORK_ROOT`, `$DEV_ROOT`, `$ARCH_ROOT`,
  or `$BW_ROOT`.
- Run `ugrep`/`grep`/`rg`/`find`/glob inside the `storage` module (or any
  `storage/` directory) — it spikes server CPU and disk. Always exclude it.
- Write shared architectural discoveries to `$DEV_ROOT/memory/` — these
  belong in `$ARCH_ROOT/<package>/arch.md`.

---

## Useful References

- Bitweaver Overview: https://www.bitweaver.org/wiki/Bitweaver+Overview
- Framework & Package List: https://www.bitweaver.org/wiki/Bitweaver+Framework
- Supermodule repo: https://github.com/BitweaverCMS/bitweaver
- Individual package repos: https://github.com/bitweaver/<package>
- ADOdb docs: https://adodb.org/dokuwiki/doku.php
- Smarty docs: https://www.smarty.net/docs/en/
