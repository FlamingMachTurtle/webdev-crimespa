# Email to Team

---

**Subject:** Crime SPA Plan - Due Fri 10pm

---

Hey Caiden and Charlotte,

Here's the plan for our crime SPA project. Due Friday at 10pm.

## TL;DR

- **Wed:** I'll do all the API setup. You two build UI shells.
- **Thu:** Data will be ready. You plug it in and build your features.
- **Fri:** Final pieces + record video.

## The Problem

Most features need the API data to work. I'll handle that today so you're not blocked tomorrow.

## Schedule

| Day | Eli | Caiden | Charlotte |
|-----|-----|--------|-----------|
| Wed | API + form (13 pts) | Search input (2 pts) | Filters + bios (6 pts) |
| Thu | Available to help | Markers + geocoding (6 pts) | Crime table (7 pts) |
| Fri | Address lookup (3 pts) | Extras/polish | Polish |
| Fri 6pm | **All: Record video** | | |

## Your Tasks

**Caiden (8 pts total):**
```
Wed: C1 - Build the location search input field (just the UI)
Thu: B3 - Add markers to map showing crime counts per neighborhood
Thu: C2 - Hook up Nominatim geocoding (address → coordinates)
Thu: C3 - Make map center on search results
```

**Charlotte (14 pts total):**
```
Wed: E1,E2,E3 - Build filter UI (date picker, dropdowns for crime type/neighborhood)
Wed: G1 - Add our bios to the About page
Thu: D1 - Create the crime table (empty at first)
Thu: D2 - Add all columns (case#, date, incident, address, neighborhood)
Thu: Wire up filters to actually filter the data
```

## What You Can Do Today

Even before the data is ready:

**Caiden:**
- Build the search input field
- Read up on Nominatim API (docs/API.md)
- Look at how Leaflet markers work

**Charlotte:**
- Build the filter dropdowns (empty options for now)
- Build the table HTML structure
- Add our photos and bios to About page

## Repo

Clone: `https://github.com/FlamingMachTurtle/webdev-crimespa.git`

```bash
npm install
npm run dev
```

Check `docs/TASKS.md` for the full task list and `docs/dashboard.html` for a visual plan.

## Checkpoints

- Wed 6pm: I'll push working API + form
- Thu 6pm: Table and markers should work
- Fri 6pm: Everything done, start video
- Fri 10pm: Submit

Let me know if you have questions.

-Eli

