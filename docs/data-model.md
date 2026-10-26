# Travel Planner: data model (DRAFT v2, 26 Oct 26)

> **Status: proposal, waiting for the owner's approval.** Nothing here is built. The SQL is a sketch to make the design concrete, not a migration.

## Owner's answers (26 Oct 26)
- The trip is **built in the Planner**; the Companion takes over for travel and allows changes on the road.
- All ideas start in the Planner.
- Past travel will be logged (archived "past trips").
- Owner's phone: **iPhone** (no share-to-app for home-screen apps).
- Mostly online, but **adding an idea offline** would be good.
- Map is a **corkboard**: rough pins on cities ("throw a dart"), string drawn between them. No exact coordinates needed.
- Lifetime log: **map first**, then countries, then cities, with years beside each.
- Supabase account exists but nothing is set up yet, so the shared database doesn't exist yet either.

## The big picture

One Supabase project, two apps, **one set of trip tables**. There is no import step: the Planner writes a draft trip straight into the shared tables, and "Hand over to Companion" just changes the trip's status from `draft` to `active`.

| Table | Owned by | Planner reads | Planner writes |
|---|---|---|---|
| `trips`, `trip_members` | travel-app (shared) | yes | draft trips only |
| `trip_stops` (new, CR-06) | travel-app (shared) | yes | draft trips only |
| `places` | travel-app (shared) | yes | draft trips, and copies of shared ideas |
| `visits` | travel-app (shared) | yes (lifetime log) | past trips only (backfill) |
| itinerary items | travel-app (shared) | later | later (detailed planning, phase 2) |
| `ideas`, `idea_shares` | travel-planner | yes | yes |

## Planner-only tables

### `ideas`: my private ideas inbox

Private to the owner, always. Sharing copies an idea into a trip; it never exposes the idea row.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Made on the phone, so ideas can be added offline |
| `owner_id` | uuid | `auth.users`, defaults to `auth.uid()` |
| `kind` | text | `place`, `activity`, `destination`, `link`, `note` |
| `title` | text | Required. The only required field |
| `notes` | text | |
| `url` | text | |
| `tags` | text[] | Free tags, e.g. `{food, rainy-day}` |
| `lat`, `lng` | double | Optional rough pin ("dart"). Both or neither |
| `place_label` | text | "Kyoto, Japan". Filled from the pin (one reverse look-up) or typed |
| `country_code` | char(2) | ISO code, filled from the pin when there is one |
| `status` | text | `inbox`, `keep`, `done`, `dropped` |
| `starred` | boolean | |
| `created_at`, `updated_at`, `updated_by`, `deleted_at` | | Same as every Companion table |

### `idea_shares`: which ideas went to which trip

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | |
| `idea_id` | uuid | → `ideas` |
| `trip_id` | uuid | → `trips` |
| `place_id` | uuid | → `places`, the trip copy |
| `shared_at`, `shared_by` | | |
| `created_at`, `updated_at`, `updated_by`, `deleted_at` | | |

Copy, not live link: later edits on either side don't change the other.

### Offline "add idea"

New ideas made offline wait in a small on-phone queue (key `tp_outbox`) and are sent when the connection is back. Only **adding** works offline; editing, the map and trips need a connection. Safe because ids are made on the phone (a resend can't make duplicates).

### Row Level Security (sketch)

```sql
create policy ideas_owner on ideas
  for all using (owner_id = auth.uid()) with check (owner_id = auth.uid());

create policy idea_shares_owner on idea_shares
  for all using (exists (select 1 from ideas i where i.id = idea_id and i.owner_id = auth.uid()))
  with check (
    exists (select 1 from ideas i where i.id = idea_id and i.owner_id = auth.uid())
    and is_trip_member(trip_id)   -- helper from travel-app's schema
  );
```

## The corkboard (draft trips)

- A draft trip is a `trips` row with `status = 'draft'`.
- Each pin on the corkboard is a **stop** (`trip_stops`, CR-06): a rough point, a label ("Riga"), an order, rough nights and notes.
- The **string** is drawn between stops in their order. Drag to reorder and the string follows. (Free strings between any two pins: later, if wanted.)
- Ideas can be dropped onto a stop. That copies them into the trip as `places` linked to the stop.
- Because stops are shared, the Companion's day / city / region grouping uses the same stops after hand-over.

## Lifetime log

- **Map first:** visited countries shaded (free world-borders file from Natural Earth, public domain), visited places as dots, strings for each past trip's route.
- Then a **countries** list, then **cities**, each with the year(s) visited.
- Source: `visits` on every trip I'm a member of (active and archived). Past travel goes in as archived trips with city- or country-level visits.

## Change requests for travel-app

The full text to hand over is in `supabase/requests/for-travel-app.md`. Summary:

| # | Change | Status |
|---|---|---|
| CR-01 | *Places belong to a user OR a trip* | **Withdraw.** Private ideas live in `ideas` |
| CR-02 | Visits: `trip_id` required, `place_id` optional, plus `label`, `country_code`, optional `lat`/`lng` | Keep |
| CR-03 | Places: `lat`/`lng` optional; optional `stop_id` | New |
| CR-04 | Trips: `status` draft / active / archived; dates optional for drafts | New |
| CR-05 | Companion CLAUDE.md notes (wishlist moves; unique storage names; settings from build) | New, docs only |
| CR-06 | New shared `trip_stops` table (the city level of a trip) | New |
