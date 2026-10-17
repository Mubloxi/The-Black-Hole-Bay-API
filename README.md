# The Black Hole Bay

A BitTorrent search and media-discovery platform with a public JSON API, web
front-end, and Electron desktop client.

The site indexes torrents from search auto-indexing, a scheduled watchlist
scraper, and a DHT crawler. Indexed torrent records and the admin watchlist are
stored in PostgreSQL; the legacy `indexed_torrents.json` and `watchlist.json`
files are retained as backups after migration. The torrent snapshot is imported
when the PostgreSQL index is empty, and the watchlist is imported when its
PostgreSQL entry does not yet exist.

## Layout

| Path | Purpose |
|------|---------|
| `server.js` | HTTP server, routing, search, and all API endpoints |
| `env.js` | `.env` loader; `get()` / `require()` accessors |
| `config.js` | Non-secret defaults, populated from `env.js` |
| `db.js` | PostgreSQL access for users, uploads, ratings, stats, indexed torrents, watchlist, and admin notes |
| `releases.js` | Desktop release store (metadata + installer files) |
| `dht_crawler.mjs` | Legacy standalone crawler; the server now manages DHT directly |
| `llms.txt` / `llms-full.txt` | Site maps for AI assistants |
| `docs/` | Documentation and API reference, served at `/docs` and `/api` |
| `legacy/` | Archived pre-Postgres JSON snapshots. **Nothing reads these.** |
| `bhb-desktop/` | Electron desktop client (separate app, talks to the API) |

## Running

```bash
npm install
cp .env.example .env    # then fill it in
npm start
```

The server prints the port and where that value came from on boot, e.g.:

```
The Black Hole Bay listening on http://localhost:3002  (port source: .env)
```

### Ports

| Port | Purpose |
|------|---------|
| `3002` | The website and API (set `PORT` in `.env`) |
| `6881` | BitTorrent DHT listener (set `DHT_PORT` in `.env`; controlled in the admin panel) |
| `19872` | Desktop client only: local image proxy on `127.0.0.1` |

If the server logs a port you did not expect, a `PORT` value in your shell is
taking precedence over `.env`. Clear it with `Remove-Item Env:PORT` (PowerShell)
or `unset PORT` (bash). Real environment variables always win over `.env` so that
hosting platforms can inject their own port.

## Configuration

All settings come from the environment. See `.env.example` for the full list.
Nothing sensitive is stored in source any more.

- `ADMIN_API_KEY` gates every `/api/admin/*` route and is compared in constant time.
  If it is unset the server generates a random key at boot and warns you.
- `TMDB_API_KEY` / `RAWG_API_KEY` enable film and game metadata.
- `JACKETT_URL` / `JACKETT_API_KEY` enable a Torznab indexer.
- `DHT_PORT` selects the UDP port used by the server's integrated DHT listener.
- `EXTRA_FEEDS` adds arbitrary RSS/Atom feeds, e.g.
  `EXTRA_FEEDS="MyIndex|https://example.org/rss|my_"`.
- `PG_USER`, `PG_HOST`, `PG_DATABASE`, `PG_PASSWORD`, and `PG_PORT` configure
  PostgreSQL. Startup imports the legacy torrent snapshot if the index table is
  empty and migrates the watchlist if it has not yet been stored in PostgreSQL;
  the source JSON files are not deleted.

## Indexers

`INDEXERS` in `server.js` is the single source of truth. Live search and the
background scraper both consume it, so adding a source wires it into both at
once. Current adapters include FitGirl RSS and the SubsPlease JSON API, alongside
the existing public search/RSS providers and configurable `EXTRA_FEEDS`. Every
indexer is isolated — one being down or rate limited never fails a search, it
just contributes nothing.

# API Reference

Full endpoint documentation with parameters, responses, and code examples

## Getting Started

Everything you need to start using the API.

### Base URL

All API requests should be made to:


```text
https://theblackholebay.lol
```


> **Note:** All endpoints are prefixed with `/api`. Example: `https://theblackholebay.lol/api/search?q=dune`

### Authentication

No authentication is required for public search and metadata endpoints.

