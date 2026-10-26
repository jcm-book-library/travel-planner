# Change requests from travel-planner to travel-app (26 Oct 26)

**For the travel-app session:** please fold these into the Companion's data model **before the owner approves it**, so no migration is needed close to the trip. The Planner never creates migrations for shared tables. Background: `travel-planner/docs/data-model.md`.

**How it fits:** both apps use the same tables. The Planner creates a trip as `draft` and builds it; "Hand over" sets it to `active`, and from then on the Companion owns it. The Companion should ignore `draft` trips, except perhaps showing them in a later trip switcher.

## CR-01: withdraw "places belong to a user OR a trip"
Places stay trip-only. The Planner keeps private ideas in its own `ideas` table.

## CR-02: visits
- `trip_id` required; `place_id` optional.
- Add `label` (text), `country_code` (char 2, ISO), optional `lat`/`lng`.
- Why: a visit without a saved place ("wandered the old town") still counts and maps. Past trips can be logged at city or country level. The Planner's lifetime log groups visits by country, city and year.

## CR-03: places
- `lat`/`lng` optional (an idea shared without a pin).
- Add optional `stop_id` → `trip_stops`.

## CR-04: trips
- `status`: `draft` | `active` | `archived` (instead of an archived flag). Archived stays read-only.
- `start_date`/`end_date` optional while `draft` (an early plan may have no dates).

## CR-05: notes for travel-app's CLAUDE.md
- The Phase 2 "wishlist on the map" moves to the Planner.
- Both apps live on `jcm-book-library.github.io`, so storage names must be unique per app: Dexie database name, service-worker scope `/travel-app/`, localStorage prefix.
- Read the Supabase URL and anon key from build settings (`VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`).

## CR-06: new shared table `trip_stops`
The city level of a trip. The Planner draws these as corkboard pins with string between them; the Companion can use them for its day / city / region grouping.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Made on the phone |
| `trip_id` | uuid | → `trips` |
| `position` | integer | Order along the route |
| `label` | text | "Riga" |
| `region` | text | Optional: "Baltics" |
| `country_code` | char(2) | |
| `lat`, `lng` | double | Rough point, optional |
| `time_zone` | text | IANA, optional while draft |
| `arrive_on`, `leave_on` | date | Optional; or `nights` (int) while draft |
| `notes` | text | |
| `created_at`, `updated_at`, `updated_by`, `deleted_at` | | Standard |

RLS: same as other trip tables (by `trip_members`).

If the Companion's own model already has a city or stay level, extend that instead of adding a second one, and tell the owner the name so the Planner matches it.

## Also: keeping the free project awake
Free Supabase projects pause after 7 days with no activity. Proposed: a scheduled GitHub Action that makes one tiny read every 3 days. See the Planner's plan for options; the owner decides where it lives.
