# Creating a New Bitweaver Package

> Generic, repeatable reference for scaffolding a brand-new Bitweaver package.
> Loaded on demand (not every session) — read this when planning or creating a
> new package. Site-specific facts (remote URLs, directory→package mapping,
> database engine) live in the site overlay (`$ARCH_ROOT`); this file is generic.

A **package** is a self-contained, independently-versioned unit of Bitweaver
functionality: its own directory at the deployment root, its own git repository,
self-registration with the kernel, and (optionally) its own DB tables,
permissions, controllers, templates, and assets.

---

## 1. Every package is its own git repo (a submodule)

Each package lives in its **own git repository** and is wired into a deployment
as a **git submodule** — the deployment checkout is the supermodule. The set of
submodules is recorded in the deployment's `.gitmodules`.

Remote URL conventions (see the site overlay for the authoritative list):

| Package origin | Remote convention |
|----------------|-------------------|
| Upstream Bitweaver | `git@github.com:bitweaver/<package>.git` |
| Site / proprietary | site SCM, e.g. `ssh://<scm-host>/home/source/git/<package>.git` |

The **directory name may differ from the package name** (the site overlay keeps a
directory→package mapping; e.g. `photos/`→`fisheye`, `bookstore/`→`commerce`).

**Sequencing:** build and stabilise the package files in place first. Initialise
the package's git repo and add it as a submodule only once it is ready for an
initial commit — not before. Until then it is simply an untracked directory in
the supermodule (`?? <package>/`).

---

## 2. Minimal required structure

The kernel discovers a package by scanning every deployment subdirectory for
`includes/bit_setup_inc.php` (`BitSystem::scanPackages( 'includes/bit_setup_inc.php', ... )`).
That one file is the **only** hard requirement for discovery + registration:

```
<package>/
└── includes/
    └── bit_setup_inc.php      # REQUIRED — registers the package with the kernel
```

---

## 3. Registration — `includes/bit_setup_inc.php`

```php
<?php
global $gBitSystem;

$registerHash = array(
    'package_name' => '<package>',
    'package_path' => dirname( dirname( __FILE__ ) ).'/',
    // 'service'   => '<service>',   // optional: Liberty content/service dispatch
);
$gBitSystem->registerPackage( $registerHash );

if( $gBitSystem->isPackageActive( '<package>' ) ) {
    // Load classes/assets, register menus/services — ONLY when active.
    // If this package depends on another, assert it here:
    // $gBitSystem->verifyPackage( '<dependency>' );
}
```

`registerPackage()` auto-defines the package constants — **always use these,
never hardcode paths**:

`<PACKAGE>_PKG_NAME`, `<PACKAGE>_PKG_PATH`, `<PACKAGE>_PKG_INCLUDE_PATH`,
`<PACKAGE>_PKG_CLASS_PATH`, `<PACKAGE>_PKG_ADMIN_PATH`, `<PACKAGE>_PKG_URL`,
`<PACKAGE>_PKG_URI`.

`registerAppMenu()` / `registerService()` are optional and belong inside the
`isPackageActive()` block. Do not register a menu whose `.tpl` does not exist yet.

---

## 4. Secondary bootstrap — `includes/<package>_setup_inc.php` (optional)

Heavyweight setup (requiring classes, loading JS/CSS via `$gBitThemes`) belongs
here, required from controllers — not in `bit_setup_inc.php`. Note: assets a
controller emits **page-locally** (its own `<link>`/`<script>` tags) must **not**
also be registered globally with `$gBitThemes`, or they leak into every themed
page.

A package that **reuses another package's engine** requires that package's setup
here (e.g. `require_once DESIGNER_PKG_INCLUDE_PATH.'designer_setup_inc.php';`) and
calls its **public class API** — it does not modify the other package.

---

## 5. Controllers & routing

- **Main view controller is `<package>/index.php`** (every package follows this):

  ```php
  require_once( '../kernel/includes/setup_inc.php' );
  require_once( <PACKAGE>_PKG_INCLUDE_PATH.'<package>_setup_inc.php' );
  $gBitSystem->verifyPackage( '<package>' );
  // ... build $output / assign Smarty vars ...
  $gBitSystem->display( 'bitpackage:<package>/<template>.tpl', $browserTitle );
  ```