| Header | Required | Description |
| --- | --- | --- |
| Authorization | No | Not required for public endpoints |
| x-session-token | Yes | Required for User endpoints (Upload, Rate, Profile Update) |

> **Note:** All third-party API keys (TMDB, RAWG) are kept server-side and are never exposed in client responses.

### Rate Limiting

The API uses IP-based rate limiting.

| Setting | Value |
| --- | --- |
| Requests | 120 / minute / IP |
| Auth Limit | 5 / minute / IP (Login/Register) |
| Enforcement | Automatic |
| Exceeded | `429 Too Many Requests` |

### Response Format

All responses are JSON with the following headers:

| Header | Value |
| --- | --- |
| Content-Type | application/json; charset=utf-8 |
| Cache-Control | no-store |

## Torrent API

Search torrents, get metadata, list files, and track downloads.

### `GET /api/search`

Queries configured torrent indexers concurrently and merges their results with the local index, including DHT and user uploads. Results are filtered and ranked for relevance before seed count. Season queries such as `the expanse S01` search the show title and return matching-season releases, including season packs. Non-empty searches return at most 500 results; `totalResults` and `truncated` indicate when more matches exist. Empty queries return up to 100 recent local results (overall or in the requested category) without waiting on live indexers.

| Parameter | Required | Default | Description |
| --- | --- | --- | --- |
| q | No | "" | Search query. Empty = browse up to 100 recent local uploads |
| category | No | 0 | Category code (see below) |

Category codes: `0`=All, `100`=Audio, `200`=Video, `300`=Apps, `400`=Games, `500`=Adult, `600`=Other


```bash
curl "https://theblackholebay.lol/api/search?q=breaking+bad&category=200"
```


```json
{
  "query": "breaking bad",
  "totalResults": 1,
  "truncated": false,
  "source": "remote_api+local+multi_indexer",
  "results": [{
    "id": "4034523",
    "name": "Breaking.Bad.S01E01.1080p.BluRay.x265",
    "info_hash": "abc123def456...",
    "magnet": "magnet:?xt=urn:btih:abc123def456...",
    "seeders": 245,
    "leechers": 12,
    "size": 1073741824,
    "sizeLabel": "1.00 GiB",
    "category": 200,
    "added": 1700000000,
    "username": "uploader123",
    "status": "trusted",
    "source": "TPB"
  }]
}
```


| Field | Type | Description |
| --- | --- | --- |
| id | string | Unique torrent ID |
| name | string | Torrent title |
| info_hash | string | BitTorrent info hash |
| magnet | string | Full magnet URI with trackers |
| seeders | number | Active seeders |
| leechers | number | Active leechers |
| size | number | Size in bytes |
| sizeLabel | string | Human-readable size |
| added | number | Unix timestamp |
| status | string | Uploader status (trusted/vip) |
| source | string | Origin, such as TPBMirror, 1337x, YTS, DHT, or User_Upload |

> **Fallback:** If the live search fails, the endpoint returns matching results from the local index with response `source: "local_fallback"`. This fallback is also capped at 500 results; check `truncated` to see whether additional matches were omitted.

### `GET /api/torrent/{id}`

Returns full metadata for a single torrent.


```bash
curl "https://theblackholebay.lol/api/torrent/4034523"
```


```json
{ "torrent": { "id":"4034523", "name":"...", "magnet":"magnet:?...", "seeders":245, "sizeLabel":"1.00 GiB" } }
```


### `GET /api/files/{id}`

Returns the list of files inside a torrent. For user-uploaded torrents, this is fetched dynamically via WebTorrent.


```json
{
  "files": [
    { "name": "video.mkv", "size": 1073741824, "sizeLabel": "1.00 GiB" },
    { "name": "subs.srt", "size": 45200, "sizeLabel": "44.14 KiB" }
  ]
}
```


### `GET /api/track/download`

Increments the download counter. Call this when a user clicks a magnet link.


```json
{ "ok": true }
```


## User API

Register, login, upload torrents, and rate users.

### `POST /api/auth/register`

Register a new user account. Returns a session token. `acceptToS: true` is required - accounts are only created when the Terms of Service and Privacy Policy are accepted.


