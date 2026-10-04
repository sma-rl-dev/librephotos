# LibrePhotos tester-env Seed Documentation

Deterministic data population for `./tester-env` in this prospect checkout
(LibrePhotos 1.1.0 @ c8ac83c51974b70ef52cce57d0985ae47f8f4bd3).

## Commands

```bash
./tester-env reset && ./tester-env deploy && ./tester-env seed && ./tester-env verify
```

- **Credentials:** `admin` / `admin123` (created by container entrypoint from
  `ADMIN_*` env, re-asserted by `seed`).
- **Reset** removes containers + volumes and the per-run data dir
  (`.tester-env-data/<run-id>/`) via a short-lived container (app files are
  root-owned on the bind mount). The shared `.tester-env-data/cache/` (CLIP
  model, ML model weights ~408MB) survives reset.
- **Seed** is idempotent and blocks until the scan is complete and the photos
  are visible via the API. Measured: full cycle ~63s (deploy ~40s build-cached,
  seed ~11s including scan wait, verify <1s).
- **Verify** exits nonzero unless: app root responds, admin login works,
  photo count >= 14, both albums exist, the folder album `Library` lists all
  14 photos, and the AA-Twin/ZZ-Twin pair shares one identical EXIF timestamp.

## Seeded State

1. **Admin user** with `scan_directory=/data` (a fresh admin starts without a
   scan directory; scans fail without it).
2. **14 photos** in the library: 12 base photos, 3 per fixed capture date,
   distinct solid colors with drawn labels, plus the AA-Twin/ZZ-Twin
   same-timestamp pair. Generated inside the app container with Pillow
   (400x300 JPEG q85, EXIF Make/Model/DateTime/DateTimeOriginal) into
   `/data/Library` (host: `.tester-env-data/<run-id>/scan/Library/`), so the
   normal recursive scanner exposes one disk-backed folder album named
   `Library` with all 14 photos.

   | Date       | Files                                                      |
   |------------|------------------------------------------------------------|
   | 2025-11-02 | crimson (#DC143C), amber (#FFBF00), teal (#008080)          |
   | 2026-01-17 | cobalt (#0047AB), emerald (#50C878), lavender (#B57EDC), AA-Twin (#FF8C00) + ZZ-Twin (#4B0082) sharing EXIF `2026:01:17 12:00:00` |
   | 2026-03-09 | marigold (#EAA221), cerulean (#2A52BE), coral (#FF7F50)     |
   | 2026-05-24 | olive (#808000), plum (#8E4585), slate (#708090)            |

   Filenames: `<date>_<color>.jpg` (e.g. `2025-11-02_crimson.jpg`), plus
   `2026-01-17_aa-twin.jpg` / `2026-01-17_zz-twin.jpg` (alphabetically
   ordered, visually distinguishable solid colors + labels). Each image
   draws the color name and date as white text with black outline. Camera EXIF:
   Make=LibrePhotos, Model=TesterCam One. All 14 photos have EXIF timestamps
   (4 distinct date groups visible in the UI timeline); the twin pair shares
   one identical DateTimeOriginal (`2026:01:17 12:00:00`) inside the 2026-01-17
   day group, so their on-screen order is decided by the filename tiebreaker
   (`order_by("-exif_timestamp", "main_file__path")` in AlbumDateViewSet).
   The twins belong to no user album (Garden Favorites keeps 4 photos,
   Winter Trip keeps 3).
3. **2 user albums** (owned by admin):
   - `Garden Favorites` — 4 photos: crimson, coral, olive, plum (cover: crimson)
   - `Winter Trip` — 3 photos: cobalt, emerald, lavender (cover: cobalt)

## Seed / Verify API Calls

Auth: `POST /api/auth/token/obtain/` `{"username":"admin","password":"admin123"}`
→ `{"access": ...}` (JWT, used as `Authorization: Bearer <token>`).

Seed:
- `POST /api/scanphotos/` → `{"status": true, "job_id": ...}` (async django-q2
  scan of `/data`; seed polls until done)
- `GET /api/photos/?page_size=1` → `{"count": N}` polled until N == 14
- `GET /api/photos/?page_size=100` → results `id` (UUID pk) + `image_path[0]`
  (filename → UUID mapping for album membership)
- `POST /api/albums/user/` `{"title": "..."}` → 201, response `id` (album UUID)
- `PATCH /api/albums/user/edit/<album_id>/`
  `{"photos": ["<uuid>", ...], "cover_photo": "<uuid>"}` (AlbumUserEditSerializer)

Verify:
- `GET /api/photos/?page_size=1` → count >= 14
- `GET /api/albums/user/?page_size=100` → titles include both albums
- `GET /api/folders/subfolders/` → a folder entry `Library` with
  `photo_count` 14 (disk-backed folder album, admin DATA_ROOT `/data`)
- `GET /api/photos/?page_size=100` → both `2026-01-17_aa-twin.jpg` and
  `2026-01-17_zz-twin.jpg` present with equal `exif_timestamp`

Additional non-CLI maintenance calls used while building the seed:
- `POST /api/deletemissingphotos` (unused in seed; seed wipes `Photo` rows via
  `manage.py shell` instead, then rescans — guarantees exactly 12).

## Media / Thumbnails

- Thumbnails are served by `GET /media/square_thumbnails/<image_hash>.webp`
  (also `thumbnails_big/`), authenticated with the **`jwt` cookie** (set by the
  frontend after login), NOT the Bearer header. Verified: HTTP 200,
  `image/webp` with `Cookie: jwt=<access token>`.

## Determinism Notes / Limitations

- Photo generation, admin/scan-directory assertion, and DB photo wipe run via
  `docker compose exec app python manage.py shell` (Pillow in-container; host
  ImageMagick 6 cannot reliably write EXIF and no exiftool on host).
- Seed wipes existing `Photo` rows before rescanning, so a second `seed` run
  converges to exactly 14 photos (no duplicates) — idempotency proven.
- ML model files (~408MB: im2txt, places365, resnet18, CLIP embeddings) resolve
  under `MEDIA_ROOT/data_models`; they are bind-mounted from the shared cache
  (`${CACHE_DIR}/data_models`) so reset+reseed does not re-download them.
- No videos (FEATURE_VIDEO=false), no faces/geocoding/captions (ML flags off).
- No locations/GPS in seed data; places views stay empty (recorded limitation).
