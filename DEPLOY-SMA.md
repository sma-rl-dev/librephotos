# LibrePhotos - Deployment, Smoke, and Mutation Notes

Baseline: LibrePhotos 1.1.0 @ `c8ac83c51974b70ef52cce57d0985ae47f8f4bd3`
(frontend vendored in-monorepo at `apps/frontend/`).

## Quick Start

```bash
./tester-env deploy    # Build unified image, start app+db, wait for readiness
./tester-env seed      # Deterministic data: admin + 12 photos + 2 albums
./tester-env verify    # Assert app responds, login works, seed present
./tester-env reset     # Clean slate (containers + volumes + per-run data dir)
```

- URL: `http://localhost:8096/`
- Credentials: `admin` / `admin123`

## CLI Commands

| Command  | Description |
|----------|-------------|
| `deploy` | Build image, start app+db, wait for `/api/healthz/`, verify root HTML + admin login |
| `seed`   | Deterministic admin + 12 photos + 2 user albums; blocks until scan completes |
| `verify` | Exits nonzero unless app root responds, admin login works, photo count >= 12, both albums exist |
| `reset`  | `compose down -v` + remove per-run data dir (root-owned files removed via short-lived container); model cache preserved |
| `stop`   | Stop containers (data preserved) |
| `logs`   | Tail app+db logs |
| `status` | Show running state and URL |

Options:
- `--port <port>` host port (default `8096`, container port `8001`).
- `--run-id <id>` isolates compose project and data dir (default `default`).
- `IMAGE_TAG` env overrides the app image name/tag (scripts/rl-env
  content-addressed tag contract; default `tester-env-librephotos:dev`).

Compose project naming: `tester-env-librephotos` (suffix `-${RUN_ID}` when
`--run-id` is set). Per-run data: `.tester-env-data/<run-id>/{db,media,logs,scan}`.
Shared across runs: `.tester-env-data/cache/` (CLIP + ML model weights, ~408MB;
reset does not re-download). Each run gets a deterministic `10.x.x.0/24`
Docker subnet derived from `RUN_ID` (host default address pools are exhausted).

## Unified Image Decision

Single unified image (no separate frontend/proxy containers): Django serves the
React frontend via WhiteNoise on `:8001` (`SERVE_FRONTEND=true`). Build context
is the monorepo root with `deploy/docker/unified/Dockerfile`. Postgres runs as
a pinned companion service. ML feature flags (video/faces/captions/geocoding/
scenes) are off so scans skip multi-GB model downloads; CLIP embeddings are
not flag-gated and download once into the shared cache volume.

Pinned digests:
- Base image `reallibrephotos/librephotos-base:dev`
  `sha256:fd306aa32bacd6643b8905382cdbe8dd0f02395fa07059c8884f0bc884ea0f64`
- DB image `pgautoupgrade/pgautoupgrade:17-bookworm`
  `sha256:43a613e92d75ecef171311211a31c23d3fc01c64be7f72f660374429ec9316be`

## Deterministic Seed Inventory

12 photos = 4 EXIF dates x 3 distinct solid colors each (400x300 JPEG,
Pillow-generated in-container with EXIF Make/Model/DateTimeOriginal, drawn
color-name + date labels). Filenames `<date>_<color>.jpg`.

| Date       | Colors                     |
|------------|----------------------------|
| 2025-11-02 | crimson, amber, teal       |
| 2026-01-17 | cobalt, emerald, lavender  |
| 2026-03-09 | marigold, cerulean, coral  |
| 2026-05-24 | olive, plum, slate         |

User albums (owned by admin):
- `Garden Favorites` - 4 photos: crimson, coral, olive, plum (cover: crimson)
- `Winter Trip` - 3 photos: cobalt, emerald, lavender (cover: cobalt)

Seed is idempotent (wipes `Photo` rows, rescans converges to exactly 12) and
blocks until the async django-q2 scan exposes all 12 photos via the API.
See `SEED.md` for the full inventory, API calls, and measured cycle times
(full reset->verify cycle ~63s).

## Browser Smoke Notes

- Login form at `http://localhost:8096/` with `admin` / `admin123`; after
  login the frontend sets a `jwt` cookie.
- Library scan workflow: Settings show `scan_directory=/data`; photos appear
  in the timeline grouped by the 4 EXIF capture dates.
- Media/thumbnail URLs (e.g. `/media/square_thumbnails/<hash>.webp`) are
  authenticated with the **`jwt` cookie**, NOT the `Authorization: Bearer`
  header - cookie-based access is required for browser-visible assertions.
- Places/location views stay empty (no GPS in seed data; recorded limitation).

## Debugging

- `./tester-env logs` tails app+db.
- API auth for scripts: `POST /api/auth/token/obtain/` with admin credentials
  returns a JWT (`{"access": ...}`) for `Authorization: Bearer <token>`.