```json
// Request Body
{
  "username": "newuser",
  "password": "supersecretpassword",
  "acceptToS": true
}

// Response 200
{
  "success": true,
  "token": "SESSION_TOKEN_HERE",
  "user": { "id": "...", "username": "newuser", "rank": 1, "xp": 0 }
}
```


### `POST /api/auth/login`

Login to an existing account. Returns a session token.


```json
// Request Body
{
  "username": "existinguser",
  "password": "supersecretpassword"
}

// Response 200
{
  "success": true,
  "token": "SESSION_TOKEN_HERE",
  "user": { "id": "...", "username": "existinguser", "rank": 5, "xp": 1200 }
}
```


### `GET /api/user/me`

Returns the currently logged-in user's private profile data. Requires `x-session-token` header.


```bash
curl -H "x-session-token: YOUR_TOKEN" "https://theblackholebay.lol/api/user/me"
```


### `GET /api/user/profile/{username}`

Returns public profile data, uploaded torrents, and rating info for a specific user.


```json
// Response 200
{
  "user": {
    "id": "abc123",
    "username": "Cybotage",
    "avatarPath": "/uploads/avatars/abc123.jpg",
    "bio": "Official uploader",
    "rank": 10,
    "badges": ["5 Uploads", "10 Uploads"]
  },
  "torrents": [ { "id": "idx_123", "name": "Movie.mkv" } ],
  "avgRating": "4.8"
}
```


### `POST /api/user/upload`

Submit a new torrent to the index. Auto-approved. Requires `x-session-token` header.


```json
// Request Body
{
  "name": "My.Custom.Torrent.1080p",
  "info_hash": "9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f1a0b", // 40 hex chars
  "category": 200,
  "description": "Optional description here"
}

// Response 200
{
  "success": true,
  "message": "Torrent uploaded successfully! (+50 XP)"
}
```


> **Note:** Duplicate info hashes are rejected with `409 Conflict`.

### `POST /api/user/rate`

Rate another user (1-5 stars). You can rate a user only once per IP. Requires `x-session-token` header.


```json
// Request Body
{
  "toUserId": "abc123",
  "score": 5
}

// Response 200
{
  "success": true,
  "newRank": 9
}
```


## Movies & TV

Movie and TV metadata powered by TMDB.

### `GET /api/trending`

Trending movies this week.


```json
{
  "results": [{
    "id": 872585,
    "title": "Oppenheimer",
    "overview": "...",
    "poster_path": "/abc.jpg",
    "backdrop_path": "/def.jpg",
    "vote_average": 8.4,
    "release_date": "2024-01-15",
    "original_language": "en"
  }]
}
```


### `GET /api/trending_tv`

Trending TV shows this week. Uses `name` instead of `title` and `first_air_date` instead of `release_date`.

### `GET /api/upcoming?page=1`

Movies currently in theaters. Posters cached locally.

| Parameter | Default | Description |
| --- | --- | --- |
| page | 1 | Page number (20 per page) |

> **Note:** `poster_path` returns a local path: `/upcoming/images/{id}.jpg` - use directly as `https://theblackholebay.lol/upcoming/images/{id}.jpg`. Movies older than 90 days are filtered out.

### `GET /api/tv_airing?page=1`

Currently popular TV shows, filtered to the last 5 years.

| Parameter | Default | Description |
| --- | --- | --- |
| page | 1 | Page number |

### `GET /api/movies?list={list}&page={n}`

Cached movie lists. `list`: `now_playing`, `popular`, `top_rated`, `upcoming`. Poster art is downloaded once and served locally from `/upcoming/images/`.

### `GET /api/tv?list={list}&page={n}`

Cached TV lists. `list`: `popular`, `top_rated`, `on_the_air`, `airing_today`.

### `GET /api/discover?type={movie|tv}&genre={id}&page={n}`

Browse movies or TV by TMDB genre id (cached 6 hours). Without a genre id this returns the popular list.

### `GET /api/tmdb/search?type={movie|tv}&q={query}&page={n}`

Proxied movie/TV search - the TMDB key never leaves the server. Results are cached for 30 minutes.

