# 🕳️ The Black Hole Bay API

> **A fast, public REST API for torrent-index search, movie & TV metadata, and game discovery.**

<p align="center">
  <img src="https://img.shields.io/badge/API-REST-ff6b35?style=for-the-badge" alt="REST API">
  <img src="https://img.shields.io/badge/Auth-None-2ea44f?style=for-the-badge" alt="No authentication">
  <img src="https://img.shields.io/badge/Rate%20Limit-120%2Fmin-0969da?style=for-the-badge" alt="120 requests per minute">
  <img src="https://img.shields.io/badge/Format-JSON-8b5cf6?style=for-the-badge" alt="JSON">
</p>

<p align="center">
  <strong>No signup · No API key · No authentication</strong><br>
  Built for developers who want simple endpoints and predictable JSON responses.
</p>

---

## ✨ What is it?

The Black Hole Bay API is a lightweight REST API that brings several types of media discovery into one interface:

- Torrent search and metadata
- Movie discovery and details
- TV discovery and details
- Game discovery and release data
- Torrent file listings
- Download-event tracking
- Cached media images
- IP-based rate limiting
- JSON responses designed for web apps, bots, dashboards, and clients

### Base URL

```text
https://theblackholebay.lol
```

---

## ⚡ Quick Start

### Search

```bash
curl "https://theblackholebay.lol/api/search?q=breaking+bad&category=200"
```

### Trending movies

```bash
curl "https://theblackholebay.lol/api/trending"
```

### Trending games

```bash
curl "https://theblackholebay.lol/api/trending_games"
```

### Movie details

```bash
curl "https://theblackholebay.lol/api/movie/872585"
```

Every API endpoint returns JSON.

---

## 🧭 API Architecture

```text
                        ┌─────────────────────┐
                        │   Client / App      │
                        │ Web · Mobile · Bot   │
                        └──────────┬──────────┘
                                   │
                                   ▼
                        ┌─────────────────────┐
                        │  The Black Hole Bay │
                        │      REST API       │
                        └──────────┬──────────┘
                                   │
                 ┌─────────────────┼─────────────────┐
                 │                 │                 │
                 ▼                 ▼                 ▼
          ┌────────────┐   ┌────────────┐   ┌────────────┐
          │   Torrent  │   │ TMDB Data  │   │ RAWG Data  │
          │   Sources  │   │ Movies/TV  │   │   Games    │
          └────────────┘   └────────────┘   └────────────┘
```

The API normalizes supported data into simple JSON responses so clients do not need to implement each provider separately.

---

# 📚 Endpoint Reference

| Method | Endpoint | Purpose |
|:---:|---|---|
| `GET` | `/api/search` | Search indexed torrents |
| `GET` | `/api/torrent/{id}` | Get torrent metadata |
| `GET` | `/api/files/{id}` | List torrent files |
| `GET` | `/api/track/download` | Record a download event |
| `GET` | `/api/trending` | Trending movies |
| `GET` | `/api/trending_tv` | Trending TV shows |
| `GET` | `/api/upcoming?page=` | Current movie listings |
| `GET` | `/api/tv_airing?page=` | Popular TV shows |
| `GET` | `/api/movie/{id}` | Movie details |
| `GET` | `/api/tv/{id}` | TV details |
| `GET` | `/api/trending_games` | Recent game releases |

---

# 🧲 Torrent API

## Search Torrents

```http
GET /api/search
```

Searches configured torrent indexes and returns merged, deduplicated results.

### Query Parameters

| Parameter | Required | Default | Description |
|---|:---:|---:|---|
| `q` | No | `""` | Search query |
| `category` | No | `0` | Torrent category |
| `page` | No | `0` | Page number |

### Categories

| Code | Category |
|---:|---|
| `0` | All |
| `100` | Audio |
| `200` | Video |
| `300` | Applications |
| `400` | Games |
| `500` | Adult |
| `600` | Other |

### Example

```bash
curl "https://theblackholebay.lol/api/search?q=breaking+bad&category=200&page=0"
```

### Response

```json
{
  "query": "breaking bad",
  "category": "200",
  "source": "remote_api+local+1337x",
  "results": [
    {
      "id": "4034523",
      "name": "Example.Release.1080p",
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
    }
  ]
}
```

