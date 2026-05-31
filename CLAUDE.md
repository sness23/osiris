# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

> ⚠️ The line above imports `AGENTS.md`: **this is Next.js 16, with breaking
> changes vs. older training data. Read `node_modules/next/dist/docs/` before
> writing framework code.** Heed deprecation notices.

## Project

OSIRIS — a real-time OSINT situational-awareness dashboard. A single GPU-rendered
MapLibre globe aggregates ~16 live intelligence layers (aviation, maritime, CCTV,
seismic, fires, news, satellites, conflict, cyber/CVE, crypto, sanctions, Telegram)
plus a browser-based RECON toolkit (port/DNS/WHOIS/SSL/IP/CVE/sanctions/crypto).

## Commands

```bash
npm run dev      # next dev — local dev server on :3000
npm run build    # next build — standalone production build (output: 'standalone')
npm start        # next start — serve the production build
npm run lint     # eslint (next/core-web-vitals + typescript configs)

docker compose up -d   # build/run the alpine standalone image on $OSIRIS_PORT (def 3000)
```

There is **no test suite** (no `test` script, no test runner). Don't fabricate one.

**`next.config.ts` sets `typescript.ignoreBuildErrors: true`** — `npm run build`
will **not** fail on type errors. Rely on `npm run lint` and your editor's
typechecker; never assume a green build means types are sound.

## Environment / Keys

`.env` is gitignored; `.env.template` is the documented source of truth (richer
than `.env.example`). **Only `SCANNER_URL` and `SCANNER_KEY` are actually read by
current code** — they point the RECON routes at a *separate* scanner backend
(`SCANNER_KEY` must equal that backend's `OSIRIS_KEY`); without them the RECON
toolkit returns `503` and every other layer still works keyless. All other keys
(`FIRMS_API_KEY`, `OPENSKY_*`, `N2YO_API_KEY`, `AIS_API_KEY`, `GEMINI_API_KEY_1..8`)
are optional/reserved — core feeds use public keyless sources.

## Architecture

**Three tiers:** client SPA → Next.js API routes (proxies) → external keyless feeds.

### Client data flow (the core pattern)
`src/app/page.tsx` is one big `'use client'` dashboard (~62 KB). It does **not**
hold feed data in React state. Instead:

1. Polls `/api/*` on staggered `setInterval`s via the shared `fetchEndpoint(url, transform?)`.
2. Merges each response into a **mutable ref** `dataRef.current` (avoids re-render
   storms from high-frequency feeds), then bumps a single `dataVersion` counter.
3. Passes `data={dataRef.current}` + `activeLayers` down to `OsirisMap`.

Layers default **OFF** for fast first paint; data is fetched on-demand when a layer
is toggled, and viewport-aware. When editing the polling loops, keep the
`document.hidden` early-return and the existing intervals — they're tuned to stay
under feed rate limits.

### Map rendering — `src/components/OsirisMap.tsx` (~88 KB)
MapLibre GL, dynamically imported with `ssr: false`. Pattern: on init it
`addSource`/`addLayer`s every layer **once** with empty GeoJSON, then on each
`data`/`activeLayers` change it imperatively `getSource(id).setData(features)` and
toggles `setLayoutProperty(id, 'visibility', …)`. All entities render via WebGL,
never the DOM. Adding a layer = (1) a `/api/<x>` route, (2) a poll in `page.tsx`,
(3) source + layer defs + a `setData` update branch here, (4) a `LayerPanel` toggle.

### API routes — `src/app/api/**/route.ts`
Thin server-side proxies that normalize external feeds (keeps keys server-side,
dodges CORS/rate limits). Long-poll/SSE/dynamic routes set `export const dynamic =
'force-dynamic'`. `vercel.json` caps every API function at `maxDuration: 30`.
- `/api/osint/*` + `/api/scanner` — RECON toolkit; forward to the external scanner backend.
- `/api/sdk/{stream,ingest}` — Polybolos SDK SSE/ingest (see below).

### Security-sensitive helpers — `src/lib/`
- **`ssrf-guard.ts`** — `validateHost()` canonicalizes + DNS-resolves any
  user-controlled host and rejects reserved/internal ranges (blocks cloud-metadata
  SSRF, decimal/IPv6 IP bypasses). **Every route that fetches a user-supplied host
  MUST go through it**, plus `isRateLimited()`/`getClientIp()`. See `scanner/route.ts`.
- `stealthFetch.ts` — randomized headers/UA to spread outbound requests across feeds.
- `ai-engine.ts` — Gemini wrapper with round-robin `rotateApiKey()` over `GEMINI_API_KEY_1..8`.
- `sanctions.ts` / `osint-utils.ts` — OFAC SDN matching + shared OSINT helpers.

### Polybolos SDK — `src/lib/sdk/`
Optional fusion layer that normalizes all OSIRIS feeds into a single
`PolybolosEntity[]` (`types.ts`), with translators in `PolybolosClient.ts` and a
`LatticeAdapter` for external providers. Streams over SSE at `/api/sdk/stream`
backed by an in-memory `globalThis.sdkEntityStore`.

## Workflow rule

**`DO_NOT_PUSH.md`: do not `git push` unless the user explicitly reviews and says
"GO FOR IT".** Commit locally if asked, but never push without that sign-off.
