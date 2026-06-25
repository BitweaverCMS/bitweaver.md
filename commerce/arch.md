# commerce — Architecture Reference

> Shared reference. Update here when conventions or gotchas are confirmed.
> Upstream repo: https://github.com/bitweaver/commerce

---

## Overview

E-commerce package. Provides product listings, shopping cart, order management,
categories, and manufacturer support. The central class `CommerceProduct`
extends `LibertyMime`, making every product a first-class Liberty content object.

Content type GUID: `BITPRODUCT_CONTENT_TYPE_GUID` (value `'bitproduct'`).

A factory function `bc_get_commerce_product($pParamHash)` returns the
correct concrete product subclass (resolved via `content_type_guid`). Always use
this factory rather than instantiating product classes directly — downstream
packages (products, designer) rely on the polymorphic dispatch it provides.

---

## Key Files

| File | Purpose |
|------|---------|
| `includes/classes/CommerceProduct.php` | Base product class — extends LibertyMime |
| `includes/bitcommerce_start_inc.php` | Package bootstrap; defines TABLE_* constants and global init |
| `includes/bitcommerce_constants_inc.php` | TABLE_PRODUCTS, TABLE_CATEGORIES, TABLE_ORDERS, etc. |
| `templates/` | Smarty templates for product display/edit |

---

## Data Model

### com_products (TABLE_PRODUCTS)

| Column | Notes |
|--------|-------|
| `products_id` | PK — the integer product identifier |
| `content_id` | FK to `liberty_content` — links product into the Liberty content tree |
| `master_categories_id` | FK to `com_categories` |
| `products_type` | FK to `com_product_types` |
| `manufacturers_id` | FK to `com_manufacturers` (nullable) |
| `related_content_id` | FK to `liberty_content` — links product to a related piece of content |

### com_products_description (TABLE_PRODUCTS_DESCRIPTION)

Per-language product name, description, short description. Joined on `products_id` + `language_id`.

### com_categories / com_categories_description (TABLE_CATEGORIES, TABLE_CATEGORIES_DESCRIPTION)

Hierarchical product categories. Products belong to `master_categories_id`.

### com_product_types (TABLE_PRODUCT_TYPES)

Product type (e.g. simple, configurable, downloadable). The concrete PHP handler
class is determined here.

---

## Conventions

- Always bootstrap with `bitcommerce_start_inc.php` before requiring any class
  file under `includes/classes/`. Other packages (designer, products) do this
  explicitly at the top of their own `*_setup_inc.php` files to avoid
  double-init conflicts.
- Use `bc_get_commerce_product($pParamHash)` to load products by `products_id`
  or `content_id`; it returns the correct subclass.
- `CommerceProduct::getNewObjectById()` and `getNewObject()` both delegate to
  `bc_get_commerce_product()`, so the LibertyBase polymorphic factory works.
- Storage subdirectory name is `'products'`; file attachments live under the
  products media path.

---

## Permissions

| Permission | Purpose |
|------------|---------|
| `p_bitcommerce_product_view` | View products |
| `p_bitcommerce_product_create` | Create products |
| `p_bitcommerce_product_update` | Edit products |
| `p_bitcommerce_admin` | Full admin access |

---

## Known Quirks & Gotchas

- **Double-init guard**: requiring any CommerceProduct subclass before calling
  `bitcommerce_start_inc.php` causes TABLE_* constants to be undefined.
  The comment at the top of `designer_setup_inc.php` makes this explicit:
  *"Startup bitcommerce so class includes don't get all out of whack."*

- **`isDeleted()` short-circuit**: `CommerceProduct::load()` clears `mInfo` and
  unsets `mProductsId` if the product is deleted and the user lacks admin
  permission or purchase history. The product object becomes invalid silently —
  always check `isValid()` after loading.
