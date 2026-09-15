# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Mopac Landscaping is a web app that replaces a physical, single-ring-bound
paper packet the company currently gives landscaping crews. The packet
contains one generic aerial photo per property with a hand-marked legend
(e.g. red = "do not touch", yellow = "mow here") — it goes out of date and
is slow to search through in the field. This app puts the same information
on a live, searchable, always-current map that crews pull up on their phone.

**Status:** no code yet. An earlier scaffold was built to preview the idea
and then intentionally deleted — the architecture below is still the plan.
See `PROJECT_PLAN.md` for the ordered, two-person breakdown of how this
actually gets built.

## Planned Architecture

- **Next.js** (React) web app, mobile-first, PWA-enabled (installable to
  home screen) — this is a responsive web app, not a native iOS/Android app.
- **Supabase**: hosted Postgres database + Auth + Storage. Chosen over a
  self-hosted Postgres + custom backend because it bundles the database,
  auto-generated API, and auth in one free-tier service — no backend
  server to build or host for a project this size.
- **MapLibre GL JS** for map rendering (free, open-source; avoids Mapbox
  GL JS's BSL license) + **Mapbox satellite tiles** as the imagery source
  (free tier comfortably covers the ~60-70 property scale).
- **Mapbox Geocoding API** converts a property's address to map
  coordinates when an admin adds it — no manual pin-placement needed to
  center the map.
- Target host: Vercel, pointed at the company's existing domain (deferred
  until closer to launch).

## Data Model

- `properties` — name, address, coordinates, timestamps.
- `regions` — a drawn polygon on a property's map, tagged with a category
  and an optional note, plus last-updated info.
- `pins` — a single point on the map with the same category/note shape as
  regions, for spot issues that aren't an area.
- Legend categories are **fixed in code for v1** (not admin-editable):
  Do Not Touch (red), Mow Here (yellow), Kill Weeds (orange), Cleanup
  (green). Adding a new category is a small code change, not a settings
  feature.

## Auth Model

Two distinct access levels — do not conflate them:
- **Admin**: individual Supabase Auth accounts (email/password). Full CRUD
  on properties, regions, and pins.
- **Crew**: a single shared PIN/passcode unlocks read-only access to all
  properties. There is no crew Supabase account — no per-user management
  overhead for v1.

**Why crew reads must not go through the Supabase anon/publishable key:**
that key ships in the browser bundle, so if the `properties`/`regions`/`pins`
SELECT policies allowed it, anyone who extracted the key could bypass the
PIN entirely. The earlier scaffold's approach (worth keeping): RLS grants
read/write only to `authenticated` (logged-in admins); crew requests are
verified server-side against a signed session cookie, then served through a
server-only Supabase client using the secret/service-role key (bypasses
RLS) on the crew's behalf. Keep a single server-side helper that decides
"admin" vs "crew" vs neither, rather than checking auth state in multiple
places.

## V1 Scope Boundaries

Explicitly deferred — do not build these unless asked:
- Crew feedback / field status updates (e.g. crew marking a task done)
- Offline caching (assume crews have at least weak signal in the field)
- Admin-editable legend categories (fixed set only, see Data Model)
- Native mobile app

## Common Commands

Not yet applicable — no project has been scaffolded in this repo yet. Once
the Next.js app is (re)created, populate this section with the actual
install/dev/build/lint/test commands.