### `GET /api/tv/{id}/season/{n}`

Episode list for a TV season (cached 24 hours).

### `POST /api/suggestions`

Submit a feature suggestion. Body: `{ title, category, description }`. Rate limited (5/min per IP), 20KB body cap, honeypot field.

### `POST /api/applications`

Submit a staff application from `/apply`. Body: `{ role, username, contact, timezone, availability, skills, experience, motivation, links }`. Rate limited (3/min per IP).

### `GET /api/music/artwork?q={artist title}`

Album artwork + artist lookup for the desktop music player (keyless iTunes-search proxy, 24h server-side cache, never errors - returns `{ artwork: null }` on miss).

### `GET /api/music/import?url={playlist link}`

Import a YouTube playlist or a public Spotify playlist. The server parses the embedded page data and returns the track list so clients can match each track to the site's audio index.

### `GET /api/movie/{id}`

Full movie details including cast, runtime, genres, and IMDb link.


```json
{
  "id": 872585,
  "title": "Oppenheimer",
  "runtime": 180,
  "imdb_id": "tt15398776",
  "genres": [{ "name": "Drama" }],
  "credits": {
    "cast": [{ "name": "Cillian Murphy", "character": "Oppenheimer" }]
  }
}
```


### `GET /api/tv/{id}`

Full TV show details including cast, seasons, and episodes.


```json
{
  "name": "Show Name",
  "number_of_seasons": 3,
  "number_of_episodes": 24,
  "episode_run_time": [55],
  "status": "Returning Series"
}
```


## Games

Game metadata powered by RAWG.

### `GET /api/trending_games`

Recently released games from the last 6 months, sorted by release date. Filtered to modern platforms (PC, PS4, PS5, Xbox One, Xbox Series, Switch). Up to 24 results.


```json
{
  "results": [{
    "name": "Black Myth: Wukong",
    "background_image": "https://media.rawg.io/media/games/xxx.jpg",
    "released": "2024-08-20",
    "rating": "85",
    "genres": "Action, RPG",
    "platforms": "PC, PlayStation 5"
  }]
}
```


| Field | Type | Description |
| --- | --- | --- |
| name | string | Game name |
| background_image | string | Direct image URL - use as-is |
| released | string | Release date (YYYY-MM-DD) |
| rating | string? | Metacritic score or RAWG rating |
| genres | string | Comma-separated genre names |
| platforms | string | Comma-separated platform names |

## Desktop App Updates

Release feed and auto-update support for the BlackHole Bay desktop app. Releases are published from the admin panel under **App Updates**. These endpoints are public and stay reachable even while the site is in maintenance mode.

### `GET /api/update`

Checks whether a newer build than the caller's is published. This is the endpoint the desktop app calls on launch to decide whether to prompt for an update.


```bash
curl "https://theblackholebay.lol/api/update?current=1.0.0&channel=stable"
```


| Parameter | Required | Description |
| --- | --- | --- |
| current | No | Version the client is currently running. Omit to just fetch the latest release. |
| channel | No | `stable` (default) or `beta`. Beta clients also receive stable builds; stable clients never see betas. |


```json
{
  "updateAvailable": true,
  "current": "1.0.0",
  "latest": {
    "id": "6b6e95cf579526b6",
    "version": "1.1.0",
    "name": "Stability & Performance Patch",
    "notes": [
      "Fixed stalled downloads on resume",
      "Improved streaming buffer handling"
    ],
    "channel": "stable",
    "size": 2097152,
    "sha256": "6ea73b45c3b229e3...",
    "publishedAt": 1790594645585,
    "fileName": "BlackHoleBaySetup1.1.0.exe",
    "available": true,
    "downloadUrl": "/api/releases/6b6e95cf579526b6/download"
  }
}
```


| Field | Type | Description |
| --- | --- | --- |
| updateAvailable | boolean | True when latest is newer than current. Always `false` when current is omitted. |
| current | string | Echo of the supplied current version. |
| latest | object? | The newest available release, or `null` if nothing is published. |

> **Always HTTP 200.** This endpoint never returns an error status for "no update" - check the `updateAvailable` field instead.

### `GET /api/releases`

