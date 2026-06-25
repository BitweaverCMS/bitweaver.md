# sms — SMS Notification Package

> Standalone Bitweaver package. **Must be activated via Admin → Packages** before
> it is available. Other packages detect it with `defined('SMS_PKG_CLASS_PATH')`.
>
> Update this file when gateway behaviour, config keys, or broker conventions
> are confirmed by code inspection.

---

## Overview

Provides outbound SMS notifications reusable across any Bitweaver install.
The architecture is two-layer:

- **`SmsGatewayInterface`** — contract each provider must satisfy.
- **`SmsBroker`** — factory, registry, phone resolution, and E.164 formatting.
  Admin forms are built entirely from `SmsBroker` methods; no config keys are
  defined anywhere else.

Phone numbers are parsed and validated with **libphonenumber** (PSR-0 autoloaded
from `config/externals/libphonenumber/`).

---

## File Map

| File | Purpose |
|------|---------|
| `includes/bit_setup_inc.php` | Package boot; registers libphonenumber PSR-0 autoloader; defines `SMS_PKG_CLASS_PATH`, `SMS_PKG_NAME`, `SMS_PKG_PATH` |
| `includes/classes/SmsGatewayInterface.php` | Gateway contract |
| `includes/classes/SmsBroker.php` | Factory / registry / orchestrator |
| `includes/classes/gateways/AnveoSmsGateway.php` | Anveo HTTP API |
| `includes/classes/gateways/TwilioSmsGateway.php` | Twilio REST API |
| `includes/classes/gateways/PlivoSmsGateway.php` | Plivo REST API |
| `includes/classes/gateways/TelnyxSmsGateway.php` | Telnyx v2 API |
| `admin/index.php` | Admin config + test-send controller |
| `templates/admin_sms_config.tpl` | Admin config UI (dynamic — no hardcoded keys) |

---

## SmsGatewayInterface

```php
interface SmsGatewayInterface {
    public static function getGatewayName(): string;  // returns the registry key, e.g. 'twilio'

    public function send( string $pToE164, string $pBody ): bool;
    public function getLastError(): ?string;
}
```

**Convention** (not enforced by the interface — required for SmsBroker integration):

```php
public static function getGatewayLabel(): string;    // human-readable name for UI
public static function getConfigKeys(): array;       // see Config Key Format below
```

`getGatewayName()` is enforced by the interface and is used by `buildRegistry()` for
auto-discovery (`is_a($class, 'SmsGatewayInterface', TRUE)` + `$class::getGatewayName()`).
Each gateway also defines `const GATEWAY_NAME = '<key>'` and implements the method as
`return self::GATEWAY_NAME` — the constant is available for direct reference; the method
satisfies the interface contract.

Note: PHP 7.4 does not allow implementing classes to override interface constants, so
`GATEWAY_NAME` lives on each concrete class, not the interface.

---

## SmsBroker — Public API

| Method | Signature | Notes |
|--------|-----------|-------|
| `registerGateway` | `(string $name, string $class, string $file): void` | Register an external gateway. If called before the registry is built, the entry is queued and processed on first use. `$file` must be a full filesystem path. |
| `getInstance` | `(?string $name = NULL): ?SmsGatewayInterface` | Returns cached instance. NULL uses `sms_default_gateway` config (default `anveo`). |
| `getBrokerConfigKeys` | `(): array` | Returns the three broker-level key definitions only (enabled, default_gateway, send_when_not_live). Used by admin template for the always-visible section. |
| `getGatewayConfigKeys` | `(): array` | Returns per-gateway sections: `[name => ['label' => '...', 'keys' => [...]]]`. Used by admin template to render each gateway's settings div. |
| `getAllConfigKeys` | `(): array` | Flat merge of broker keys + all gateway keys. Used by the admin save loop. |
| `getGatewayOptions` | `(): array` | `name => label` map. Used by the test-send gateway selector. |
| `resolvePhone` | `($order): ?array` | Returns `['number' => raw, 'iso2' => 'US']`; tries `billing.telephone` first, then `delivery.telephone`. |
| `formatE164` | `(string $raw, string $iso2): ?string` | Parses with libphonenumber; returns E.164 string or null. |
| `notify` | `($order, string $msg, ?string $gateway, ?string $phoneOverride): bool` | Full orchestration. Checks `sms_enabled`. Suppresses on non-live sites unless `sms_send_when_not_live` is set (logs suppression). Stores errors in `$order->mErrors['sms']`. |
| `sendTest` | `(string $raw, string $iso2, string $msg, ?string $gateway): array` | Admin test helper. Returns `['success' => bool, 'error' => ?string]`. Bypasses order resolution. |

### Registry auto-discovery

On first access of the registry, `buildRegistry()` calls
`glob( SMS_PKG_CLASS_PATH.'gateways/*SmsGateway.php' )`, requires each file,
and reads `$class::GATEWAY_NAME`. Any class with a non-empty `GATEWAY_NAME` is
registered. Files not matching `*SmsGateway.php` are ignored.

---

## Config Keys

All config values are stored via `$gBitSystem->storeConfig()`.

### Broker-level

| Key | Type | Purpose |
|-----|------|---------|
| `sms_enabled` | select | Master on/off |
| `sms_default_gateway` | select | Active gateway (auto-populated from discovered gateways) |
| `sms_send_when_not_live` | select | Allow sending on non-live sites (default: suppress + log) |

### Anveo

| Key | Type | Purpose |
|-----|------|---------|
| `sms_anveo_from_number` | text | Sender DID (E.164) |
| `sms_anveo_userkey` | text | API key (labeled USERKEY in portal) |

