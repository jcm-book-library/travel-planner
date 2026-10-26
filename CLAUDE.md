# Travel Planner: notes for Claude

> **DRAFT (26 Oct 26). PLAN ONLY.** No building in this repo until the Travel Companion's Phase 1 is frozen (mid-Jan 27) **and** the owner approves the data model in `docs/data-model.md`. Lines marked **(proposed)** are not decided yet.

**Read the house rules first:** [HOUSE-RULES.md](https://github.com/jcm-book-library/control-room/blob/main/HOUSE-RULES.md) (plain text: https://raw.githubusercontent.com/jcm-book-library/control-room/main/HOUSE-RULES.md). Then read the Companion's notes: [travel-app/CLAUDE.md](https://github.com/jcm-book-library/travel-app/blob/main/CLAUDE.md). Where this file is more specific, it wins. The charter is `project.json`.

## What this is
- A personal, phone-first app to collect travel ideas, shape them into draft trips, and keep a lifetime log of where I've been.
- Partner to the **Travel Companion** (`travel-app`), which is used live on a trip. Same Supabase project, same sign-in.
- **Flow:** idea → draft trip, built in the Planner → "Hand over" makes it active in the Companion (changes on the road happen there) → archived trip, with its visits kept as my log.
- **No import step:** both apps use the same trip tables. Hand-over only changes `trips.status` from `draft` to `active`.
- **Split:** the Companion records visits day by day on a trip; the Planner shows the lifetime view across all trips.
- **Out of scope:** bookings, money tracking, photos.

## The Companion comes first
- Companion Phase 1 must be done by mid-Jan 27; departure Feb 27. If work here would slow it down, stop and say so.
- The only urgent Planner work before the freeze is getting the **change requests** (below) into the Companion's schema.

## Owner's answers (26 Oct 26)
- Trips are built in the Planner; all ideas start here.
- Owner's phone is an **iPhone** (home-screen apps can't receive shares, so capture is paste-a-link).
- Mostly online; **adding an idea offline** should work.
- Map is a **corkboard**: rough pins on cities ("throw a dart") with string between them. No exact coordinates.
- Lifetime log: map first, then countries, then cities, years beside each.
- Past travel will be logged as archived "past trips".

## Stack (proposed: same as the Companion, minus the offline database)
- Vite + React + TypeScript, `vite-plugin-pwa` for install and an offline app shell.
- Supabase JS for sign-in and data. **No Dexie in v1:** the Planner is online-only and says so when offline, except **adding an idea**, which waits in a small queue (`tp_outbox`) and sends when back online (proposed).
- Same map library and tile provider as the Companion. Pins are dropped by tapping the map; one reverse look-up per pin fills the label and country (e.g. Nominatim, within its usage policy: no search-as-you-type).
- Lifetime map shades countries from a free world-borders file (Natural Earth, public domain), bundled with the app.
- Tests: Vitest for logic, Playwright at 390px and 1280px.
- Packages pinned to exact versions. GitHub Pages built by a GitHub Action from `main`, published at `/travel-planner/`.

## Shared code with the Companion
- Copied, not linked, in v1: `lib/supabase.ts`, the email-code sign-in, `styles/tokens.css`, date and time helpers, shared-table types. Each copied file starts with a comment naming its source in `travel-app`.
- Fix shared bugs in `travel-app` first, then copy across.

