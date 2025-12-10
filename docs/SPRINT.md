# Sprint Plan

**Deadline:** Friday, December 12, 2024 @ 10:00 PM  
**Duration:** 2 days (Wed-Fri)  
**Team:** 3 members

---

## Schedule

### Wednesday, December 10

| Time      | M1 (Backend/API)                        | M2 (Map/UI)                        | M3 (Frontend)                               |
| --------- | --------------------------------------- | ---------------------------------- | ------------------------------------------- |
| Morning   | API connection dialog                   | Location search input UI           | Filter UI shells (date, type, neighborhood) |
| Afternoon | Fetch: neighborhoods, codes, incidents  | Nominatim geocoding integration    | About page (bios, placeholder for video)    |
| Evening   | **Checkpoint:** All data fetching works | **Checkpoint:** Search centers map | **Checkpoint:** Filter inputs render        |

### Thursday, December 11

| Time      | M1 (Backend/API)                        | M2 (Map/UI)                           | M3 (Frontend)                                               |
| --------- | --------------------------------------- | ------------------------------------- | ----------------------------------------------------------- |
| Morning   | New incident form UI                    | Neighborhood markers (names + counts) | Crime table (display incidents)                             |
| Afternoon | Form validation + POST                  | Marker popups on click                | Table columns: case#, date, incident, address, neighborhood |
| Evening   | **Checkpoint:** Can submit new incident | **Checkpoint:** Markers show counts   | **Checkpoint:** Table displays all data                     |

### Friday, December 12

| Time      | M1 (Backend/API)                   | M2 (Map/UI)                 | M3 (Frontend)                         |
| --------- | ---------------------------------- | --------------------------- | ------------------------------------- |
| Morning   | Address lookup (Nominatim reverse) | Help with table/integration | Filter logic (call API with params)   |
| Afternoon | Bug fixes, extras if time          | Bug fixes, extras if time   | Bug fixes, extras if time             |
| 6-8 PM    | **All:** Final testing             | **All:** Record demo video  | **All:** Update About page with video |
| 8-10 PM   | **All:** Final commit and submit   |                             |                                       |

---

## Task Board

### Core Features (45 pts required)

| ID                            | Task                             | Pts | Owner | Status | Due     |
| ----------------------------- | -------------------------------- | --- | ----- | ------ | ------- |
| **API Integration (10 pts)**  |                                  |     |       |        |
| A1                            | API connection dialog            | 2   | M1    | To Do  | Wed AM  |
| A2                            | Fetch neighborhoods              | 2   | M1    | To Do  | Wed PM  |
| A3                            | Fetch crime codes                | 3   | M1    | To Do  | Wed PM  |
| A4                            | Fetch incidents                  | 3   | M1    | To Do  | Wed PM  |
| **Map Display (10 pts)**      |                                  |     |       |        |
| B1                            | Leaflet map render               | 2   | -     | Done   | -       |
| B2                            | Neighborhood boundaries          | 2   | -     | Done   | -       |
| B3                            | Neighborhood markers + counts    | 3   | M2    | To Do  | Thu PM  |
| B4                            | Pan/zoom within bounds           | 3   | -     | Done   | -       |
| **Location Search (5 pts)**   |                                  |     |       |        |
| C1                            | Address/lat-lng input            | 2   | M2    | To Do  | Wed AM  |
| C2                            | Nominatim geocoding              | 2   | M2    | To Do  | Wed PM  |
| C3                            | Center map on result             | 1   | M2    | To Do  | Wed PM  |
| **Crime Table (10 pts)**      |                                  |     |       |        |
| D1                            | Display crime list               | 3   | M3    | To Do  | Thu AM  |
| D2                            | Required columns                 | 4   | M3    | To Do  | Thu PM  |
| D3                            | Address lookup (reverse geocode) | 3   | M1    | To Do  | Fri AM  |
| **Filtering (5 pts)**         |                                  |     |       |        |
| E1                            | Date range filter                | 2   | M3    | To Do  | Wed AM  |
| E2                            | Crime type filter                | 2   | M3    | To Do  | Wed AM  |
| E3                            | Neighborhood filter              | 1   | M3    | To Do  | Wed AM  |
| **New Incident Form (3 pts)** |                                  |     |       |        |
| F1                            | Form UI                          | 1   | M1    | To Do  | Thu AM  |
| F2                            | Required fields                  | 1   | M1    | To Do  | Thu AM  |
| F3                            | POST to API                      | 1   | M1    | To Do  | Thu PM  |
| **About Page (2 pts)**        |                                  |     |       |        |
| G1                            | Team bios + photos               | 1   | M3    | To Do  | Wed PM  |
| G2                            | Demo video                       | 1   | All   | To Do  | Fri Eve |

### Extra Features (5 pts each, pick for higher grade)

| ID  | Task                   | Pts | Owner | Status | Priority |
| --- | ---------------------- | --- | ----- | ------ | -------- |
| X1  | Delete incident button | 5   | M1    | To Do  | High     |
| X2  | Crime location markers | 5   | M2    | To Do  | High     |
| X3  | Sortable table columns | 5   | M3    | To Do  | Medium   |
| X4  | Search within results  | 5   | M3    | To Do  | Medium   |

---

## Dependencies

```
A1 (API connect)
 └─► A2, A3, A4 (data fetch)
      ├─► B3 (markers need incident counts)
      ├─► D1, D2 (table needs incidents)
      └─► E1-E3 (filters need codes/neighborhoods)
```

**Blockers:**
- M2 and M3 cannot complete data-dependent tasks until M1 finishes A1-A4
- Demo video (G2) requires all features working

---

## Quick Reference

### Key Files
| File                                        | Purpose                         |
| ------------------------------------------- | ------------------------------- |
| `src/App.vue`                               | Main component (all UI + logic) |
| `public/css/style.css`                      | Custom styles                   |
| `public/data/StPaulDistrictCouncil.geojson` | Neighborhood boundaries         |

### Commands
```bash
npm install     # First time setup
npm run dev     # Start dev server (localhost:5173)
npm run build   # Production build
```

### API Endpoints
```
GET  /codes           → [{code, type}, ...]
GET  /neighborhoods   → [{id, name}, ...]
GET  /incidents?start_date=&end_date=&code=&neighborhood=&limit=
POST /new-incident    → {case_number, date, time, code, ...}
```

### Nominatim
```
Forward: https://nominatim.openstreetmap.org/search?q=ADDRESS&format=json
Reverse: https://nominatim.openstreetmap.org/reverse?lat=LAT&lon=LNG&format=json
```

---

## Checkpoints

| When        | What to Verify                                        |
| ----------- | ----------------------------------------------------- |
| Wed Evening | API returns data, search works, filter UI exists      |
| Thu Evening | Table shows crimes, markers have counts, form submits |
| Fri 6pm     | All 45 pts working, start video                       |
| Fri 9pm     | Video uploaded, final testing                         |
| Fri 10pm    | Submit                                                |

---

## Grade Targets

| Grade | Points | Requirements      |
| ----- | ------ | ----------------- |
| C     | 45     | All core features |
| B     | 50     | Core + 1 extra    |
| A     | 55     | Core + 2 extras   |
| A+    | 60     | Core + 3 extras   |