### Result Fields

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Unique torrent ID |
| `name` | `string` | Torrent title |
| `info_hash` | `string` | BitTorrent info hash |
| `magnet` | `string` | Magnet URI |
| `seeders` | `number` | Active seeders |
| `leechers` | `number` | Active leechers |
| `size` | `number` | Size in bytes |
| `sizeLabel` | `string` | Human-readable size |
| `category` | `number` | Category code |
| `added` | `number` | Unix timestamp |
| `username` | `string` | Uploader name |
| `status` | `string` | Uploader status |
| `source` | `string` | Origin of the result |

### Source Fallback

If the primary remote torrent source is unavailable, the API can fall back to its local index and other configured sources.

---

## Torrent Details

```http
GET /api/torrent/{id}
```

Returns metadata for a torrent ID.

```bash
curl "https://theblackholebay.lol/api/torrent/4034523"
```

```json
{
  "torrent": {
    "id": "4034523",
    "name": "Example.Release.1080p",
    "info_hash": "abc123def456...",
    "magnet": "magnet:?xt=urn:btih:abc123def456...",
    "seeders": 245,
    "leechers": 12,
    "size": 1073741824,
    "sizeLabel": "1.00 GiB",
    "category": 200,
    "added": 1700000000,
    "username": "uploader123",
    "status": "trusted"
  }
}
```

---

## Torrent Files

```http
GET /api/files/{id}
```

Returns the files contained in an indexed torrent.

```bash
curl "https://theblackholebay.lol/api/files/4034523"
```

```json
{
  "files": [
    {
      "name": "example.mkv",
      "size": 1073741824,
      "sizeLabel": "1.00 GiB"
    },
    {
      "name": "subtitles.srt",
      "size": 45200,
      "sizeLabel": "44.14 KiB"
    }
  ]
}
```

---

## Track Download

```http
GET /api/track/download
```

Records a download event for analytics.

```json
{
  "ok": true
}
```

---

# 🎬 Movies & TV

Movie and TV metadata is powered by TMDB.

## Trending Movies

```http
GET /api/trending
```

Returns movies currently trending.

```bash
curl "https://theblackholebay.lol/api/trending"
```

Example:

```json
{
  "results": [
    {
      "id": 872585,
      "title": "Example Movie",
      "overview": "Movie overview...",
      "poster_path": "/poster.jpg",
      "backdrop_path": "/backdrop.jpg",
      "vote_average": 8.4,
      "release_date": "2024-01-15",
      "original_language": "en"
    }
  ]
}
```

---

## Trending TV

```http
GET /api/trending_tv
```

Returns trending TV shows.

TV responses use:

- `name` instead of `title`
- `first_air_date` instead of `release_date`

---

## Now Playing Movies

```http
GET /api/upcoming?page={page}
```

Returns current movie listings.

| Parameter | Default | Description |
|---|---:|---|
| `page` | `1` | Page number |
| Results | `20/page` | Results per page |

```bash
curl "https://theblackholebay.lol/api/upcoming?page=1"
```

Poster images are served from the API's local cache.

---

## Popular TV Shows

```http
GET /api/tv_airing?page={page}
```

Returns currently popular TV shows.

```bash
curl "https://theblackholebay.lol/api/tv_airing?page=1"
```

---

## Movie Details

```http
GET /api/movie/{id}
```

Returns detailed movie metadata including cast, runtime, genres, and IMDb information.

```bash
curl "https://theblackholebay.lol/api/movie/872585"
```

Example:

```json
{
  "id": 872585,
  "title": "Example Movie",
  "overview": "...",
  "runtime": 180,
  "release_date": "2024-01-15",
  "vote_average": 8.4,
  "genres": [
    {
      "id": 18,
      "name": "Drama"
    }
  ],
  "imdb_id": "tt0000000",
  "homepage": "https://example.com",
  "status": "Released",
  "original_language": "en",
  "poster_path": "/abc.jpg",
  "backdrop_path": "/def.jpg",
  "credits": {
    "cast": [
      {
        "name": "Example Actor",
        "character": "Example Character",
        "profile_path": "/headshot.jpg",
        "order": 0
      }
    ]
  }
}
```

---

## TV Details

```http
GET /api/tv/{id}
```

Returns detailed TV metadata including cast, seasons, and episodes.

Additional TV fields:

```json
{
  "number_of_seasons": 3,
  "number_of_episodes": 24,
  "episode_run_time": [55],
  "status": "Returning Series",
  "last_air_date": "2024-06-15"
}
```

---

# 🎮 Games

## Trending Games

```http
GET /api/trending_games
```

Returns recently released games from the last six months, sorted by release date.

Game metadata is powered by RAWG.

Supported modern platforms include:

