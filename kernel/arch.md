# Kernel — Bitweaver Bootstrap Architecture

> Covers `kernel/includes/setup_inc.php` — the master bootstrap executed by
> every PHP page entry-point. Load order, globals constructed, and post-setup
> hooks available to packages and site overrides.

---

## Bootstrap Sequence (`setup_inc.php`)

All page controllers begin with:

```php
require_once( dirname(__FILE__).'/../kernel/includes/setup_inc.php' );
```

The file executes in the following order:

### 1. Root path detection

```php
define( 'BIT_ROOT_PATH', empty($_SERVER['VHOST_DIR']) ? $rootDir.'/' : $_SERVER['VHOST_DIR'].'/' );
```

`VHOST_DIR` server variable allows Apache VirtualHost overrides of the document root.

### 2. Core definitions and error handler

```php
require_once( BIT_ROOT_PATH.'kernel/includes/config_defaults_inc.php' );
require_once( KERNEL_PKG_INCLUDE_PATH.'bit_error_inc.php' );
require_once( KERNEL_PKG_INCLUDE_PATH.'kernel_lib.php' );
```

`config_defaults_inc.php` defines all `*_PKG_PATH`, `*_PKG_INCLUDE_PATH`,
`*_PKG_CLASS_PATH`, and `*_PKG_URL` constants for all packages.

### 3. Input sanitisation

```php
detoxify( $_GET, TRUE, FALSE );
$_REQUEST = array_merge( $_REQUEST, $_GET );
```

Runs before any package code; `detoxify()` is in `kernel_lib.php`.

### 4. Database connection

```php
$gBitDb = new $dbClass();   // $dbClass = 'BitDbAdodb' by default
```

Single shared ADOdb connection. All packages share `$gBitDb`.

### 5. `BitSystem` singleton + `BitSmarty`

```php
$gBitSmarty = new BitSmarty();
$gBitSmarty->loadFilter( 'pre', 'tr' );   // i18n pre-filter
BitSystem::loadSingleton();                // reads kernel_config table → $gBitSystem
```

`BitSystem::loadSingleton()` populates `$gBitSystem->mConfig` from the database.
After this point `$gBitSystem->getConfig()` / `isFeatureActive()` are safe to call.

### 6. Version check

If the installed database schema is older than `MIN_BIT_VERSION`, sets
`INSTALLER_FORCE` to redirect to the installer.

### 7. `scanPackages()` — load all active packages

```php
$gBitSystem->scanPackages( 'includes/bit_setup_inc.php', TRUE, 'active', TRUE, TRUE );
```

Iterates every active package directory, `require_once`-ing
`<package>/includes/bit_setup_inc.php`. **The users package `bit_setup_inc.php`
runs inside this call** — so `$gBitUser` is constructed and fully loaded before
`scanPackages()` returns.

### 8. Override hook

```php
if( file_exists( CONFIG_PKG_INCLUDE_PATH.'kernel/override_inc.php' ) ) {
    require_once( CONFIG_PKG_PATH.'kernel/override_inc.php' );
}
```

This is the canonical site-specific hook that runs **after** all packages are
loaded but **before** Smarty is assigned `$gBitSystem`. Create
`config/kernel/override_inc.php` to inject per-site globals or post-setup logic.

> **Note:** `CONFIG_PKG_INCLUDE_PATH` is checked for existence, but
> `CONFIG_PKG_PATH` is used for the actual `require_once`. Both should resolve to
> `config/kernel/` for normal deployments; confirm the constants match if the file
> is not being picked up.

### 9. Post-setup assignments and checks

After `scanPackages()` and the override hook:

- `$gBitSmarty->assignByRef('gBitSystem', $gBitSystem)`
- LibertySystem `preload_function` plugins are invoked
- Site-closed check (`site_closed` feature flag vs `$gBitUser->hasPermission('p_access_closed_site')`)
- Server load threshold check (`/proc/loadavg` vs `site_load_threshold`)
- HTTPS login redirect logic
- Banned user check (`content_status_id < 0` → fatal error)

---

## Key Constants (defined in `config_defaults_inc.php`)

| Constant | Resolves to |
|----------|-------------|
| `BIT_ROOT_PATH` | Deployment root with trailing `/` |
| `CONFIG_PKG_PATH` | `BIT_ROOT_PATH . 'config/'` |
| `CONFIG_PKG_INCLUDE_PATH` | `BIT_ROOT_PATH . 'config/includes/'` |
| `KERNEL_PKG_PATH` | `BIT_ROOT_PATH . 'kernel/'` |
| `KERNEL_PKG_INCLUDE_PATH` | `BIT_ROOT_PATH . 'kernel/includes/'` |
| `KERNEL_PKG_CLASS_PATH` | `BIT_ROOT_PATH . 'kernel/includes/classes/'` |
| `EXTERNAL_LIBS_PATH` | `BIT_ROOT_PATH . 'config/externals/'` |
| `BIT_SESSION_NAME` | Session cookie name (from `config_defaults_inc.php`) |
| `ANONYMOUS_USER_ID` | Numeric ID of the anonymous user (typically `0`) |

---

## Security guards in `setup_inc.php`

- **`sort_mode` injection guard** (line 19): kills the request immediately if
  `$_REQUEST['sort_mode']` contains `'http'` (prevents an old SQL injection vector).
- **`detoxify()`** strips dangerous patterns from `$_GET` before merging into
  `$_REQUEST`.

---

## Input from Apache environment

| `$_SERVER` key | Set by | Purpose |
|----------------|--------|---------|
| `VHOST_DIR` | Apache `SetEnv` | Override deployment root (multi-site) |
| `HTTP_USER_AGENT` | Request header | Initialised to `""` by `users/bit_setup_inc.php` if absent |
| `HTTPS` | Apache mod_ssl | HTTPS detection for `BIT_BASE_URI` |

---

## Shell-script mode

```php
global $gShellScript;
```

If `$gShellScript` is set before `setup_inc.php` is included (CLI scripts),
session handling and certain web-only checks are skipped in package setup files.

---

## BitSystem — selected methods

Documented in `$BW_ROOT/core/arch.md`. The kernel `setup_inc.php` constructs
`$gBitSystem` via `BitSystem::loadSingleton()`. By the time any page controller
runs, `$gBitSystem` is fully ready.