Returns every published release, newest first, plus the current stable build. Powers the version history list on the download page.


```bash
curl "https://theblackholebay.lol/api/releases"
```


```json
{
  "releases": [
    {
      "id": "6b6e95cf579526b6",
      "version": "1.1.0",
      "name": "Stability & Performance Patch",
      "notes": ["Fixed stalled downloads on resume"],
      "channel": "stable",
      "size": 2097152,
      "sha256": "6ea73b45c3b229e3...",
      "publishedAt": 1790594645585,
      "fileName": "BlackHoleBaySetup1.1.0.exe",
      "available": true,
      "downloadUrl": "/api/releases/6b6e95cf579526b6/download"
    }
  ],
  "latest": { "version": "1.1.0", "...": "..." }
}
```


> **Note:** Beta releases appear in `releases` for every client. Only `latest` respects the channel split, since this endpoint is for display.

### `GET /api/releases/{id}/download`

Streams the installer binary for a release. Supports `Range` requests, so interrupted downloads resume instead of restarting.


```bash
# Full download
curl -L "https://theblackholebay.lol/api/releases/6b6e95cf579526b6/download" -o app.exe

# Resume an interrupted download
curl -L -C - "https://theblackholebay.lol/api/releases/6b6e95cf579526b6/download" -o app.exe
```


| Header | Description |
| --- | --- |
| Content-Disposition | Forces a save using the original installer filename. |
| Accept-Ranges | Always `bytes`. |
| X-Release-Version | Version of the file being served. |

> **Status codes:** `200` full body, `206` partial (Range), `404` unknown id, `410` the record exists but the binary is gone from disk.

> **Verify the checksum.** Each release carries a `sha256` computed server-side while the file was uploaded. Compare it against your downloaded file before executing the installer.

### `GET /api/desktop-version`

Simple latest-version lookup kept for older clients. Prefer `/api/update` for new integrations, since it reports whether an update is actually needed.


```json
{
  "version": "1.1.0",
  "name": "Stability & Performance Patch",
  "url": "https://theblackholebay.lol/api/releases/6b6e95cf579526b6/download",
  "notes": ["Fixed stalled downloads on resume"],
  "sha256": "6ea73b45c3b229e3...",
  "size": 2097152
}
```


> **Note:** `version` is `null` when no release has been published yet.

## Client Integration

Minimal update-check flow for the desktop app. Poll on launch, compare the returned version, then download and verify.


```javascript
const crypto = require('crypto');

async function checkForUpdate(currentVersion, channel = 'stable') {
  const res = await fetch(
    'https://theblackholebay.lol/api/update?current=' +
      encodeURIComponent(currentVersion) + '&channel=' + channel
  );
  const data = await res.json();   // always HTTP 200

  if (!data.updateAvailable || !data.latest) return { update: false };

  return {
    update: true,
    version: data.latest.version,
    notes: data.latest.notes,       // show these in the update prompt
    url: 'https://theblackholebay.lol' + data.latest.downloadUrl,
    sha256: data.latest.sha256,     // verify after download
    size: data.latest.size
  };
}
```


> **Verify before running.** Compute the SHA-256 of the downloaded file and compare it to `latest.sha256` before launching the installer. If they differ, discard the file and report a failed update.

## Image URLs

| Source | Format |
| --- | --- |
| TMDB | `https://image.tmdb.org/t/p/{size}{path}` - sizes: w342, w500, w780, original |
| Local cache | `https://theblackholebay.lol/upcoming/images/{id}.jpg` |
| RAWG | Use `background_image` URL directly |
| User Avatars | `https://theblackholebay.lol/uploads/avatars/{userId}.png` |

## Errors

| Status | Meaning | Response |
| --- | --- | --- |
| 400 | Bad Request | `{ "error": "Invalid request" }` |
| 401 | Unauthorized | `{ "error": "Unauthorized" }` |
| 403 | IP blocked | `Forbidden: IP banned.` |
| 404 | Not found | `{ "error": "Failed." }` |
| 409 | Conflict | `{ "error": "This torrent hash has already been uploaded." }` |
| 429 | Rate limit exceeded | `Too Many Requests.` |
| 500 | Server error | `{ "error": "Internal Server Error" }` |