### Twilio

| Key | Type | Purpose |
|-----|------|---------|
| `sms_twilio_from_number` | text | Sender number (E.164) |
| `sms_twilio_account_sid` | text | Account SID |
| `sms_twilio_auth_token` | text | Auth token |

### Plivo

| Key | Type | Purpose |
|-----|------|---------|
| `sms_plivo_from_number` | text | Sender number (E.164) |
| `sms_plivo_auth_id` | text | Auth ID (from Plivo console) |
| `sms_plivo_auth_token` | password | Auth token |

### Telnyx

| Key | Type | Purpose |
|-----|------|---------|
| `sms_telnyx_from_number` | text | Sender number (E.164) |
| `sms_telnyx_api_key` | password | API key (from Telnyx portal) |

---

## Adding a New Gateway

1. Create `sms/includes/classes/gateways/MyProviderSmsGateway.php` implementing
   `SmsGatewayInterface`. The file name must end in `SmsGateway.php`.

2. Define `GATEWAY_NAME` and the static convention methods:

   ```php
   class MyProviderSmsGateway implements SmsGatewayInterface {
       const GATEWAY_NAME = 'myprovider';

       public static function getGatewayLabel(): string { return 'My Provider'; }
       public static function getConfigKeys(): array { return [ ... ]; }

       public function send( string $pToE164, string $pBody ): bool { ... }
       public function getLastError(): ?string { ... }
   }
   ```

3. That's it. SmsBroker discovers the file automatically on next page load.
   No edits to SmsBroker or the admin template are required.

**For external packages** (gateway lives outside the `sms/` package directory):

```php
SmsBroker::registerGateway( 'myprovider', 'MyProviderSmsGateway', '/full/path/to/MyProviderSmsGateway.php' );
```

Call this from the external package's `bit_setup_inc.php` (before any SmsBroker method is invoked).

**HTTP transport convention:** use `stream_context_create()` + `file_get_contents()`
with `'ignore_errors' => TRUE` and `'timeout' => 15`. Do not introduce curl dependencies.

---

## Admin UI

The config page renders in two parts:

1. **General Settings** (always visible) — broker-level keys from `getBrokerConfigKeys()`.
2. **Gateway Settings** (one `<div>` per gateway, show/hide) — per-gateway keys from
   `getGatewayConfigKeys()`. A vanilla JS snippet tied to the Default Gateway selector
   shows the matching section and hides the rest on page load and on change.

The save form always submits all gateway fields (hidden sections are still in the DOM),
so switching gateways does not wipe pre-configured credentials for inactive providers.

---

## bookstore Integration

### `CommerceOrder::updateStatus()` — `bookstore/includes/classes/CommerceOrder.php`

SMS block runs after the email notification block (line ~1452). Triggers when:
- `defined('SMS_PKG_CLASS_PATH')` — sms package is active
- `$pParamHash['notify_sms'] === 'on'`

Uses `$pParamHash['sms_message']` if provided (strips tags), otherwise composes:
```
Hello {billing_name}, your order #{order_id} from {site_title} has an update
that needs your attention. Please check your email at {email_address}.
```

Passes `$pParamHash['sms_to']` as `$pPhoneOverride` to `SmsBroker::notify()`.
Logs the send to order history with a 📳 prefix.

### Admin template — `bookstore/templates/admin_order_status_history_inc.tpl`

Rendered only when `SMS_PKG_PATH` is defined. Shows:
- "SMS Customer" checkbox (toggles detail div)
- Pre-filled phone from `$gBitOrder->billing.telephone` (fallback: delivery)
- Pre-composed message with billing name, order ID, store name, customer email

---

## Gateway API Reference

| Gateway | Endpoint | Auth | Body |
|---------|----------|------|------|
| Anveo | `https://www.anveo.com/api/v1.asp` | `apikey` form param | `application/x-www-form-urlencoded`; `action=sms`, `from`, `destination`, `message` |
| Twilio | `https://api.twilio.com/2010-04-01/Accounts/{SID}/Messages.json` | HTTP Basic (SID:token) | `application/x-www-form-urlencoded`; `From`, `To`, `Body` |
| Plivo | `https://api.plivo.com/v1/Account/{AUTH_ID}/Message/` | HTTP Basic (AUTH_ID:token) | `application/json`; `src`, `dst`, `text` |
| Telnyx | `https://api.telnyx.com/v2/messages` | Bearer token | `application/json`; `from`, `to`, `text` |

---

## Known Quirks & Gotchas

- **Anveo sandbox:** `SANDBOX_URL` currently points to the live URL —
  `sandbox.anveo.com` appears down. Test sends will hit the real API.

- **`isLive()` suppression:** When `$gBitSystem->isLive()` is false and
  `sms_send_when_not_live` is not set, `SmsBroker::notify()` returns TRUE but
  logs the suppressed message to `error_log`. No SMS is sent — this is silent
  success from the UI's perspective.

- **Gateway instance caching:** `SmsBroker::getInstance()` caches one instance
  per gateway name in `$mInstances`. Config values are read in `__construct()`,
  so changing config mid-request does not affect an already-cached instance.

- **E.164 phone override:** When `$pPhoneOverride` is provided to `notify()`,
  libphonenumber still needs a region hint. It uses the order's billing country
  ISO-2, then delivery, then falls back to `'US'`.

- **Registry built once per request:** `buildRegistry()` sets `$mRegistryBuilt`
  on first call and never re-scans. Dropping a gateway file mid-request will not
  affect a request already in flight. Gateway discovery is per PHP process lifetime.
