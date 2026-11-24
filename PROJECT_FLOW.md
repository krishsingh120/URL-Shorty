# 📘 URL Shorty — Complete Project Flow & Logic

> This document walks through **every layer** of the application — from a user clicking "Shorten" all the way down to MongoDB and back — including the key logic decisions and how every file connects.

---

## 📌 Table of Contents

1. [High-Level Flow Diagram](#1-high-level-flow-diagram)
2. [Layer-by-Layer Breakdown](#2-layer-by-layer-breakdown)
3. [Flow 1 — Shortening a URL](#3-flow-1--shortening-a-url)
4. [Flow 2 — Viewing All Short URLs](#4-flow-2--viewing-all-short-urls)
5. [Flow 3 — Redirect via Short URL](#5-flow-3--redirect-via-short-url)
6. [Flow 4 — REST API Usage](#6-flow-4--rest-api-usage)
7. [Core Business Logic Deep Dive](#7-core-business-logic-deep-dive)
8. [Database Schema & Model](#8-database-schema--model)
9. [Caching Strategy](#9-caching-strategy)
10. [Data Flow Summary Table](#10-data-flow-summary-table)
11. [File Responsibility Map](#11-file-responsibility-map)

---

## 1. High-Level Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        USER (Browser)                        │
└────────────┬───────────────────────┬────────────────────────┘
             │                       │
     [Submits URL form]     [Visits /urls/[id]]
             │                       │
             ▼                       ▼
┌────────────────────┐   ┌───────────────────────┐
│  Server Action     │   │  Dynamic Route Page   │
│  ShortenUrlAction  │   │  /urls/[id]/page.tsx  │
└────────┬───────────┘   └──────────┬────────────┘
         │                          │
         ▼                          ▼
┌──────────────────────────────────────────────┐
│            UrlShortenerService               │
│  - shortenUrl()                              │
│  - getUrlByShortUrl()                        │
│  - getAllUrls()                              │
└──────────────────┬───────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────┐
│              UrlRepository                   │
│  - getUrlByOriginalUrl()                     │
│  - getUrlByShortUrl()                        │
│  - createUrl()                               │
│  - getAllUrls()                              │
└──────────────────┬───────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────┐
│               MongoDB (Mongoose)             │
│   Collection: urls                           │
│   Fields: originalUrl, shortUrl, timestamps  │
└──────────────────────────────────────────────┘
```

---

## 2. Layer-by-Layer Breakdown

The project follows a clean **4-layer architecture**:

```
┌──────────────────────────────────────────────────────┐
│  LAYER 4 — Presentation (Pages + API Routes)         │
│  src/app/page.tsx                                    │
│  src/app/urls/page.tsx                               │
│  src/app/urls/[id]/page.tsx                          │
│  src/app/api/shorten/route.ts                        │
│  src/app/api/urls/route.ts                           │
│  src/app/serverActions/ShortenUrlAction.ts           │
├──────────────────────────────────────────────────────┤
│  LAYER 3 — Service / Business Logic                  │
│  src/services/UrlShortenerService.ts                 │
├──────────────────────────────────────────────────────┤
│  LAYER 2 — Repository / Data Access                  │
│  src/repositories/UrlRepository.ts                   │
├──────────────────────────────────────────────────────┤
│  LAYER 1 — Model + Database                          │
│  src/models/Url.ts                                   │
│  src/config/db.ts                                    │
└──────────────────────────────────────────────────────┘
```

> [!IMPORTANT]
> Each layer **only talks to the layer directly below it**.  
> Pages call Services. Services call Repositories. Repositories call the Model. No layer skips another.

---

## 3. Flow 1 — Shortening a URL

### Step-by-Step

```
User types URL ──► Home Page (/) ──► <form action={shortenUrl}>
                                            │
                              [Server Action fires on submit]
                                            │
                                   ShortenUrlAction.ts
                                            │
                            Extract `originalUrl` from FormData
                                            │
                              UrlShortenerService.shortenUrl()
                                            │
                         ┌──────── URL already exists? ────────┐
                         │ YES                                  │ NO
                         │                                      │
                  return existing shortUrl          Generate shortId with shortid()
                                                               │
                                                  Short ID collision in DB? ──YES──► regenerate
                                                               │ NO
                                                               │
                                              Save { originalUrl, shortUrl: "urls/<id>" }
                                                               │
                                                    revalidatePath('/urls')
                                                    (clears cached /urls page)
```

### Key Files Involved

| File | Role |
|---|---|
| `src/app/page.tsx` | Renders the form; passes `shortenUrl` Server Action as the form's `action` |
| `src/app/serverActions/ShortenUrlAction.ts` | Marked `'use server'`; extracts FormData, calls service, revalidates cache |
| `src/services/UrlShortenerService.ts` | Contains the core dedup + collision-safe shortening logic |
| `src/repositories/UrlRepository.ts` | `getUrlByOriginalUrl()`, `getUrlByShortUrl()`, `createUrl()` |
| `src/models/Url.ts` | Mongoose model that maps to the `urls` collection |

### Code Trace

```ts
// 1. page.tsx — form with Server Action
<form method="POST" action={shortenUrl}>
  <input type="text" name="originalUrl" />
  <button type="submit">Shorten</button>
</form>

// 2. ShortenUrlAction.ts
'use server'
const shortenURL = async (formData: FormData) => {
    const originalUrl = formData.get('originalUrl') as string;
    const shortenerService = new UrlShortenerService();
    await shortenerService.shortenUrl(originalUrl);
    revalidatePath('/urls');   // ← invalidates the cached URL list
}

// 3. UrlShortenerService.ts — the core logic
async shortenUrl(originalUrl?: string): Promise<string> {
    if (!originalUrl) return "";

    // Deduplication check
    let url = await this.urlRepository.getUrlByOriginalUrl(originalUrl);
    if (url) return url.shortUrl;   // ← already shortened, return it

    // Collision-safe ID generation
    let shortUrl = shortId();
    url = await this.urlRepository.getUrlByShortUrl(shortUrl);
    while (url) {                   // ← keep trying until unique
        shortUrl = shortId();
        url = await this.urlRepository.getUrlByShortUrl(shortUrl);
    }

    // Persist — shortUrl stored as "urls/<id>"
    await this.urlRepository.createUrl(originalUrl, `urls/${shortUrl}`);
    return shortUrl;
}
```

---

## 4. Flow 2 — Viewing All Short URLs

### Step-by-Step

```
User clicks "View All Shortened URLs"
            │
            ▼
      /urls (page.tsx)
            │
   fetchUrls() — calls GET /api/urls
   (uses Next.js force-cache on first load)
            │
            ▼
   /api/urls/route.ts
            │
   UrlShortenerService.getAllUrls()
            │
   UrlRepository.getAllUrls()  ──► MongoDB .find()
            │
            ▼
   Returns array of { _id, originalUrl, shortUrl }
            │
            ▼
   Rendered as a DaisyUI zebra-striped table
   Each row: originalUrl | clickable shortUrl link
```

### Key Files Involved

| File | Role |
|---|---|
| `src/app/urls/page.tsx` | Async Server Component; fetches from internal API and renders the table |
| `src/app/api/urls/route.ts` | `GET` handler; calls service; sets Cache-Control headers |
| `src/services/UrlShortenerService.ts` | `getAllUrls()` delegates to repository |
| `src/repositories/UrlRepository.ts` | `getAllUrls()` — runs `Url.find().lean()` |

### Code Trace

```ts
// urls/page.tsx — Server Component fetch
async function fetchUrls() {
    const response = await fetch(`${process.env.NEXT_PUBLIC_BASE_URL}/api/urls`, {
        cache: 'force-cache'   // ← Next.js caches this until revalidatePath is called
    });
    return response.json();
}

// api/urls/route.ts
export async function GET() {
    const urls = await fetchUrls();
    const response = NextResponse.json({ urls });
    response.headers.set(
        'Cache-Control',
        'public, max-age=180, s-maxage=180, stale-while-revalidate=59'
    );
    return response;
}
```

> [!NOTE]
> `force-cache` on the fetch + `Cache-Control` headers on the API together ensure the URL list
> is **not re-fetched on every render**. When a new URL is shortened, `revalidatePath('/urls')`
> purges this cache so the next visit shows fresh data.

---

## 5. Flow 3 — Redirect via Short URL

### Step-by-Step

```
User visits  /<shortUrl>  e.g. /urls/abc123
            │
            ▼
  Next.js matches /urls/[id] dynamic route
            │
            ▼
  urlRedirect({ params: { id: 'abc123' } })
            │
  Calls: UrlShortenerService.getUrlByShortUrl("urls/abc123")
            │
  UrlRepository.getUrlByShortUrl("urls/abc123")  ──► MongoDB findOne
            │
        Found?
       /       \
     YES        NO
      │          │
  redirect()  redirect('/404')
  originalUrl
```

### Key Files Involved

| File | Role |
|---|---|
| `src/app/urls/[id]/page.tsx` | Catches the dynamic segment, looks up the original URL, issues server-side redirect |
| `src/services/UrlShortenerService.ts` | `getUrlByShortUrl()` — passes lookup to repository |
| `src/repositories/UrlRepository.ts` | `getUrlByShortUrl()` — `findOne({ shortUrl })` |

### Code Trace

```ts
// urls/[id]/page.tsx
async function fetchOriginalUrl(url: string) {
    const urlService = new UrlShortenerService();
    const response = await urlService.getUrlByShortUrl(url);
    return response?.originalUrl;
}

export default async function urlRedirect({ params }: { params: { id: string } }) {
    // params.id = "abc123" → prefixed to match stored "urls/abc123"
    const original = await fetchOriginalUrl(`urls/${params.id}`);
    if (original) redirect(original);   // ← 307 Temporary Redirect to original URL
    redirect('/404');                   // ← not found
}
```

> [!IMPORTANT]
> Short URLs are stored in MongoDB as `"urls/<id>"` (with the prefix).
> The dynamic route captures only `<id>`, so the prefix is re-added before the DB lookup.

---

## 6. Flow 4 — REST API Usage

The app also exposes two REST endpoints for programmatic access.

### POST `/api/shorten`

```
Client ──► POST /api/shorten
           Body: { "originalUrl": "https://very-long-url.com" }
                │
                ▼
         api/shorten/route.ts
                │
         UrlShortenerService.shortenUrl(originalUrl)
                │
         (same dedup + collision logic as Flow 1)
                │
                ▼
         Response 201: { "shortUrl": "abc123" }
```

### GET `/api/urls`

```
Client ──► GET /api/urls
                │
                ▼
         api/urls/route.ts
                │
         UrlShortenerService.getAllUrls()
                │
                ▼
         Response 200: { "urls": [ { _id, originalUrl, shortUrl, ... } ] }
```

---

## 7. Core Business Logic Deep Dive

### 7.1 Deduplication (Idempotency)

```
shortenUrl("https://example.com")
        │
        ▼
  getUrlByOriginalUrl("https://example.com")
        │
  Found in DB?  ──YES──► return existing shortUrl (no new record)
        │
       NO
        │
        ▼
  Generate new shortId + save
```

**Why this matters:** Submitting the same URL multiple times always returns the **same short link**.  
Prevents duplicate entries and keeps the database clean.

---

### 7.2 Collision-Safe ID Generation

```
shortId()  ──► "abc123"
      │
  Already used in DB?  ──YES──► shortId() again (loop)
      │
     NO
      │
  Save it ✔
```

`shortid` generates 7–14 character alphanumeric IDs. Collisions are extremely rare, but the
`while` loop guarantees uniqueness before writing.

```ts
let shortUrl = shortId();
url = await this.urlRepository.getUrlByShortUrl(shortUrl);
while (url) {
    shortUrl = shortId();
    url = await this.urlRepository.getUrlByShortUrl(shortUrl);
}
```

---

### 7.3 `shortUrl` Storage Format

Short URLs are stored in MongoDB as **`"urls/<id>"`**, not just the bare ID.

| Stored value | Example |
|---|---|
| `shortUrl` in DB | `"urls/abc123"` |
| URL user visits | `/urls/abc123` |
| Dynamic param captured | `params.id = "abc123"` |
| Lookup key reconstructed | `"urls/abc123"` |

This design means the route path and the stored key always align without any transformation magic.

---

### 7.4 Cache Invalidation Strategy

```
New URL shortened
       │
       ▼
revalidatePath('/urls')   ← Server Action
       │
       ▼
Next.js purges the force-cache for GET /api/urls
       │
       ▼
Next visit to /urls fetches fresh data from MongoDB
```

The `/urls` list page uses `cache: 'force-cache'` for performance, but it is explicitly
invalidated every time a new short URL is created, keeping it always up to date.

---

## 8. Database Schema & Model

### Mongoose Schema (`src/models/Url.ts`)

```ts
const urlSchema = new mongoose.Schema({
    originalUrl: {
        type: String,
        required: true,
        unique: true       // ← no duplicate long URLs
    },
    shortUrl: {
        type: String,
        required: true,
        unique: true       // ← no duplicate short codes
    }
}, {
    timestamps: true       // ← adds createdAt + updatedAt automatically
});
```

### Resulting MongoDB Document

```json
{
  "_id": "665f1a2b3c4d5e6f7a8b9c0d",
  "originalUrl": "https://www.example.com/some/very/long/path",
  "shortUrl": "urls/abc123",
  "createdAt": "2026-11-24T10:00:00.000Z",
  "updatedAt": "2026-11-24T10:00:00.000Z",
  "__v": 0
}
```

### Database Connection (`src/config/db.ts`)

```ts
const connectDB = async () => {
    return mongoose.connect(process.env.MONGODB_URI as string);
}
```

`connectDB()` is called in the **Repository constructor**, so a connection is established
automatically the first time any repository is instantiated during a request.

> [!NOTE]
> Mongoose internally pools and reuses connections, so calling `connectDB()` on every request
> doesn't open a new socket each time — it reuses the existing connection if already established.

---

## 9. Caching Strategy

| What | Mechanism | Duration |
|---|---|---|
| `/urls` page data | `fetch` with `cache: 'force-cache'` | Until `revalidatePath` |
| `GET /api/urls` HTTP response | `Cache-Control: public, max-age=180` | 3 minutes |
| Stale-while-revalidate | `stale-while-revalidate=59` | 59 seconds |
| Cache bust on new URL | `revalidatePath('/urls')` in Server Action | Immediate |

---

## 10. Data Flow Summary Table

| User Action | Entry Point | Service Method | Repository Method | DB Operation |
|---|---|---|---|---|
| Submit long URL (form) | `ShortenUrlAction.ts` | `shortenUrl()` | `getUrlByOriginalUrl()` → `createUrl()` | `findOne` → `create` |
| Submit long URL (API) | `POST /api/shorten` | `shortenUrl()` | same as above | same as above |
| View all URLs | `GET /api/urls` | `getAllUrls()` | `getAllUrls()` | `find()` |
| Visit short link | `/urls/[id]/page.tsx` | `getUrlByShortUrl()` | `getUrlByShortUrl()` | `findOne` |

---

## 11. File Responsibility Map

```
src/
├── app/
│   ├── page.tsx
│   │    └── UI only. Renders gradient hero + shorten form.
│   │        Delegates submission to Server Action.
│   │
│   ├── layout.tsx
│   │    └── Root HTML shell. Sets Inter font and global CSS.
│   │
│   ├── globals.css
│   │    └── Minimal global styles (Tailwind base).
│   │
│   ├── api/
│   │   ├── shorten/route.ts
│   │   │    └── REST POST handler. Thin controller → calls service → returns JSON.
│   │   └── urls/route.ts
│   │        └── REST GET handler. Thin controller → calls service → returns JSON + cache headers.
│   │
│   ├── serverActions/
│   │   └── ShortenUrlAction.ts
│   │        └── 'use server' function. Reads FormData, calls service, busts /urls cache.
│   │
│   └── urls/
│       ├── page.tsx
│       │    └── Async Server Component. Fetches /api/urls, renders DaisyUI table.
│       └── [id]/page.tsx
│            └── Dynamic route. Looks up short URL → redirects to original.
│
├── config/
│   └── db.ts
│        └── Single exported async function that connects Mongoose to MongoDB URI.
│
├── models/
│   └── Url.ts
│        └── Mongoose schema + IUrl TypeScript interface.
│            Singleton pattern: reuses existing model if already registered.
│
├── repositories/
│   └── UrlRepository.ts
│        └── All DB queries live here. No business logic.
│            Methods: getById, getByShortUrl, getByOriginalUrl, getAll, create, delete.
│
└── services/
    └── UrlShortenerService.ts
         └── All business logic lives here. No direct DB calls.
             Methods: shortenUrl (dedup + collision), getAllUrls, getUrlByShortUrl.
```

---

> [!TIP]
> When adding new features (e.g. click tracking, custom aliases, expiry dates):
> 1. **Add fields** to `Url.ts` (model)
> 2. **Add queries** to `UrlRepository.ts` (repository)
> 3. **Add logic** to `UrlShortenerService.ts` (service)
> 4. **Add routes or actions** to `src/app/` (presentation)
>
> Never put DB queries in routes/pages, and never put business logic in repositories.
