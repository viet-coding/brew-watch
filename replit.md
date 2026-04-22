# Overview

This project is a pnpm workspace monorepo using TypeScript, designed to provide competitor pricing intelligence for Highlands Coffee. It tracks weekly beverage prices across 7 Vietnamese coffee chains, offering a comprehensive dashboard for market analysis. The system aims to give Highlands Coffee a strategic advantage by providing timely insights into market dynamics, competitor strategies, and potential business opportunities.

Key capabilities include:
- A dashboard for visualizing KPIs, price history, and changes.
- An API for data retrieval and ingestion.
- A "Like-for-Like" catalog for standardized product comparisons.
- Multi-source data collection (web and ShopeeFood lanes) for comprehensive coverage.
- Automated price-change alerts.
- A Playwright-based scraper for data collection.
- Macroeconomic signal tracking for input costs and pricing power.
- Social media listening for promotions.

## User Preferences

- **Do not make changes to the folder `Z`**.
- **Do not make changes to the file `Y`**.
- Re-scraping is expensive — never use `--wipe-data` casually, and never wipe scraped data without first asking the user.
- Promote store directory changes manually: the team adds proven query overrides to `BRAND_QUERY_OVERRIDES` rather than ever auto-promoting staging rows.
- Wide sweep is manual only — never scheduled — because the ~150-query run adds CAPTCHA risk on top of the production weekly budget.
- HARD RULE: real, citable data only — every observation row in the macro layer is anchored to a `sourceUrl` recorded in its ingestion run.
- HARD RULE: never use the em dash character "—" (U+2014) in AI-generated content. Use a regular hyphen "-" or rephrase instead.

## System Architecture

The project is structured as a pnpm workspace monorepo, leveraging TypeScript for type safety across all packages.

**Technical Stack:**
- **Monorepo tool**: pnpm workspaces
- **Node.js version**: 24
- **Package manager**: pnpm
- **TypeScript version**: 5.9
- **API framework**: Express 5
- **Database**: PostgreSQL + Drizzle ORM
- **Validation**: Zod (`zod/v4`), `drizzle-zod`
- **API codegen**: Orval (from OpenAPI spec)
- **Build**: esbuild (CJS bundle)

**UI/UX Decisions:**
The `artifacts/brew-watch` dashboard is built with React, Vite, wouter, recharts, and shadcn/ui.
- **Overview KPIs**: Summarizes key performance indicators.
- **Prices**: Filterable table with 12-week history, side panel, and CSV export.
- **Changes**: Week-grouped log of price modifications.
- **Like-for-Like (LFL) Catalogue**: Defines ~20 canonical drinks with competitor name aliases and `expectedSize` for fair comparison. The matcher is deterministic. The Overview page renders the LFL table with a Highlands baseline column, per-competitor price + Δ% chips, an "Only ≥3 chains" toggle, and click-through to `/prices?canonical=<slug>`.
- **Multi-source per chain**: Supports `web` and `shopeefood` sources. `/prices/like-for-like` emits three lanes per cell (`web`, `shopeefood`, `min`).
- **Admin Panel**: Provides tools for managing store directories and macro data ingestion.

**API (`artifacts/api-server`)**:
Express routes under `/api/*` for health checks, competitor data, categories, current and historical prices, LFL prices, changes, promos, overview KPIs, and a bearer-token-protected `/ingest` endpoint.

**Database Schema (`lib/db/src/schema/`)**:
Includes tables for `competitors`, `categories`, `canonical_products`, `raw_products`, `ingestion_runs`, `price_snapshots`, `promos`, `price_alerts_log`, `competitor_stores_staging`, `wide_sweep_runs`, `store_directory_runs`, `macro_series`, `macro_observations`, `macro_ingestion_runs`, `macro_events`, and `macro_series_events`.

**Key Features Implementation:**
- **Price-change alerts**: After each ingest, `priceAlerts.ts` diffs snapshots, recording changes in `price_alerts_log` if `|Δ%| ≥ ALERT_PRICE_THRESHOLD_PCT` or the competitor is a benchmark. Alerts are grouped per competitor and sent via Resend email.
- **Scraper (`tools/scraper`)**: Playwright-based ShopeeFood scraper. Scheduled daily by the api-server.
- **Wide-sweep store-directory engine**: Staging-only QC tool for on-demand Google Maps sweeps across all 63 Vietnamese provinces, storing results in `competitor_stores_staging`.
- **Store-directory scraper**: Per-brand dispatcher pulling locator data from various sources (Haravan, WordPress, official sites, Google Maps).
- **Cong Caphe official locator**: `scripts/scrape-cong-official.ts` parses the Next.js `__NEXT_DATA__` blob inlined on `https://congcaphe.com/stores`, filters to the 17 VN province terms under the "Việt Nam" parent, and upserts under `sourceKind='cong_official'` (authority 100). Coords absent from the locator are filled by `backfill-coords-trackasia.ts --brand=cong`. Stale `trackasia_search` rows are deactivated only when they are >200m from any cong_official row to avoid nuking real stores on address-string drift; in-proximity duplicates are left for the nightly within-30m+name-overlap dedup pass.
- **Macro Layer v1**: Tracks Vietnam macro signals (PVOIL fuel prices, VND/USD exchange rate, CPI series) via `macroFetcher.ts`. Data is stored in `macro_series`, `macro_observations`, `macro_ingestion_runs`, and `macro_events` tables.
- **Social listener QC**: Applies URL, recency, and within-run dedup filters to Serper results for Facebook/Instagram promo listening.

## External Dependencies

- **Resend**: For sending price-change alert emails. Configured with `RESEND_API_KEY`, `ALERT_EMAIL_RECIPIENTS`, `ALERT_FROM_EMAIL`.
- **Playwright**: Used by the scraper for browser automation.
- **Serper API**: Utilized by the social listener for searching Facebook and Instagram posts.
- **Google Maps**: Used for store directory wide sweeps and some store locator scraping.
- **Vietcombank**: XML feed for VND/USD exchange rate data.
- **PVOIL**: Website (`https://www.pvoil.com.vn/tin-gia-xang-dau`) for fuel price data.
- **CEIC**: Subscription service for Vietnam CPI data, with exports used for manual import.