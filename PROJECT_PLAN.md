# Mopac Landscaping — Build Plan (Harlem & Micah)

Work through phases **in order** — each one depends on the last. Inside a
phase, each of you owns a track end-to-end (UI → server logic → database),
not a frontend/backend split. Swap tracks between phases if you want; the
point is you both touch the whole stack.

Every task below has: what it's for, the steps, and how you'll know it's
done. See `CLAUDE.md` for the underlying architecture/data model/auth
decisions these tasks implement.

---

## Phase 0 — Foundation (do together)

- [ ] Create the Supabase project, grab the API keys
- [ ] Scaffold the Next.js app (TypeScript, Tailwind, App Router)
- [ ] Write the SQL migration: `properties`, `regions`, `pins` tables, the
      legend category enum, RLS policies (admin-only reads/writes via RLS;
      crew reads happen through a server-only client — see `CLAUDE.md`)
- [ ] Apply the migration to Supabase, confirm tables exist
- [ ] Set up `.env.local.example` documenting every required env var
- [ ] Get a blank page building and deploying to Vercel

**Done when:** an empty Next.js app is deployed and connected to a live
Supabase database with the schema in place.

---

## Phase 1 — Access Control

Nothing past this point is reachable without a working login/PIN gate.

### Harlem — Track A: Admin Login

- [ ] **Supabase client helpers**
  Give the app a way to talk to Supabase from both the browser and the
  server.
  - Install `@supabase/supabase-js` and `@supabase/ssr`.
  - `src/lib/supabase/client.ts` — exports `createClient()` using
    `createBrowserClient(url, publishableKey)`. This is what client
    components use.
  - `src/lib/supabase/server.ts` — exports an **async** `createClient()`
    using `createServerClient`, wired to Next's `cookies()` for reading and
    writing the session cookie. This is what server components/actions use.
  - **Done when:** a server component can call `createClient()` and run
    `supabase.auth.getUser()` without throwing.

- [ ] **Login page + sign-in action**
  Let an admin log in with email/password.
  - `/login` route: email + password inputs, submit button, inline error
    display.
  - A server action `signIn(prevState, formData)` that calls
    `supabase.auth.signInWithPassword({ email, password })`. On success,
    `redirect('/properties')`. On failure, return `{ error: '...' }`.
  - Wire the form to the action with React's `useActionState` so the error
    shows without a full page reload.
  - **Done when:** a real admin account logs in and lands on `/properties`;
    wrong credentials show an inline error, no crash.

- [ ] **Session refresh (`proxy.ts` / middleware)**
  Keep the admin's session alive across requests instead of it silently
  expiring mid-visit.
  - `src/proxy.ts` (Next.js 16+ names this `proxy.ts`; older Next uses
    `middleware.ts` — check what your scaffold generated) exporting an async
    `proxy(request)` function.
  - Build a `createServerClient` inside it wired to read/write the
    *request's* cookies, call `supabase.auth.getUser()` to force a refresh,
    and return a `NextResponse` carrying the refreshed cookies.
  - Set `export const config = { matcher: [...] }` to skip static assets.
  - **Done when:** staying on the app and navigating around for several
    minutes doesn't randomly log you out.

- [ ] **Sign out**
  - A server action calling `supabase.auth.signOut()`, then
    `redirect('/login')`.
  - A button in a shared header/layout.
  - **Done when:** signing out and then visiting `/properties` bounces you
    to `/login`.

### Micah — Track B: Crew PIN Access

- [ ] **Crew session helper (signed cookie)**
  A way to mark "this browser proved it knows the PIN" that can't be
  trivially forged by just setting a cookie by hand.
  - Add env vars `CREW_PIN` (the real passcode) and `CREW_SESSION_SECRET`
    (a long random string, `openssl rand -hex 32`).
  - `src/lib/crew-access/session.ts`:
    - `verifyCrewPin(pin)` — compares the submitted PIN against `CREW_PIN`
      using `crypto.timingSafeEqual` (not `===` — avoids leaking info via
      response-time differences).
    - `createCrewSession()` — sets an `httpOnly` cookie whose value is an
      HMAC-SHA256 (keyed by `CREW_SESSION_SECRET`) of a fixed string.
    - `hasCrewSession()` — recomputes that HMAC and compares it
      (timing-safe) to the cookie's value.
    - `clearCrewSession()` — deletes the cookie.
  - **Done when:** manually editing the cookie to a wrong value fails
    `hasCrewSession()`; going through `createCrewSession()` then calling
    `hasCrewSession()` returns `true`.

