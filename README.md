# 🔗 URL Shorty

A clean, full-stack URL shortener built with **Next.js 14**, **MongoDB**, and **Tailwind CSS + DaisyUI**. Paste any long URL and get a short, shareable link in seconds.

---

## ✨ Features

- 🔗 **Shorten any URL** — generates a unique short ID using `shortid`
- ♻️ **Idempotent shortening** — submitting the same URL twice returns the same short link
- 📋 **View all shortened URLs** — browse a table of every link you've created
- ↗️ **Auto-redirect** — visiting a short URL transparently redirects to the original
- ⚡ **Server Actions** — form submission handled server-side without a client-side fetch
- 🗄️ **REST API** — also exposes `/api/shorten` and `/api/urls` endpoints
- 🎨 **DaisyUI + Tailwind** — polished, responsive UI out of the box

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 14 (App Router) |
| Language | TypeScript 5 |
| Database | MongoDB via Mongoose 8 |
| Styling | Tailwind CSS 3 + DaisyUI 4 |
| ID Generation | shortid |
| Runtime | Node.js 20+ |

---

## 📁 Project Structure

```
URL-Shorty/
├── src/
│   ├── app/
│   │   ├── page.tsx                    # Home page — URL input form
│   │   ├── layout.tsx                  # Root layout (Inter font, global CSS)
│   │   ├── globals.css                 # Global styles
│   │   ├── api/
│   │   │   ├── shorten/route.ts        # POST /api/shorten — REST endpoint
│   │   │   └── urls/route.ts           # GET  /api/urls    — list all URLs
│   │   ├── serverActions/
│   │   │   └── ShortenUrlAction.ts     # Next.js Server Action (form handler)
│   │   └── urls/
│   │       ├── page.tsx                # /urls — table of all shortened URLs
│   │       └── [id]/page.tsx           # /urls/[id] — redirect to original URL
│   ├── config/
│   │   └── db.ts                       # MongoDB connection helper
│   ├── models/
│   │   └── Url.ts                      # Mongoose schema & IUrl interface
│   ├── repositories/
│   │   └── UrlRepository.ts            # Data-access layer (CRUD over Url model)
│   └── services/
│       └── UrlShortenerService.ts      # Business logic (shortening, dedup, lookup)
├── public/                             # Static assets
├── tailwind.config.ts                  # Tailwind + DaisyUI config
├── next.config.mjs                     # Next.js config
├── tsconfig.json                       # TypeScript config
└── package.json
```

---

## ⚙️ Architecture Overview

```
Browser
  │
  ├─► Home Page (/)
  │     └─► <form> → Server Action (ShortenUrlAction)
  │                       └─► UrlShortenerService.shortenUrl()
  │                               └─► UrlRepository → MongoDB
  │
  ├─► /urls               → fetches GET /api/urls → lists all links
  │
  ├─► /urls/[id]          → Server Component → UrlShortenerService.getUrlByShortUrl()
  │                               └─► redirect(originalUrl)
  │
  └─► REST API
        ├─► POST /api/shorten  { originalUrl } → { shortUrl }
        └─► GET  /api/urls                     → { urls: [...] }
```

**Layered design:**

- **Model** (`Url.ts`) — Mongoose document schema with `originalUrl` + `shortUrl` fields and automatic timestamps  
- **Repository** (`UrlRepository.ts`) — thin data-access wrapper; isolates all Mongoose queries  
- **Service** (`UrlShortenerService.ts`) — business logic: deduplication, collision-safe ID generation, delegation to repository  
- **API Routes / Server Actions** — thin controllers that call the service and return HTTP responses or trigger cache revalidation  

---

## 🚀 Getting Started

### Prerequisites

- **Node.js** v20 or later
- **npm** (or pnpm / yarn)
- A **MongoDB** instance (local or [MongoDB Atlas](https://www.mongodb.com/atlas))

### 1. Clone the repository

```bash
git clone https://github.com/your-username/url-shorty.git
cd url-shorty
```

### 2. Install dependencies

```bash
npm install
```

### 3. Configure environment variables

Create a `.env.local` file in the project root:

```env
# MongoDB connection string
MONGODB_URI=mongodb://localhost:27017/url-shorty

# Public base URL (used to build displayed short links)
NEXT_PUBLIC_BASE_URL=http://localhost:3000
```

> [!TIP]
> For MongoDB Atlas, replace the URI with your cluster connection string, e.g.
> `mongodb+srv://<user>:<password>@cluster0.xxxxx.mongodb.net/url-shorty`

### 4. Run the development server

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) in your browser.

---

## 📜 Available Scripts

| Command | Description |
|---|---|
| `npm run dev` | Start the development server with hot-reload |
| `npm run build` | Build the app for production |
| `npm start` | Start the production server (run after `build`) |
| `npm run lint` | Run ESLint across the project |

---

## 🌐 API Reference

### `POST /api/shorten`

Shorten a URL via REST.

**Request body:**
```json
{ "originalUrl": "https://example.com/very/long/path" }
```

**Response `201`:**
```json
{ "shortUrl": "abc123" }
```

---

### `GET /api/urls`

Retrieve all shortened URLs.

**Response `200`:**
```json
{
  "urls": [
    {
      "_id": "...",
      "originalUrl": "https://example.com/very/long/path",
      "shortUrl": "urls/abc123",
      "createdAt": "2026-11-24T00:00:00.000Z",
      "updatedAt": "2026-11-24T00:00:00.000Z"
    }
  ]
}
```

> [!NOTE]
> Responses are cached with `Cache-Control: public, max-age=180, stale-while-revalidate=59`.
> The `/urls` page uses `revalidatePath` after each new submission to keep the list fresh.

---

## 🗄️ MongoDB Schema

| Field | Type | Constraints |
|---|---|---|
| `originalUrl` | String | Required, Unique |
| `shortUrl` | String | Required, Unique |
| `createdAt` | Date | Auto (Mongoose timestamps) |
| `updatedAt` | Date | Auto (Mongoose timestamps) |

---

## 🔄 How Short URL Generation Works

1. User submits a long URL.  
2. The service checks if the URL **already exists** in the database — if so, the existing short code is returned (idempotent).  
3. A new short ID is generated with `shortid`.  
4. The database is checked for a **collision** on that short ID; if one is found, a new ID is generated and the check repeats.  
5. The collision-free pair is saved as `{ originalUrl, shortUrl: "urls/<id>" }`.  
6. Visiting `/<shortUrl>` triggers a server-side redirect to the original URL.

---

## 🚢 Deployment

### Vercel (recommended)

1. Push the repository to GitHub.  
2. Import it on [vercel.com](https://vercel.com).  
3. Add the environment variables (`MONGODB_URI`, `NEXT_PUBLIC_BASE_URL`) in the Vercel dashboard.  
4. Deploy — Vercel handles build and serving automatically.

### Self-hosted

```bash
npm run build
npm start
```

Set `PORT` and the two environment variables in your server environment before running.

---

## 🤝 Contributing

1. Fork the repository  
2. Create a feature branch: `git checkout -b feat/your-feature`  
3. Commit your changes: `git commit -m "feat: add your feature"`  
4. Push to the branch: `git push origin feat/your-feature`  
5. Open a Pull Request  

---

## 📄 License

This project is open source. Feel free to use, modify, and distribute it.
