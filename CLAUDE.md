# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A **TREK plugin** (type `trip-page`) called 找地方 (amap-search) that lets users search Amap (高德地图) POIs from inside a TREK trip and copy/add results into the trip. It is not a standalone app — it only runs when packaged and uploaded into a TREK instance (TREK >=3.4.0, <4.0.0), built on `trek-plugin-sdk`.

## Commands

- `npx trek-plugin-sdk pack` (or `npm run pack`) — builds `plugin.zip` for upload via TREK Admin → Plugins → Upload.
- There is no local dev server, lint, or test suite in this repo — the plugin is only runnable/testable inside a real TREK instance after packing and uploading. `playwright` is a devDependency but there are no Playwright tests currently checked in.

## Architecture

Two files carry all the logic:

- **`server/index.js`** — the plugin backend, defined via `definePlugin({ routes: [...] })` from `trek-plugin-sdk`. All routes are mounted under the plugin's namespace and require `auth: true`. Handlers receive `(req, ctx)` where `ctx` exposes TREK's plugin SDK surface (`ctx.settings`, `ctx.trips`, `ctx.places`, `ctx.log`). Key routes:
  - `GET /key-status` — whether the current user has set their Amap key (client uses this to show/hide onboarding hints).
  - `GET /search` — proxies to Amap's `v3/place/text` Web Service API, paginated, converts each POI's coordinates from GCJ-02 (Amap) to WGS-84 (TREK) via `gcj02ToWgs84`, and keeps the original GCJ-02 coords (`gcj_location`) for building Amap map links.
  - `GET /trip-places` — current trip's places (for marking POIs "already added").
  - `GET /trip-anchor` — first place in the trip with coordinates (previously used for distance sort; currently unused by client per changelog).
  - `GET /trip-city` — 3-tier city inference for the current trip: (1) match trip title against the built-in `CITY_NAMES` list, (2) reverse-geocode (`regeoCity`) the trip's first place with coordinates via Amap, (3) match city names in that place's address text.
  - `POST /add` — writes a searched POI into the trip via `ctx.places.create`, mapping Amap fields to TREK place fields: coordinates (WGS-84) → lat/lng, `type` → `description`, generated Amap map link → `website`, `tel` → `notes`.
  - `GET /trips` — lists trips accessible to the current user.

  The GCJ-02 ↔ WGS-84 conversion (`wgs84ToGcj02` / `gcj02ToWgs84`, the eviltransform algorithm) is central and used in both directions: incoming POI coordinates are converted GCJ-02→WGS-84 for storage/display, while `regeoCity` converts a stored WGS-84 place back to GCJ-02 before querying Amap (which expects GCJ-02 for all its APIs).

- **`client/index.html`** — the entire frontend as one static HTML file (markup + inline `<style>` + inline `<script>`, no build step, no framework). It runs inside a TREK-hosted iframe and receives trip context (`ctx.tripId`) from the host. Key behaviors: search-and-render POI cards, client-side filter (by Amap `type`) and sort (rating/cost/default — distance sort is intentionally disabled, see below), copy-to-clipboard of POI details, "add to trip" with persisted added-state, incremental `loadMore` pagination, skeleton/empty/error states, and city auto-detect UI tied to `/trip-city`.
  - Important constraint noted in the changelog: static HTML in this file does **not** support `${}` template interpolation — that only works inside `<script>` blocks. A past bug (v1.3.22) came from writing `${ICON_SEARCH}` directly in markup.
  - Distance-based sorting/filtering is deliberately not exposed in the UI: the TREK SDK has no "selected place" event to know what the user is anchored to, so an anchor-based distance feature was removed rather than left half-working.

- **`trek-plugin.json`** — the plugin manifest: id, display name, version, TREK version range, declared `permissions` (`db:read:trips`, `db:write:places`, `http:outbound:restapi.amap.com`) and `egress` (only `restapi.amap.com`), and the one user-scoped, secret setting `amap_key` (the user's own Amap Web Service API key — required, never checked into code).

## Working conventions specific to this repo

- Keep the plugin's declared `permissions`/`egress` in `trek-plugin.json` minimal and in sync with what `server/index.js` actually does — the plugin only ever talks to `restapi.amap.com`, and TREK shows these permissions to admins on activation.
- Comments and commit-worthy documentation in this codebase are written in Chinese; match that style when editing `server/index.js` / `client/index.html`.
- `README.md` (Chinese, primary) and `README.en.md` (English) must be kept in sync, including the versioned changelog (`更新日志`) at the bottom of `README.md`, which is the authoritative history of behavior changes — check it before assuming why something is implemented a certain way (e.g. why distance sort is disabled, why certain fields were removed from cards).
- Bump `version` in both `trek-plugin.json` and `package.json` together when releasing, and add a changelog entry in `README.md`.