- [ ] **PIN entry page + action**
  - `/crew-access` route: a single password-type input, submit button.
  - A server action `submitCrewPin(prevState, formData)` — calls
    `verifyCrewPin`; on success calls `createCrewSession()` then
    `redirect('/properties')`; on failure returns `{ error: '...' }`.
  - Wire with `useActionState`, same pattern as the admin login form.
  - **Done when:** the right PIN unlocks `/properties`; the wrong PIN shows
    an error and does not set the cookie.

- [ ] **Exit crew session**
  - A server action calling `clearCrewSession()`, then
    `redirect('/crew-access')`. A button in the shared header.
  - **Done when:** after exiting, `/properties` bounces back to the PIN
    page.

### Merge Task (pair on this, or whoever's free first)

- [ ] **Access gate**
  One source of truth for "who's allowed to see this," so admin and crew
  logic never has to be checked ad hoc in multiple places.
  - `src/lib/access.ts` — exports `getAccessLevel()`: checks
    `supabase.auth.getUser()` first (return `{ level: 'admin', userId }` if
    present), else checks `hasCrewSession()` (return `{ level: 'crew' }`),
    else `null`.
  - `src/app/properties/layout.tsx` (server component) calls
    `getAccessLevel()` and redirects to `/crew-access` if it's `null`. Admin
    gets extra UI (e.g. "Add property" link) that crew doesn't.
  - **Done when:** visiting any `/properties/*` route while logged out
    redirects; both an admin session and a crew PIN session land inside
    successfully.

---

## Phase 2 — Property List & Map Display

Depends on Phase 1 (routes must be gated). Insert 2-3 test rows directly
into `properties` via the Supabase table editor before starting — you need
something to look at.

### Harlem — Track A: Property List

- [ ] **Test data**
  - In Supabase's Table Editor, insert 2-3 rows into `properties` (name,
    address, and real lat/lng — right-click a spot in Google Maps to copy
    coordinates).
  - **Done when:** `select * from properties` in the SQL editor returns
    your rows.