- **Pretty URLs** via `<package>/.htaccess` (mod_rewrite): the standard boilerplate
  (pass through real files and transparent `.php`) plus package rules, e.g.
  `RewriteRule ^([0-9]+)$ index.php?products_id=$1 [L,QSA]`.

- **AJAX/JSON endpoints** are plain `*.php` controllers in the package root that
  bootstrap the kernel + setup, gate on permission, and `print` their response.

---

## 6. Schema & install — `admin/schema_inc.php` (only if the package owns data)

Run on install/activation. Uses `$gBitInstaller`:

```php
global $gBitInstaller;
$gBitInstaller->registerSchemaTable( <PACKAGE>_PKG_NAME, $tableName, $tableDef );   // new tables
$gBitInstaller->registerSchemaIndexes( <PACKAGE>_PKG_NAME, $indices );
$gBitInstaller->registerSchemaSequences( <PACKAGE>_PKG_NAME, $sequences );
$gBitInstaller->registerSchemaDefault( <PACKAGE>_PKG_NAME, array( /* seed SQL */ ) );
$gBitInstaller->registerPackageInfo( <PACKAGE>_PKG_NAME, array(
    'description' => '…', 'version' => '0.1', 'state' => 'beta', 'dependencies' => '<dep>' ) );
$gBitInstaller->registerUserPermissions( <PACKAGE>_PKG_NAME, array(
    array( 'p_<package>_view', '…', 'registered', <PACKAGE>_PKG_NAME ), /* … */ ) );
$gBitInstaller->registerRequirements( <PACKAGE>_PKG_NAME, array( '<dep>' => array( 'min' => '0.0.0' ) ) );
```

**Additive columns on another package's table:** keep the owning package pristine —
put an **idempotent** migration in *this* package's `schema_inc.php` via
`registerSchemaDefault`, e.g.
`"ALTER TABLE ".BIT_DB_PREFIX."<other_table> ADD COLUMN IF NOT EXISTS <col> <type>"`.
(Column type SQL is engine-specific — see the site overlay for the deployment's DB.)

---

## 7. Standard directory layout

```
<package>/
├── includes/
│   ├── bit_setup_inc.php           # REQUIRED
│   ├── <package>_setup_inc.php     # secondary bootstrap (optional)
│   ├── classes/                    # <Class>.php business logic
│   └── *_inc.php                   # shared includes (lookups, libs)
├── admin/
│   └── schema_inc.php              # DB schema + permissions (if it owns data)
├── templates/  *.tpl               # Smarty templates
├── modules/    mod_*.php|.tpl      # sidebar/content modules
├── css/  js/  icons/  images/      # assets
├── index.php                       # main view controller
├── *.php                           # other controllers / AJAX endpoints
└── .htaccess                       # pretty-URL rewrites
```

Only `includes/bit_setup_inc.php` is strictly required; everything else is added
as the package needs it.

---

## 8. Activation

A discovered package is **inactive** until installed/activated (Admin → Packages,
which runs `admin/schema_inc.php`). Activation state is stored as the config key
`package_<name>`.

---

## 9. New-package checklist

- [ ] Choose package name + directory name (add a site-overlay mapping row if they differ).
- [ ] `includes/bit_setup_inc.php` — `registerPackage()` (+ `verifyPackage()` for deps).
- [ ] `includes/<package>_setup_inc.php` — if there is heavyweight setup.
- [ ] `admin/schema_inc.php` — tables / permissions / requirements, if it owns data.
- [ ] `index.php` main controller + `.htaccess` routes; AJAX endpoints as needed.
- [ ] Templates / CSS / JS — emit page-local assets from controllers, not globally.
- [ ] Reusing another package? `verifyPackage` + `registerRequirements`, and call its
      **public class API** — never edit the dependency.
- [ ] `php -l` every controller; confirm package constants resolve; confirm zero
      changes leak into any package you depend on.
- [ ] When stable: `git init` the package repo, push to the SCM remote, add as a
      submodule (record in `.gitmodules`), then install/activate.
- [ ] Document the package's facts in `$ARCH_ROOT/<package>/arch.md`.

---

## Related references

- Framework globals, inheritance, template resolution: `$BW_ROOT/core/arch.md`
- Per-package architecture: `$BW_ROOT/<package>/arch.md` (upstream) /
  `$ARCH_ROOT/<package>/arch.md` (site/proprietary)
- Remote URL conventions, directory→package mapping, DB engine: the site overlay
  (`$ARCH_ROOT/CLAUDE.md`)
