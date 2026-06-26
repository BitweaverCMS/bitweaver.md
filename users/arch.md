# Users — Package Architecture

> Covers `users/includes/bit_setup_inc.php` (package bootstrap),
> `BitPermUser` / `BitUser` (class hierarchy), session and cookie management,
> and the Apache log integration point.

---

## Package Bootstrap (`users/includes/bit_setup_inc.php`)

Executed by `$gBitSystem->scanPackages()` during kernel setup. At exit,
`$gBitUser` is always a valid `BitPermUser` instance (never null).

### User class resolution

```php
$userClass = $gBitSystem->getConfig( 'user_class', 'BitPermUser' );
require_once( USERS_PKG_CLASS_PATH.$userClass.'.php' );
```

Sites can override `$gBitUser`'s class via the `user_class` config key.

### Session start

```php
session_name( BIT_SESSION_NAME );
session_set_cookie_params( ... );
session_start();   // skipped when $gShellScript is set
```

Session lifetime and cookie params are driven by `site_session_lifetime`,
`users_remember_me`, `cookie_path`, and `cookie_domain` config keys.

Session storage defaults to the filesystem; set `site_store_session_db` to
store sessions in the `bit_sessions` table via ADOdb Session.

### Three user-load paths

All three paths converge on `$gBitUser` being fully loaded before the file exits.

#### Path A — Override login function

```php
global $gOverrideLoginFunction;
if( !empty( $gOverrideLoginFunction ) ) {
    $gBitUser = new $userClass();
    $gBitUser->mUserId = $gOverrideLoginFunction();
    if( $gBitUser->mUserId ) {
        $gBitUser->load();
        $gBitUser->loadPermissions();
    }
}
```

Allows non-cookie auth mechanisms (API keys, SSO) to inject a user ID.

#### Path B — Cookie-based load

```php
$siteCookie = $userClass::getSiteCookieName();
if( !empty( $_COOKIE[$siteCookie] ) ) {
    if( $gBitUser = $userClass::loadFromCache( $_COOKIE[$siteCookie] ) ) {
        // served from object cache
    } else {
        $gBitUser = new $userClass();
        if( $gBitUser->mUserId = $gBitUser->getUserIdFromCookieHash( $_COOKIE[$siteCookie] ) ) {
            $gBitUser->load( TRUE );
        }
    }
}
```

The site cookie value is an opaque hash linked to `users_cnxn` table rows.
Invalidating rows in `users_cnxn` forces a logout more reliably than session expiry.
`loadFromCache()` is a static method — a per-process in-memory cache.

#### Path C — Anonymous fallback

```php
if( empty( $gBitUser ) || !$gBitUser->isValid() ) {
    if( !($gBitUser = $userClass::loadFromCache( ANONYMOUS_USER_ID )) ) {
        $gBitUser = new $userClass( ANONYMOUS_USER_ID );
        $gBitUser->load( TRUE );
    }
}
```

Anonymous user always has `mUserId = ANONYMOUS_USER_ID` and
`$gBitUser->isValid()` returns FALSE.

### Post-load steps

After `$gBitUser` is loaded:

1. `$gBitSmarty->assignByRef('gBitUser', $gBitUser)` — available in all templates
2. CSRF challenge generation (`feature_challenge` → `$_SESSION['challenge']`)
3. Virtual domain routing (`users_domains` feature)
4. Per-user theme override (`users_themes = 'y'`)
5. **Apache log note** — `apache_note('bw_login', ...)` (see below)
6. App menu registration (`My Site` menu for registered users)

---

## Apache Log Integration

```php
if( function_exists('apache_note') ) {
    apache_note( 'bw_login', $gBitUser->getField('login', '-') );
}
```

Sets the Apache note `bw_login` on every PHP request after the user is loaded.
Used in the LogFormat as `%{bw_login}n`. Anonymous users produce `'-'`.

The `function_exists` guard makes this safe under PHP-FPM (where `apache_note`
is not available); under PHP-FPM the field will always show `'-'`.

Corresponding LogFormat entry (in `inc_app_env.txt`):

```
LogFormat "%V %h %{bw_login}n %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-agent}i\" \"%{Cookie}n\""  combinedcookie
```

Static assets (CSS, images, JS) served by Apache without invoking PHP will also
show `'-'` for `%{bw_login}n` — expected and correct.

---

## Class Hierarchy

```
BitBase  (kernel)
  └─ LibertyBase  (liberty)
       └─ LibertyContent  (liberty)
            └─ BitUser   (users/includes/classes/BitUser.php)
                 └─ BitPermUser  (users/includes/classes/BitPermUser.php)
```

`$gBitUser` is always a `BitPermUser` instance (or subclass via `user_class` config).

### Key `BitUser` methods

| Method | Purpose |
|--------|---------|
| `load( $pFull )` | Load user row from DB; `$pFull=TRUE` also loads preferences |
| `getField( $key, $default )` | Read a user data field (e.g. `'login'`, `'email'`) |
| `isValid()` | TRUE if `mUserId > 0` (i.e., not anonymous) |
| `isRegistered()` | TRUE if user is fully registered (not just anonymous) |
| `getUserIdFromCookieHash( $hash )` | Resolve cookie hash → user ID via `users_cnxn` |
| `getSiteCookieName()` | Static; returns the deployment's cookie name |
| `loadFromCache( $id )` | Static; returns cached instance or FALSE |

### Key `BitPermUser` methods

| Method | Purpose |
|--------|---------|
| `loadPermissions()` | Load groups and permission strings for this user |
| `hasPermission( $perm )` | Returns bool; preferred for conditionals |
| `verifyPermission( $perm, $msg )` | Dies with error page if absent |
| `isAdmin()` | TRUE if user is in the Admins group |
| `isInGroup( $groupMixed )` | Check group by ID or name |

---

## CSRF / Challenge System

When `feature_challenge` is active and the request is not to `users/validate.php`:

```php
$_SESSION['challenge'] = $gBitUser->generateChallenge();
```

The generated challenge token must be passed back in form submissions and
validated by the target controller.

---

## Virtual Domain Routing (`users_domains`)

If `users_domains` is active, the subdomain is extracted from `HTTP_HOST` and
matched against the users domain table. On a match, `$_REQUEST['user_id']` and
`$gBitSystem->mDomainInfo` are populated for downstream controllers.