## Code Examples

### JavaScript


```javascript
const API = "https://theblackholebay.lol";

// Search torrents
const res = await fetch(`${API}/api/search?q=oppenheimer&category=200`);
const data = await res.json();

data.results.forEach(t => {
  console.log(`${t.name} | ${t.sizeLabel} | S:${t.seeders} L:${t.leechers}`);
});

// Login and upload
const loginRes = await fetch(`${API}/api/auth/login`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ username: 'user', password: 'pass' })
});
const { token } = await loginRes.json();

await fetch(`${API}/api/user/upload`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'x-session-token': token },
  body: JSON.stringify({ name: 'My.Torrent', info_hash: 'abc123...', category: 200 })
});
```


### Python


```python
import requests

API = "https://theblackholebay.lol"

response = requests.get(f"{API}/api/search", params={
    "q": "breaking bad", "category": 200, "page": 0
}, timeout=10)

for torrent in response.json()["results"]:
    print(f"{torrent['name']} | {torrent['sizeLabel']}")
```


### cURL


```bash
# Search
curl "https://theblackholebay.lol/api/search?q=dune&category=200"

# Trending movies
curl "https://theblackholebay.lol/api/trending"

# Trending games
curl "https://theblackholebay.lol/api/trending_games"

# Movie details
curl "https://theblackholebay.lol/api/movie/872585"
```


### Minimal API Client


```javascript
const API = "https://theblackholebay.lol";

const client = {
  search: (q, cat = 0) =>
    fetch(`${API}/api/search?q=${q}&category=${cat}`).then(r => r.json()),

  torrent: (id) =>
    fetch(`${API}/api/torrent/${id}`).then(r => r.json()),

  files: (id) =>
    fetch(`${API}/api/files/${id}`).then(r => r.json()),

  trendingMovies: () =>
    fetch(`${API}/api/trending`).then(r => r.json()),

  trendingTV: () =>
    fetch(`${API}/api/trending_tv`).then(r => r.json()),

  trendingGames: () =>
    fetch(`${API}/api/trending_games`).then(r => r.json()),

  movie: (id) =>
    fetch(`${API}/api/movie/${id}`).then(r => r.json()),

  tv: (id) =>
    fetch(`${API}/api/tv/${id}`).then(r => r.json())
};
```


## Best Practices

### Caching

API responses use `Cache-Control: no-store`. For frequently-requested data (like trending movies), implement your own short-lived cache (30-60 seconds) to avoid hitting rate limits.

### Error Handling


```javascript
switch (response.status) {
  case 200: break;           // Success
  case 400: break;           // Bad request
  case 401: break;           // Unauthorized
  case 403: break;           // IP blocked
  case 404: break;           // Not found
  case 409: break;           // Conflict (duplicate)
  case 429: break;           // Back off and retry
  case 500: break;           // Temporary failure
}
```


### Security

Do not expose provider API keys in frontend applications. All keys are kept server-side. The recommended architecture is:


```text
Frontend ? HTTPS ? Your App ? Server-side request ? The Black Hole Bay API
```


## Desktop auto-update

1. The client calls `GET /api/update?current=<version>&channel=<stable|beta>`.
2. If newer, it shows the release name, size, and patch notes.
3. It streams the installer to a temp file, hashing as it writes.
4. The digest is compared against the server's published SHA-256. On mismatch the
   file is deleted and the update is abandoned.
5. On success the installer is launched and the app exits.

Publish builds from **Admin → App Updates**. Installers stream to disk (2 GB cap,
extension allowlist) rather than being buffered in memory, and metadata is written
atomically so a crash cannot orphan an uploaded file.

> **Deploy note:** `releases.json` and `downloads/releases/` must live on a
> persistent volume. On ephemeral filesystems every release disappears on redeploy.

## Security notes

- Search results are safe by default: all renderer HTML goes through `esc()`.
- The content filter is anchored on word boundaries so it does not block
  legitimate searches such as "child play", "minor league", or "CPython".
- `/llms.txt` and `/docs` are public by design; `/api/admin/*` is not.
