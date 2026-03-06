# likethisgame.com

**Find games you'll love. AI-powered recommendations based on games you enjoy.**

[Live Site](https://likethisgame.com)

![likethisgame](https://likethisgame.com/og-image.png)

## What is LikeThisGame?

Search for any game and get AI-powered recommendations for similar titles. Built on IGDB's database with real-time SSE streaming, multi-model AI fallback, and a 5-layer cost protection system.

## Stats

| Metric | Value |
|--------|-------|
| Games in database | 26,600+ |
| Recommendation records | 12,700+ |
| Games with recommendations | 7,500+ |
| Test coverage | 21 files, ~447 tests |
| Top-tier coverage (500+ ratings) | 100% |

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 16 (App Router, standalone) |
| UI | React 19 + Tailwind CSS v4 |
| Database | SQLite (better-sqlite3, WAL mode) |
| ORM | Drizzle ORM |
| Cache | Redis (ioredis) + circuit breaker |
| AI (primary) | Gemini 2.5 Flash (via OpenRouter) |
| AI (fallback chain) | Gemini direct → DeepSeek → GPT-4o-mini |
| Game Data | IGDB API |
| Security | Cloudflare Turnstile + FingerprintJS v5 |
| Monitoring | Sentry + OpenPanel + Clarity |
| Hosting | Hetzner CX33 + Coolify + Cloudflare CDN |

## Architecture

```
User Search
    │
    ▼
┌─ Redis Cache ──── hit ──→ JSON response (sub-second)
│       │
│      miss
│       │
│   Turnstile verify → Budget check → Rate limit → SSE slot acquire
│       │
│       ▼
│   getCandidatePool()
│   ├── IGDB similarGames (franchise-filtered)
│   └── 2024+ games (genre/tag relevance scoring)
│       │
│       ▼
│   Gemini 2.5 Flash ──→ SSE Stream ──→ Client
│       │                     │
│      fail                   ▼
│       ▼              RecommendationExtractor
│   DeepSeek ──fail──→ GPT-4o-mini    (streaming JSON parser)
│                          │
│                          ▼
│                    enrichWithCanonicalImages()
│                          │
│              ┌───────────┴───────────┐
│              ▼                       ▼
│        setCache(24h)          DB upsert (non-blocking)
└──────────────────────────────────────────────────────
```

## Notable Engineering Decisions

- **SSE streaming with custom JSON parser**: `RecommendationExtractor` does character-by-character brace-depth tracking to yield each recommendation object as soon as the AI completes it, not waiting for the full response.
- **Circuit breaker on Redis**: 3-state (closed/open/half-open), 3 failures → open, 30s reset. Redis down doesn't crash the app; it degrades gracefully.
- **5-layer cost protection**:
  1. Sliding window rate limit (Lua script, IP + fingerprint)
  2. Daily AI budget (atomic Redis INCR, default 200 req/day)
  3. Criteria combo dedup (8 unique combos/IP/hour via SHA-256)
  4. Anomaly detection (>5 fingerprints/IP/hour → block)
  5. Cloudflare Turnstile (invisible CAPTCHA, only on cache miss)
- **4-model AI fallback chain**: OpenRouter Gemini → Direct Gemini → DeepSeek → GPT-4o-mini. Each provider uses the same OpenAI SDK with `baseURL` override.
- **Organic database growth**: Googlebot's crawl chains trigger IGDB lookups for unknown games, passively expanding the database from 24K to 26K+.
- **Programmatic SEO**: ISR pages with JSON-LD (VideoGame + ItemList + FAQPage), 3-part sitemap, IndexNow for Bing/Yandex.

## Source Code

Source code is in a private repository. This repo serves as a public project overview.

## Author

**Mehmet Aras** — Senior Frontend Engineer
- [arasmehmet.com](https://arasmehmet.com)
- [LinkedIn](https://linkedin.com/in/arasmehmet7)