- PC
- PlayStation 4
- PlayStation 5
- Xbox One
- Xbox Series
- Nintendo Switch

```bash
curl "https://theblackholebay.lol/api/trending_games"
```

Example:

```json
{
  "results": [
    {
      "name": "Example Game",
      "background_image": "https://media.rawg.io/media/games/example.jpg",
      "released": "2024-08-20",
      "rating": "85",
      "genres": "Action, RPG",
      "platforms": "PC, PlayStation 5, Xbox Series S/X",
      "price": null,
      "savings": null
    }
  ]
}
```

Up to 24 games can be returned per response.

---

# 🖼️ Images

## TMDB

```text
https://image.tmdb.org/t/p/{size}{path}
```

Available sizes:

```text
w342
w500
w780
original
```

Example:

```text
https://image.tmdb.org/t/p/w500/abc.jpg
```

## Local Posters

```text
https://theblackholebay.lol/upcoming/images/{id}.jpg
```

## RAWG

Game images are returned as complete URLs and can be used directly.

---

# 🛡️ API Behavior

## Response Format

All API endpoints return:

```http
Content-Type: application/json
```

Responses are intentionally lightweight and client-friendly.

## Caching

API responses currently use:

```http
Cache-Control: no-store
```

Clients that repeatedly request the same endpoint should implement their own short-lived caching where appropriate.

## Authentication

No authentication is currently required.

```text
Authorization: not required
API-Key: not required
```

## HTTP Methods

The public API is currently primarily read-oriented:

```text
GET
```

No public write API is documented.

---

# 🚦 Rate Limiting

The API uses IP-based rate limiting.

| Setting | Value |
|---|---|
| Requests | `120 / minute / IP` |
| Enforcement | Automatic |
| Exceeded limit | `429 Too Many Requests` |
| IP cleanup | Every 60 seconds |

### Handling `429`

Applications should use exponential backoff rather than continuously retrying.

Example:

```js
async function fetchWithRetry(url, retries = 3) {
  for (let attempt = 0; attempt <= retries; attempt++) {
    const response = await fetch(url);

    if (response.status !== 429) {
      return response;
    }

    const delay = 1000 * 2 ** attempt;
    await new Promise(resolve => setTimeout(resolve, delay));
  }

  throw new Error("Rate limit exceeded");
}
```

---

# ❌ Errors

| Status | Meaning | Example |
|---:|---|---|
| `403` | IP blocked | `Forbidden: IP banned.` |
| `404` | Resource not found | `{ "error": "Failed." }` |
| `429` | Rate limit exceeded | `Too Many Requests.` |
| `500` | Server error | `{ "error": "Internal Server Error" }` |

### Recommended Client Handling

```js
switch (response.status) {
  case 200:
    // Success
    break;

  case 403:
    // Access blocked
    break;

  case 404:
    // Resource not found
    break;

  case 429:
    // Back off and retry later
    break;

  case 500:
    // Temporary server failure
    break;
}
```

---

# 💻 Code Examples

## JavaScript

```js
const API = "https://theblackholebay.lol";

async function searchTorrents(query, category = 0) {
  const params = new URLSearchParams({
    q: query,
    category: String(category),
    page: "0"
  });

  const response = await fetch(`${API}/api/search?${params}`);

  if (!response.ok) {
    throw new Error(`API error: ${response.status}`);
  }

  return response.json();
}

const data = await searchTorrents("oppenheimer", 200);

for (const torrent of data.results) {
  console.log(
    `${torrent.name} | ${torrent.sizeLabel} | ` +
    `S:${torrent.seeders} L:${torrent.leechers}`
  );
}
```

## Python

```python
import requests

API = "https://theblackholebay.lol"

response = requests.get(
    f"{API}/api/search",
    params={
        "q": "breaking bad",
        "category": 200,
        "page": 0,
    },
    timeout=10,
)

response.raise_for_status()

data = response.json()

for torrent in data["results"]:
    print(
        f"{torrent['name']} | "
        f"{torrent['sizeLabel']} | "
        f"S:{torrent['seeders']} "
        f"L:{torrent['leechers']}"
    )
```

## cURL

```bash
# Search
curl "https://theblackholebay.lol/api/search?q=dune&category=200"

# Torrent details
curl "https://theblackholebay.lol/api/torrent/4034523"

# Torrent files
curl "https://theblackholebay.lol/api/files/4034523"

# Trending movies
curl "https://theblackholebay.lol/api/trending"

# Trending TV
curl "https://theblackholebay.lol/api/trending_tv"

# Trending games
curl "https://theblackholebay.lol/api/trending_games"

# Current movies
curl "https://theblackholebay.lol/api/upcoming?page=1"

# Popular TV
curl "https://theblackholebay.lol/api/tv_airing?page=1"

# Movie details
curl "https://theblackholebay.lol/api/movie/872585"

# TV details
curl "https://theblackholebay.lol/api/tv/67890"
```

