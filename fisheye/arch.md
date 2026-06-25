# fisheye — Architecture Reference

> Shared reference. Update here when conventions or gotchas are confirmed.
> Upstream repo: https://github.com/bitweaver/fisheye

---

## Overview

Photo gallery and image management package. Provides image upload, gallery/album
organisation, and image display. Both images and galleries are Liberty content
objects. `FisheyeBase extends LibertyMime`; images and galleries both inherit
from it.

Content type GUIDs: `'fisheyeimage'` (image), `'fisheyegallery'` (gallery).

The `intersite` package extends fisheye with virtual image/gallery classes that
proxy third-party photo sources (Flickr, Facebook, Instagram, Zenfolio, PBase)
through the same `FisheyeImage` / `FisheyeGallery` interface.

---

## Key Files

| File | Purpose |
|------|---------|
| `includes/classes/FisheyeImage.php` | Image content class — `FisheyeImage extends FisheyeBase` |
| `includes/classes/FisheyeGallery.php` | Gallery container — `FisheyeGallery extends FisheyeBase` |
| `includes/classes/FisheyeBase.php` | Common base — `FisheyeBase extends LibertyMime` |
| `templates/` | Smarty templates |

---

## Data Model

### fisheye_image

| Column | Notes |
|--------|-------|
| `image_id` | PK |
| `content_id` | FK to `liberty_content` |

### fisheye_gallery

| Column | Notes |
|--------|-------|
| `gallery_id` | PK |
| `content_id` | FK to `liberty_content` |

Images belong to galleries through a separate join table. `FisheyeGallery::loadImages()`
populates `$mItems` with `FisheyeImage` instances.

---

## Conventions

- Use `FisheyeGallery::lookup($pLookupHash)` to load galleries polymorphically
  (resolves to the correct subclass if intersite is active).
- Use `FisheyeImage::getParentGalleries()` to reverse-look up which gallery an
  image belongs to — used by the designer photo drawer to pre-select the source
  gallery when editing an existing book page.
- Always call through the virtual image/gallery classes from `intersite` when
  the source could be a third-party service.

---

## Permissions

| Permission | Purpose |
|------------|---------|
| `p_fisheye_view` | View images/galleries |
| `p_fisheye_create` | Upload images / create galleries |
| `p_fisheye_update` | Edit images / galleries |
| `p_fisheye_admin` | Full admin access |

---

## intersite extensions (virtual photo sources)

The `intersite` package provides `FisheyeVirtualImage` and `FisheyeVirtualGallery`
which extend the fisheye classes. `FisheyeVirtualImage::getLibertyObject()` routes
by content_id prefix:

| Prefix | Source |
|--------|--------|
| numeric | Local fisheye image (standard lookup) |
| `flickr…` | `FisheyeFlickrImage` |
| `facebook…` | `FisheyeFacebookImage` |
| `instagram…` | `FisheyeInstagramImage` |
| `zenfolio…` | `FisheyeZenfolioImage` |
| `pbase…` | PBase gallery image |

The designer `drawer_photo.php` uses `FisheyeVirtualImage` and `FisheyeVirtualGallery`
throughout so it transparently handles all photo sources.

---

## Known Quirks & Gotchas

- `FisheyeGallery::lookup()` is polymorphic — it queries `liberty_content` to
  determine the actual content_type_guid and returns the right subclass. Do not
  instantiate `FisheyeGallery` directly if the gallery might be a virtual source.

- `FisheyeVirtualImage::load()` is a no-op stub (empty body). The image data
  is loaded lazily by the concrete subclass on first access.