- [ ] **Property list page**
  - `/properties/page.tsx` (server component): read a `q` search param from
    the URL.
  - Query Supabase for `id, name, address`, ordered by name; if `q` is
    present, filter with `.or('name.ilike.%q%,address.ilike.%q%')`.
  - Render each result as a link to `/properties/[id]`.
  - A simple `<form>` with a GET-method search input tied to the `q` query
    param (no client-side JS needed for this).
  - **Done when:** typing part of a name or address filters the list;
    clicking a result navigates to its detail route (fine if that 404s
    until Micah's map component lands).

- [ ] **Empty/error states**
  - Handle zero results with a clear "No properties found" message.
  - Handle a Supabase query error by showing a message instead of crashing
    the page.
  - **Done when:** searching a nonsense string shows the empty-state
    message, not a blank page.

### Micah — Track B: Property Map

- [ ] **Legend constants**
  One shared definition of the 4 fixed categories so the map and any UI
  never drift apart.
  - `src/lib/legend.ts` — an array of `{ value, label, color }` for
    `do_not_touch` (red), `mow_here` (yellow), `kill_weeds` (orange),
    `cleanup` (green), plus `legendColor()` / `legendLabel()` lookup
    helpers.
  - **Done when:** every place that needs a category's color/label imports
    from this file — no color hex codes hardcoded elsewhere.

- [ ] **Map component**
  Render a property's live satellite map with its regions/pins.
  - Install `maplibre-gl`.
  - A client component that creates a `maplibregl.Map` centered on the
    property's `lat`/`lng`, using a Mapbox satellite style URL (needs
    `NEXT_PUBLIC_MAPBOX_TOKEN`).
  - On the map's `load` event: add a GeoJSON source + fill/outline layers
    for `regions` (colored via the category's `legendColor()`), and a
    source + circle layer for `pins` (same coloring).
  - Click handlers on both layers open a popup showing the category label
    and note.
  - **Done when:** a property with one manually-inserted region (any small
    polygon GeoJSON) and one pin renders both, correctly colored, and
    clicking either shows a popup with its info.

- [ ] **Property detail page**
  - `/properties/[id]/page.tsx` (server component): fetch the property, its
    `regions`, and its `pins` in parallel (`Promise.all`).
  - Render the name, address, a legend key, and the map component with that
    data. Handle a not-found id (`notFound()`).
  - **Done when:** navigating from the list to a specific property shows
    its name, address, legend, and a live map with any test data on it.

---

## Phase 3 — Admin Content Tools

The heaviest phase. Depends on Phase 2 — the map component has to exist
before you can build tools that write to it.

### Harlem — Track A: Add Property Flow

- [ ] **Geocoding helper**
  Turn a typed address into map coordinates automatically — no manual
  pin-dropping to locate a new property.
  - A server-side function that calls the Mapbox Geocoding API
    (`/geocoding/v5/mapbox.places/{address}.json?access_token=...`), parses
    the first result's `center` as `[lng, lat]`, and throws a clear error if
    nothing comes back.
  - **Done when:** calling it with a real street address returns sane
    coordinates; a garbage string returns a handled error, not a crash.

- [ ] **Add Property page + action**
  - `/properties/new/page.tsx` — admin-only (redirect non-admins via
    `getAccessLevel()`); form for name + address.
  - A server action that re-checks admin access (don't trust the page
    redirect alone), calls the geocoder, inserts into `properties`, and
    redirects to the new property's detail page.
  - **Done when:** submitting a real address creates the property and
    lands you on its (still-empty) map.

- [ ] **Edit/archive property**
  - An edit form on the property detail page (admin-only) to update
    name/address (re-geocode if the address changes).
  - A way to archive or delete a property that's no longer serviced — your
    call on soft-delete vs. hard delete.
  - **Done when:** an admin can rename a property and see the change on
    next load.

### Micah — Track B: Region/Pin Editor

This is the actual replacement for "drawing on the binder photo" — take
the time to get the interaction right.

- [ ] **Pin placement**
  - Add an "add pin" mode toggle to the map component (admin-only).
  - While in that mode, a map click captures `lat`/`lng`; show a small
    form/popup to pick a legend category + optional note.
  - On submit, a server action inserts into `pins`; refresh the map's pin
    layer with the new data (no full page reload needed).
  - **Done when:** an admin can click the map, pick "Kill Weeds," add a
    note, and see the new pin appear immediately.

- [ ] **Region drawing**
  - Add a "draw region" mode: capture a sequence of clicked points as a
    polygon (close the loop on double-click or a "finish" button).
  - Same category + note form as pins; a server action inserts into
    `regions` as GeoJSON; refresh the region layer.
  - **Done when:** an admin can draw a rough boundary (e.g. around a flower
    bed), mark it "Do Not Touch," and see it render as a shaded region.

- [ ] **Edit & delete**
  - Clicking an existing region/pin in admin mode opens it for editing
    (category/note) or deletion, backed by update/delete server actions.
  - **Done when:** an admin can change a pin's note and delete a stale
    region, both reflected on the map without a full page reload.

---

## Phase 4 — Polish & Field-Readiness

Depends on Phases 2-3 being functionally complete — this phase makes them
good enough to hand to an actual crew.

### Harlem — Track A: Mobile & Installability

- [ ] **PWA manifest + icons**
  - `public/manifest.webmanifest` (name, short_name, `start_url`,
    background/theme colors, real icon assets — not a generic placeholder).
  - Link the manifest and theme color via the root layout's metadata.
  - **Done when:** "Add to Home Screen" on a phone browser shows the real
    name/icon and opens straight into `/properties`.

- [ ] **Mobile layout pass**
  - Check every screen at a small viewport: search bar, list, map, legend,
    forms. Big enough tap targets, no horizontal scroll, map takes up real
    screen space.
  - **Done when:** you've personally tested the full flow on an actual
    phone (bonus points for testing it outside, standing where a crew
    would).

### Micah — Track B: Error/Empty/Loading States

- [ ] **Loading states**
  - Add `loading.tsx` files (Next.js file convention) for the list and
    detail routes — a simple skeleton or spinner beats a blank screen.
  - **Done when:** throttling your connection in dev tools shows a loading
    state instead of nothing.

- [ ] **Error states**
  - `error.tsx` boundaries where useful; explicit handling for a missing
    property (a real "not found" message, not a crash); required-field
    validation messages on the admin forms.
  - **Done when:** intentionally breaking each of the above (bad id, empty
    form submit, wrong Mapbox token) shows a readable message — never a raw
    stack trace.

---

## Phase 5 — Deploy & Handoff (do together)

- [ ] Point the company's domain at the Vercel deployment
- [ ] Create real admin accounts (retire any test ones)
- [ ] Set the real crew PIN (retire any test PIN)
- [ ] Walk one real property end-to-end with an actual crew member,
      collect feedback, file it as backlog items below

---

## Explicitly Not v1 (backlog — don't build until this list is revisited)

- Crew feedback / field status updates (crew marking something done)
- Offline caching for dead zones
- Admin-editable legend categories (categories are fixed in code for now)
- Native mobile app
