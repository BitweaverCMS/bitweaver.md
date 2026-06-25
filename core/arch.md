# Core — Bitweaver Framework Architecture

> Shared reference. Loaded every session for every developer.
> Update here when framework behaviour is confirmed by code inspection.
> Do not edit per-developer copies — this is the single source of truth.

---

## Global Objects (available in every PHP page after setup_inc.php)

| Variable          | Class           | Role |
|-------------------|-----------------|------|
| `$gBitSystem`     | `BitSystem`     | Config, package registry, display, error handling |
| `$gBitUser`       | `BitPermUser`   | Current authenticated (or anonymous) user + permissions |
| `$gBitThemes`     | `BitThemes`     | Style, layout, module management |
| `$gBitSmarty`     | `BitSmarty`     | Smarty instance; template rendering |
| `$gBitDb`         | ADOdb           | Shared database connection |
| `$gLibertySystem` | `LibertySystem` | Content-type registry; service dispatch |

Bootstrap order: `setup_inc.php` → constructs `$gBitSystem` and `$gBitSmarty`
→ scans active packages (`includes/bit_setup_inc.php` in each) → loads
`$gBitUser` → calls `config/kernel/override_inc.php` if present.

---

## Inheritance Chain

```
BitBase  (abstract — kernel)
  └─ LibertyBase  (liberty)
       └─ LibertyContent  (liberty) — base for all content objects
            └─ LibertyMime
                 └─ BitUser
                      └─ BitPermUser  (users)

BitSingleton  (kernel)
  ├─ BitSystem   (kernel)
  ├─ BitThemes   (themes)
  └─ ProductCatalog, etc.
```

---

## LibertyContent — Content Polymorphism

`liberty/includes/classes/LibertyContent.php`

The base for every content type in the system (wiki pages, blog posts,
photos, products, etc.). Polymorphism is driven by `mContentTypeGuid` —
each subclass declares its own GUID constant and sets:

```php
$this->mContentTypeGuid = WIKI_CONTENT_GUID; // example
```

`liberty_content` table stores one row per content object, keyed by
`content_id`. Each package extends this row in its own table.

### Key methods

| Method | Purpose |
|--------|---------|
| `load()` | Load content row + joined package data by `mContentId` |
| `verify( &$pParamHash )` | Validate input before store; sets `content_type_guid` |
| `store( &$pParamHash )` | INSERT or UPDATE; fires `invokeServices()` hooks |
| `expunge()` | DELETE content + cascade to all service tables |
| `isViewable()` | Check view permission for current user |
| `isEditable()` | Check edit permission |
| `hasPermission( $perm )` | Check named permission on this content object |
| `invokeServices( $func )` | Dispatch to all registered service plugins |
| `getLibertySql(...)` | Build JOIN/WHERE fragments for list queries |

### Pattern: subclass store/load

Package classes (e.g. `WikiPage`) call `parent::store()` / `parent::load()`
then insert/select their own table. Never write directly to `liberty_content`.

---

## BitPermUser — Permissions & Groups

`users/includes/classes/BitPermUser.php`

Represents the current user (`$gBitUser`). Permissions are string keys
(e.g. `'p_wiki_view'`) assigned to groups; users inherit via group membership.

### Key methods

| Method | Purpose |
|--------|---------|
| `hasPermission( $perm )` | Returns bool — preferred for conditionals |
| `verifyPermission( $perm, $msg )` | Dies with error page if permission absent |
| `isAdmin()` | True if user is in the Admins group |
| `isInGroup( $groupMixed )` | Check group membership by id or name |
| `loadPermissions()` | Reload from DB (called automatically on load) |

### Anonymous users

Unauthenticated users are an instance of `BitPermUser` with `mUserId = -1`.
`$gBitUser->isValid()` returns FALSE. All `hasPermission()` checks still work
— they just reflect the Anonymous group's assigned permissions.

---

## BitSystem — Configuration & Display

`kernel/includes/classes/BitSystem.php`

Singleton. Owns the site config table, package registry, and the main
render path.

### Key methods

| Method | Purpose |
|--------|---------|
| `getConfig( $key, $default )` | Read a config value |
| `setConfig( $key, $value )` | Set in memory (not persisted) |
| `storeConfig( $key, $value, $pkg )` | Persist to `kernel_config` table |
| `display( $mid, $title, $opts )` | Render a page: wraps `$mid` template in full HTML shell |
| `isPackageActive( $pkg )` | Check if a package is installed and active |
| `scanPackages( $scanFile )` | Discover and register all active packages |
| `fatalError( $msg )` | Render error page and exit |
| `verifyPermission( $perm )` | Delegates to `$gBitUser->verifyPermission()` |
| `outputJson( $data )` | Emit JSON response and exit |

### display() flow

`$gBitSystem->display( 'bitpackage:products/catalog.tpl', 'Title' )` →
renders `bitpackage:kernel/html.tpl` which includes the `$mid` template
in the centre column, wrapped by layout modules.

---

## BitThemes — Styles, Layouts & Modules

`themes/includes/classes/BitThemes.php`

Manages the active style (CSS), page layout (column configuration), and
sidebar/header/footer modules.

### Style path

Active style is stored in config key `'style'` (e.g. `cerulean`).
`getStylePath()` resolves to `CONFIG_PKG_PATH . 'themes/' . $styleName . '/'`
and defines `THEMES_STYLE_PATH`.

### Template (.tpl) resolution order

For `bitpackage:<package>/<template>.tpl`, the Smarty resource plugin
(`themes/smartyplugins/resource.bitpackage.php`) checks these locations
in order, using the first file that exists:

1. `config/themes/force/<package>/<template>.tpl` — unconditional override
2. `config/themes/force/<template>.tpl` — unconditional, no package prefix
3. `config/themes/<activestyle>/<package>/<template>.tpl` — theme override with package scope
4. `config/themes/<activestyle>/<template>.tpl` — theme override, no package prefix
5. `<package>/templates/<template>.tpl` — package default

`<activestyle>` is the value of the `style` config key (e.g. `cerulean`).
`THEMES_STYLE_PATH` resolves to `config/themes/<activestyle>/`.

The common pattern is to place site-specific overrides at
`config/themes/<activestyle>/some-file.tpl` (location 4) or
`config/themes/<activestyle>/<package>/some-file.tpl` (location 3).
The `force/` tier overrides regardless of which style is active.

CSS and JS overrides use `$gBitThemes->overrideCss()` /
`$gBitThemes->overrideJavascript()`.

### Module system

Modules are `.tpl` files (prefixed `mod_`) in `<package>/modules/`. If a
sibling `.php` file exists with the same name, the resource plugin executes
it first to populate Smarty variables before rendering the template.
