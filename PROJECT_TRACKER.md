# Project Tracker - St. Paul Crime SPA

## Team Information

| Role | Member | Focus Areas |
|------|--------|-------------|
| **Member 1** | _TBD_ | Backend/API: REST integration, data fetching, new incident form |
| **Member 2** | _TBD_ | Map/UI: Leaflet map, markers, location search, pan/zoom |
| **Member 3** | _TBD_ | Frontend/Polish: Crime table, filters, About page, styling |

## Grade Target

| Target Grade | Points Needed | Status |
|--------------|---------------|--------|
| C | 45 pts (all base) | In Progress |
| B | 50 pts (+1 extra) | - |
| A | 55 pts (+2 extras) | - |
| A+ | 60 pts (+3 extras) | - |

**Current Progress:** `0 / 45` base points | `0 / 15` extra points

---

## Status Legend

| Status | Meaning |
|--------|---------|
| `To Do` | Not started |
| `In Progress` | Currently being worked on |
| `Review` | Done, needs testing/review |
| `Done` | Complete and verified |
| `Blocked` | Cannot proceed (see notes) |

---

## Core Features (45 Points Required)

### REST API Integration (10 pts)

| # | Feature | Pts | Assignee | Status | Notes |
|---|---------|-----|----------|--------|-------|
| 1.1 | Connect to REST API via dialog | 2 | - | To Do | Dialog already in starter code |
| 1.2 | Fetch neighborhoods (`GET /neighborhoods`) | 2 | - | To Do | Store in reactive state |
| 1.3 | Fetch crime codes (`GET /codes`) | 3 | - | To Do | Needed for filters |
| 1.4 | Fetch incidents (`GET /incidents`) | 3 | - | To Do | Initial load of 1000 crimes |

### Map Display (10 pts)

| # | Feature | Pts | Assignee | Status | Notes |
|---|---------|-----|----------|--------|-------|
| 2.1 | Leaflet map rendering | 2 | - | Done | Already in starter |
| 2.2 | Neighborhood boundaries (GeoJSON) | 2 | - | Done | Already in starter |
| 2.3 | Neighborhood markers with crime counts | 3 | - | To Do | Use `neighborhood_markers` array |
| 2.4 | Pan and zoom within bounds | 3 | - | Done | Bounds set in starter |

### Location Search (5 pts)

| # | Feature | Pts | Assignee | Status | Notes |
|---|---------|-----|----------|--------|-------|
| 3.1 | Address/lat-lng input field | 2 | - | To Do | Add to UI |
| 3.2 | Nominatim geocoding integration | 2 | - | To Do | Forward + reverse lookup |
| 3.3 | Center map on search result | 1 | - | To Do | Call `map.leaflet.setView()` |

### Crime Table (10 pts)

| # | Feature | Pts | Assignee | Status | Notes |
|---|---------|-----|----------|--------|-------|
| 4.1 | Display crime incidents in table | 3 | - | To Do | Foundation CSS table styling |
| 4.2 | Show: case#, date/time, incident, address, neighborhood | 4 | - | To Do | All required columns |
| 4.3 | Convert block coords to addresses (Nominatim) | 3 | - | To Do | Batch requests, rate limit |

### Filtering (5 pts)

| # | Feature | Pts | Assignee | Status | Notes |
|---|---------|-----|----------|--------|-------|
| 5.1 | Date range filter (start/end) | 2 | - | To Do | Date picker inputs |
| 5.2 | Crime type filter (by code) | 2 | - | To Do | Multi-select dropdown |
| 5.3 | Neighborhood filter | 1 | - | To Do | Checkboxes or multi-select |

### New Incident Form (3 pts)

| # | Feature | Pts | Assignee | Status | Notes |
|---|---------|-----|----------|--------|-------|
| 6.1 | Input form UI | 1 | - | To Do | Modal or separate section |
| 6.2 | All required fields | 1 | - | To Do | case#, date, time, code, incident, grid, neighborhood, block |
| 6.3 | POST to `/new-incident` | 1 | - | To Do | Handle success/error response |

### About Page (2 pts)

| # | Feature | Pts | Assignee | Status | Notes |
|---|---------|-----|----------|--------|-------|
| 7.1 | Team member bios + photos | 1 | - | To Do | One per member |
| 7.2 | Demo video embedded | 1 | - | To Do | Record after features complete |

---

## Extra Features (5 Points Each)

Choose extras based on team capacity. Need 1 for B, 2 for A, 3 for A+.

| # | Feature | Pts | Assignee | Status | Priority | Notes |
|---|---------|-----|----------|--------|----------|-------|
| E1 | Delete incident button + `DELETE /remove-incident` | 5 | - | To Do | High | Quick win if API supports it |
| E2 | Crime location markers on map | 5 | - | To Do | High | Show individual crimes |
| E3 | Marker clustering | 5 | - | To Do | Medium | Leaflet.markercluster plugin |
| E4 | Dark mode toggle | 5 | - | To Do | Medium | CSS variables for theming |
| E5 | Filter presets (save/load) | 5 | - | To Do | Low | LocalStorage |
| E6 | Export to CSV | 5 | - | To Do | Low | Client-side generation |
| E7 | Statistics dashboard | 5 | - | To Do | Low | Chart.js or similar |
| E8 | Responsive/mobile design | 5 | - | To Do | Medium | Foundation Grid helpers |
| E9 | Search within table results | 5 | - | To Do | Medium | Text filter |
| E10 | Sortable table columns | 5 | - | To Do | Medium | Click to sort |

---

## Progress Tracking

### Sprint 1: Core Infrastructure
- [ ] REST API connection working
- [ ] All data endpoints integrated
- [ ] Basic crime table displaying

### Sprint 2: Map Features  
- [ ] Neighborhood markers with counts
- [ ] Location search functional
- [ ] Map-table interaction

### Sprint 3: Filters & Forms
- [ ] All filters working
- [ ] New incident form complete
- [ ] Data refreshes after changes

### Sprint 4: Polish & Extras
- [ ] About page complete
- [ ] Extra features implemented
- [ ] Demo video recorded
- [ ] Code cleaned and documented

---

## Quick Reference

### Key Files to Edit
- `src/App.vue` - Main application component
- `public/css/style.css` - Custom styles
- `index.html` - Page structure (minimal changes needed)

### Commands
```bash
npm install          # Install dependencies
npm run dev          # Start dev server (Vite)
npm run build        # Production build
```

### API Endpoints
```
GET  /codes              # Crime codes list
GET  /neighborhoods      # Neighborhood list
GET  /incidents          # Crime data (with filters)
POST /new-incident       # Add new crime
DELETE /remove-incident  # Remove crime
```

---

## Meeting Notes

_Add notes from team meetings here_

### Meeting 1 - _Date_
- Attendees: 
- Discussed:
- Action items:

---

## Questions / Blockers

_Track questions for instructor or blockers here_

| Issue | Owner | Status | Resolution |
|-------|-------|--------|------------|
| _Example: How to handle Nominatim rate limits?_ | - | Open | - |