---

# 🔌 Example API Client

A minimal client can wrap the API into reusable functions:

```js
const API = "https://theblackholebay.lol";

const client = {
  async search(query, category = 0, page = 0) {
    const params = new URLSearchParams({
      q: query,
      category: category.toString(),
      page: page.toString()
    });

    return fetch(`${API}/api/search?${params}`).then(r => r.json());
  },

  async torrent(id) {
    return fetch(`${API}/api/torrent/${id}`).then(r => r.json());
  },

  async files(id) {
    return fetch(`${API}/api/files/${id}`).then(r => r.json());
  },

  async trendingMovies() {
    return fetch(`${API}/api/trending`).then(r => r.json());
  },

  async trendingTV() {
    return fetch(`${API}/api/trending_tv`).then(r => r.json());
  },

  async trendingGames() {
    return fetch(`${API}/api/trending_games`).then(r => r.json());
  },

  async movie(id) {
    return fetch(`${API}/api/movie/${id}`).then(r => r.json());
  },

  async tv(id) {
    return fetch(`${API}/api/tv/${id}`).then(r => r.json());
  }
};
```

---

# 🏗️ Server Notes

The public API is designed around a simple request → aggregation → JSON response flow.

```text
Request
  ↓
Rate limiter
  ↓
API route
  ↓
Provider / local index
  ↓
Normalize data
  ↓
Return JSON
```

### Recommended Production Setup

```text
Internet
   │
   ▼
Cloudflare / Reverse Proxy
   │
   ▼
Nginx
   │
   ▼
Node.js API
   │
   ├── Local torrent index
   ├── External torrent sources
   ├── TMDB
   └── RAWG
```

For production deployments, keep provider credentials and private configuration server-side and never expose them in browser code.

---

# 📊 API Design Principles

The API aims to keep responses:

- **Predictable** — consistent JSON structures
- **Lightweight** — only useful fields are returned
- **Discoverable** — endpoints are easy to understand
- **Client-friendly** — works with JavaScript, Python, cURL, mobile apps, and other HTTP clients
- **Provider-independent** — clients consume one API instead of integrating multiple services directly

---

# 🔐 Security Notes

Do not expose private provider API keys in frontend applications.

Recommended architecture:

```text
Frontend
   │
   │ HTTPS
   ▼
Your application
   │
   │ Server-side request
   ▼
The Black Hole Bay API
   │
   ├── TMDB
   ├── RAWG
   └── Torrent providers
```

For public deployments, also consider:

- HTTPS only
- Reverse-proxy rate limiting
- Request validation
- Structured server logs
- Provider timeouts
- Error handling and retries
- Health monitoring
- Separate development and production environments
- Keeping secrets in environment variables

---

# 🧪 Testing

Basic endpoint smoke tests can be performed with cURL:

```bash
curl -i "https://theblackholebay.lol/api/trending"
curl -i "https://theblackholebay.lol/api/trending_tv"
curl -i "https://theblackholebay.lol/api/trending_games"
curl -i "https://theblackholebay.lol/api/search?q=test"
```

A healthy endpoint should return a successful HTTP status and JSON response.

---

# 📦 Data Providers

| Provider | Used For |
|---|---|
| **TMDB** | Movies and TV metadata |
| **RAWG** | Game metadata |
| **Configured torrent indexes** | Torrent search data |

This project is not affiliated with TMDB, RAWG, The Pirate Bay, or 1337x.

---

# 🤝 Contributing

Contributions, bug reports, and feature requests are welcome.

1. Fork the repository.
2. Create a feature branch.
3. Make your changes.
4. Test the affected endpoints.
5. Open a pull request.

For API changes, include updated endpoint documentation and example responses.

---

# 📄 License

Add your project's license here.

If this repository is public, adding a license is recommended so contributors and users know how the code may be used.

---

<div align="center">

## 🕳️ The Black Hole Bay API

**Fast REST endpoints · Simple JSON · No API key**

[Report an Issue](https://github.com/yourusername/the-black-hole-bay/issues) ·
[Request a Feature](https://github.com/yourusername/the-black-hole-bay/issues)

</div>
