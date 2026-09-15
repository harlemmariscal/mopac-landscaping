# Mopac Landscaping — Build Plan (Harlem & Micah)

This splits the app into phases. Do them **in order** — each phase depends
on the one before it. Within a phase, the two tracks can usually be built
in parallel, but each track is a full vertical slice (UI → server logic →
database), not a frontend/backend split. Swap who takes which track any
time; the point is both of you touch the whole stack.

See `CLAUDE.md` for the architecture/data model/auth decisions referenced
below.

---

## Phase 0 — Foundation (do together)

Small and sequential enough that splitting it doesn't help.

- [ ] Create the Supabase project, grab the API keys
- [ ] Scaffold the Next.js app (TypeScript, Tailwind, App Router)
- [ ] Write the SQL migration: `properties`, `regions`, `pins` tables, the
      legend category enum, RLS policies (see Auth Model in `CLAUDE.md` —
      admin-only reads via RLS, crew reads via a server-only client)
- [ ] Apply the migration to Supabase, confirm tables exist
- [ ] Set up `.env.local.example` documenting every required env var
- [ ] Get a blank page building and deploying to Vercel

**Done when:** an empty Next.js app is deployed and connected to a live
Supabase database with the schema in place.

---

## Phase 1 — Access Control

Nothing past this point is reachable without a working login/PIN gate, so
it goes first.

- **Track A — Admin login:** login page UI, Supabase Auth sign-in server
  action, session-refresh middleware/proxy, sign-out.
- **Track B — Crew PIN:** PIN entry page UI, PIN-check server action,
  signed session cookie (HMAC'd with a server secret — don't just set a
  plain "logged in" cookie), sign-out/exit.

**Merge point:** a shared `getAccessLevel()` server helper that returns
`admin`, `crew`, or `null`, used by a protected layout that gates every
`/properties` route.

**Done when:** visiting the app with no session redirects to a choice of
admin login or crew PIN, and each path correctly unlocks the app.

---

## Phase 2 — Property List & Map Display

Depends on Phase 1 (routes need to be gated) and Phase 0 (schema needs
data — insert 2-3 test properties directly in Supabase to build against).

- **Track A — Property list:** searchable list page, server-side query
  against `properties`, links into the detail page.
- **Track B — Property map:** the map component (MapLibre GL JS + Mapbox
  satellite tiles), rendering a property's `regions`/`pins` with legend
  colors, tap-to-see-note interaction, the fixed legend key.

**Done when:** both of you can log in (or PIN in) and browse from the
property list into a property's map, seeing the manually-inserted test
data render correctly.

---

## Phase 3 — Admin Content Tools

The heaviest phase. Depends on Phase 2 (map component must exist before
you can build an editor for it).

- **Track A — Add Property flow:** form UI, address → coordinates via the
  Mapbox Geocoding API, insert into `properties`.
- **Track B — Region/pin editor:** draw a region (polygon) or drop a pin
  directly on the map, assign it a legend category + note, save/edit/
  delete. This is the part that replaces "someone drawing on the binder
  photo" — take the time to get the interaction right.

**Done when:** an admin can add a brand-new property and mark it up
entirely through the browser, with zero manual database edits.

---

## Phase 4 — Polish & Field-Readiness

Depends on Phases 2-3 being functionally complete — this phase makes them
good enough to hand to an actual crew.

- **Track A — Mobile & installability:** PWA manifest + icons, "Add to
  Home Screen" flow, a real pass on mobile layout/touch targets (this is
  used on a phone in the field, not a laptop).
- **Track B — Error/empty/loading states:** what a crew member sees on a
  bad connection, an empty property, a failed geocode, etc. Also basic
  form validation on the admin side.

**Done when:** you'd hand a crew member a phone with this open and not be
nervous about it.

---

## Phase 5 — Deploy & Handoff (do together)

- [ ] Point the company's domain at the Vercel deployment
- [ ] Create real admin accounts (not test ones)
- [ ] Set the real crew PIN
- [ ] Walk one real property end-to-end with an actual crew member,
      collect feedback

---

## Explicitly Not v1 (backlog — don't build until this list is revisited)

- Crew feedback / field status updates (crew marking something done)
- Offline caching for dead zones
- Admin-editable legend categories (categories are fixed in code for now)
- Native mobile app
