# Travel Planner: data model (DRAFT, 26 Oct 26)

> **Status: proposal, waiting for the owner's approval.** Nothing here is built. The SQL is a sketch to make the design concrete, not a migration.

## The big picture

One Supabase project, two apps:

| Table | Owned by | Planner reads | Planner writes |
|---|---|---|---|
| `trips`, `trip_members`, `places`, `visits` (+ itinerary tables) | **travel-app** (shared) | yes | `trips` (drafts only), `places` (only when sharing an idea to a trip) |
| `ideas`, `idea_shares` | **travel-planner** (planner-only) | yes | yes |

The Companion never reads the planner's tables. Once an idea is shared, it becomes an ordinary trip place, so the Companion needs no new code to show it.

## Planner-only tables

### `ideas`: my private ideas inbox

One row per idea. Private to the owner, always. Sharing never exposes the idea itself, only a copy (see `idea_shares`).

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Made on the phone |
| `owner_id` | uuid | `auth.users`, defaults to `auth.uid()` |
| `kind` | text | `place`, `activity`, `destination`, `link`, `note` |
| `title` | text | Required. "Hot-air balloon at dawn" |
| `notes` | text | Free text |
| `url` | text | Optional link (blog, reel, booking page to look at later) |
| `tags` | text[] | Free tags, e.g. `{food, rainy-day}`. Simple array for v1 |
| `lat`, `lng` | double | Optional map pin. Both or neither |
| `place_label` | text | Shown name of the pin: "Luxor, Egypt" |
| `country_code` | char(2) | ISO code, for "ideas in Japan" and the lifetime map |
| `status` | text | `inbox` (new), `keep`, `done`, `dropped` |
| `starred` | boolean | Quick favourite |
| `created_at`, `updated_at`, `updated_by`, `deleted_at` | | Same as every Companion table (soft delete) |

### `idea_shares`: which ideas went to which trip

Created when I tap "Add to trip". The app copies the idea into the trip as a `places` row (owned by the trip), then records the link here.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | |
| `idea_id` | uuid | → `ideas` |
| `trip_id` | uuid | → `trips` |
| `place_id` | uuid | → `places`, the trip copy |
| `shared_at`, `shared_by` | | |
| `created_at`, `updated_at`, `updated_by`, `deleted_at` | | |

- **Copy, not live link.** Editing the idea later doesn't change the trip copy, and the trip's edits don't change my idea. My private notes stay private: the share screen shows what will be copied.
- Ideas with no pin are copied as a place with no coordinates (needs `places.lat/lng` to be optional, see CR-03).

### Row Level Security (sketch)

```sql
-- ideas: only the owner, for everything
create policy ideas_owner on ideas
  for all using (owner_id = auth.uid()) with check (owner_id = auth.uid());

-- idea_shares: owner of the idea, and only into trips they belong to
create policy idea_shares_owner on idea_shares
  for all using (exists (select 1 from ideas i where i.id = idea_id and i.owner_id = auth.uid()))
  with check (
    exists (select 1 from ideas i where i.id = idea_id and i.owner_id = auth.uid())
    and is_trip_member(trip_id)   -- helper from travel-app's schema
  );
```

## Trips and the lifetime log (shared tables, read by the planner)

- **Flow:** idea → `trips.status = 'draft'` → `'active'` (opened in the Companion) → `'archived'`.
- **Lifetime log** = all `visits` on trips I'm a member of, grouped by country, city and year. Past travel from before the app is entered as archived "past trips" with city- or country-level visits.

## Change requests for travel-app (shared tables)

Apply these through travel-app, **before its schema is approved**, so the Companion doesn't need a migration mid-trip.

| # | Change | Why | Status |
|---|---|---|---|
| CR-01 | *Places belong to a user OR a trip* | Was for the wishlist | **Propose withdrawing.** Private ideas now live in `ideas`, so `places` can stay trip-only. Less change to the Companion |
| CR-02 | `visits.trip_id` required, `visits.place_id` optional; add `label`, `country_code`, optional `lat`/`lng` | A visit without a saved place ("wandered the old town") still counts and maps; past trips can be logged at city level | Keep (extends the earlier proposal) |
| CR-03 | `places.lat`/`lng` optional | Shared ideas without a pin | New, small |
| CR-04 | `trips.status`: `draft` / `active` / `archived` (replaces an archived flag); `start_date`/`end_date` optional | Draft trips made in the Planner; the Companion shows only active (and archived read-only) | New |
| CR-05 | Companion CLAUDE.md notes: wishlist moves to the Planner; Dexie name and service-worker scope must be unique per app; Supabase URL and key read from build settings | Both apps share one web address (`jcm-book-library.github.io`) | New, docs only |

## Open questions that change this model

See the questions in the planning reply (26 Oct 26). The ones that matter most: where the day-by-day plan is built (Q1), copy vs live link (Q2), and backfilling past trips (Q3).