## Shared database rules
- **travel-app owns all migrations for shared tables** (`trips`, `trip_members`, `places`, `visits`, itinerary tables). Never create or change them from here. Write a change request in `supabase/requests/` and the owner applies it via travel-app.
- **Planner-only tables:** `ideas`, `idea_shares` and any future planner-only tables. Their SQL lives in `supabase/planner/` as numbered files (`P001_ideas.sql`, ...) (proposed).
- The Planner may write to shared tables only for: **draft** trips (and their `trip_stops` and `places`), hand-over (`draft` → `active`), and visits on **archived past trips** (backfill). Once a trip is active, the Companion owns it.
- Same rules as the Companion: RLS on every table, uuid ids made on the phone, `created_at`, `updated_at`, `updated_by`, `deleted_at` on every row, soft deletes, last write wins.
- **Ideas are private.** RLS: only the owner can see their ideas. Sharing copies an idea into a trip; it never exposes the idea row.
- The Supabase URL and anon key come from build settings (`VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`), so a private copy can point at a different project. The service-role key never goes in the repo or the app.
- **Free tier watch:** 500 MB database, projects pause after 7 days unused. Flag anything that could need a paid plan.

## Same web address as the Companion
- Both apps live on `jcm-book-library.github.io`, so they share browser storage with each other and every other house app.
- Keep names unique: localStorage keys prefixed `tp_`, service worker scoped to `/travel-planner/`, any IndexedDB named `travel-planner`.
- On Android the sign-in is shared between the two apps. On iPhone each home-screen app has its own storage, so expect to sign in once per app.

## Rules for the code
- **Mobile-first, one-handed:** main actions in the bottom third, touch targets at least 44px, no sideways scrolling.
- **Readable in light and dark mode** (house look, Proton Seasons tokens).
- **Quick capture beats perfect data:** adding an idea needs only a title; everything else is optional.
- **Display:** Australian English, dates `19 Oct 26`, 24-hour times, °C, km.

## Privacy (the repo and site are public)
- No real places I've been, trip plans, names, bookings or photos in the repo. Real data goes into Supabase only.
- Test and seed data is made up.
- **No work data** in this repo or this Supabase project. If the app is tested for work, use a private copy with its own backend.

## Folder structure (proposed)
```
travel-planner/
  CLAUDE.md  README.md  project.json
  index.html  package.json  vite.config.ts  .env.example
  public/                 manifest.webmanifest, icons/
  src/
    main.tsx
    app/                  routes, layout, bottom nav (Ideas, Trips, Map, Log)
    features/
      auth/               email-code sign-in (copied from travel-app)
      ideas/              inbox, list, filters, tags, idea editor
      capture/            quick add, paste a link, offline outbox
      trips/              draft trips, corkboard (stops + string), hand-over
      map/                shared map pieces (pins, strings, borders)
      log/                lifetime log: map, countries, cities, years; past trips
    lib/                  supabase.ts, types.ts, dates.ts, geo.ts
    components/           shared UI
    styles/tokens.css     house look tokens (copied from travel-app)
  supabase/
    planner/              P001_ideas.sql, ... (planner-only tables)
    requests/             for-travel-app.md: change requests for travel-app
    seed.sql              made-up demo ideas
  docs/data-model.md
  tests/  unit/  e2e/
  .github/workflows/pages.yml
```

## Phase plan (proposed)
**Now to mid-Jan 27: plan only.** Change requests into travel-app (`supabase/requests/for-travel-app.md`); agree the data model.

**After the Companion freeze (start date to be agreed):**
1. Shell: install as PWA, sign in, empty Ideas screen.
2. Ideas: quick add (works offline), list, edit, tags, status, star.
3. Corkboard: draft trip, drop pins (stops), string between them, reorder, drop ideas onto stops.
4. Hand over: draft → active, opens in the Companion.
5. Lifetime log: map, countries, cities, years; enter past trips.

**Phase 2 (only once the above is in use):** detailed day-by-day planning in the Planner (timed items, using the Companion's item rules), free strings between any pins, link previews, iOS Shortcut "Save to Planner".

## Working with the owner
- Action items first, short steps, say exactly what to click.
- 2–3 options with trade-offs and a recommendation. Push back.
- Small steps; commit and push each working milestone to the session branch; `main` only through a PR the owner merges.
- Model use per HOUSE-RULES section 3: lead session for data model, RLS and shared-table decisions; Sonnet subagents for screens from a spec and Playwright tests; Haiku for sweeps and checks.
