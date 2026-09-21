# Presentations — Combined Ligue 1 View

A page, separate from the Wix-hosted fantasy-coach.fr, that combines
info/data from the sibling [`DNP`](../DNP) (`l1.dnp.fantasy-coach.fr`) and
[`compos`](../compos) (`l1.compos.fantasy-coach.fr`) projects into a single
view: one match per line, home team on top and away team on the bottom of
a shared pitch diagram (probable lineup, from compos), each team's header
paired with a simplified DNP capsule of its unavailable players.

Same $0 hosting pattern as the sibling projects: a static page deployed via
GitHub Actions to GitHub Pages, on its own subdomain of fantasy-coach.fr.

## Architecture

```
DNP Worker + KV        compos Worker + KV
(dnp-l1-cache)          (compos-l1-cache)
      \                        /
       \                      /
        client-side fetch (CORS, GET only)
                |
                v
frontend/index.html (static, GitHub Pages, l1.presentations.fantasy-coach.fr)
```

This project has **no backend of its own** — no new Cloudflare Worker, no
new KV namespace, no new Apps Script project. The frontend fetches directly,
client-side, from the two existing Workers below, which already cache
everything in their own KV. That read traffic is free: Cloudflare's Workers
KV free tier caps **1,000 "put" operations per day for the whole account**
(shared with DNP and compos), and this page only ever issues GETs — only
puts count against that quota.

- DNP: `https://dnp-l1-cache.fantasycoachfr.workers.dev`
  — `?meta=1`, `?journee=<name>`, `?risqueSuspension=<name>`,
  `?matchsSelection=1`
- compos: `https://compos-l1-cache.fantasycoachfr.workers.dev`
  — `?meta=1`, `?journee=<name>`

Both already send permissive CORS headers, so the frontend can call them
directly from the browser.

## Frontend

`frontend/index.html` supports the same `?api=`-style override pattern as
the sibling frontends, for local/dev testing — split into two params here
since this page combines two data sources:

- `?api_dnp=<url>` overrides the DNP Worker base (default
  `https://dnp-l1-cache.fantasycoachfr.workers.dev`).
- `?api_compos=<url>` overrides the compos Worker base (default
  `https://compos-l1-cache.fantasycoachfr.workers.dev`).

Neither override is persisted, so production use is unaffected.

### How matches and capsules are built

- A journée picker (same UX as compos') drives one `?journee=<n>` request
  to compos and one `?journee=Journée <n>` request to DNP (DNP's journée
  labels are `"Journée N"`; compos' are bare numbers — both fetched by the
  same picker selection).
- compos' API is per-team, not per-match: each team row carries its own
  `fixture` (`opponent`, `isHome`, `kickoff`). `buildMatches_()` pairs two
  teams sharing a reciprocal opponent into one match, client-side.
- Matches are grouped by calendar day (Europe/Paris) and sorted
  chronologically by kickoff, same grouping convention as compos' own
  `dayGroupKey_`.
- Each match renders as one continuous pitch: the home team's own goal at
  the top (goalkeeper first, attack nearest the shared halfway line), the
  away team's own goal at the bottom (mirrored) — built by reusing compos'
  pitch-drawing logic (`parseFormationLines`, shirt icons, `TEAM_COLORS`)
  with an added orientation flag.
- The DNP capsule next to each team's header simplifies DNP's own row
  format (icon + reason + expected return) down to icon + "Lastname F.",
  covering every DNP category (blessure/suspendu/hors_groupe/personnel/
  transfert/incertain/disponible). The whole capsule links to
  `https://l1.dnp.fantasy-coach.fr/` — DNP's frontend has no `?journee=`
  deep-link support today, so it lands on the homepage rather than the
  matching journée/team.

## Shared nav/ads/footer

Like the sibling projects, this page pulls in
[`fc-shared`](https://github.com/lionelleboiteux/fc-shared) (nav, ads,
feedback widget, GA, pageview tracking), pinned to a commit SHA — never
`@main`, so a change there can't silently break this page.

`fc-shared/nav.js` doesn't have an entry for this project yet — its menu
placement/label is still to be decided, so `<fc-nav current="presentations">`
won't show a matching highlighted link until that's added there.

## Hosting

1. Repo Settings > Pages > Source: **GitHub Actions** (the included
   `.github/workflows/pages.yml` handles the rest on every push to `main`).
2. **Manual step**: add a DNS **CNAME** record: `l1.presentations` →
   `<github-username>.github.io` (same as was done for
   `l1.dnp.fantasy-coach.fr` and `l1.compos.fantasy-coach.fr`). Not done by
   this repo/workflow — the domain owner needs to add it directly.
3. Once DNS propagates and a deploy has run, enable the custom domain in
   Repo Settings > Pages (it should pick up the `CNAME` file automatically
   once a build has run; confirm/re-save it there after the first deploy).
4. https://l1.presentations.fantasy-coach.fr should then serve the page.

## Out of scope (for now)

- `fc-shared/nav.js` menu placement/label for this project.
- Deep-linking the DNP capsule to a specific journée/team (would need
  `?journee=`/`?equipe=` support added to DNP's own frontend).
- Supabase — only revisit if real relational joins across DNP/compos data
  are needed later.
