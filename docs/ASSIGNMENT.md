# Assignment 4: St. Paul Crime SPA

## Grade Scale

| Grade | Points |
|-------|--------|
| C | 45 |
| B | 50 |
| A | 55 |
| A+ | 60 |

---

## Base Features (45 pts)

### 1. REST API Integration — 10 pts

| Feature | Pts |
|---------|-----|
| API connection dialog (URL input, validation) | 2 |
| GET /neighborhoods | 2 |
| GET /codes | 3 |
| GET /incidents (with query params) | 3 |

### 2. Map Display — 10 pts

| Feature | Pts |
|---------|-----|
| Leaflet map centered on St. Paul | 2 |
| Neighborhood boundaries (GeoJSON) | 2 |
| 17 neighborhood markers with crime counts | 3 |
| Pan/zoom constrained to St. Paul | 3 |

### 3. Location Search — 5 pts

| Feature | Pts |
|---------|-----|
| Address or lat/lng input field | 2 |
| Nominatim geocoding (forward/reverse) | 2 |
| Map centers on search result | 1 |

### 4. Crime Table — 10 pts

| Feature | Pts |
|---------|-----|
| Table displaying crime incidents | 3 |
| Columns: case#, date/time, incident, address, neighborhood | 4 |
| Address lookup via Nominatim reverse geocode | 3 |

### 5. Filtering — 5 pts

| Feature | Pts |
|---------|-----|
| Date range (start/end) | 2 |
| Crime type (by code) | 2 |
| Neighborhood selection | 1 |

### 6. New Incident Form — 3 pts

| Feature | Pts |
|---------|-----|
| Form UI | 1 |
| All required fields (case#, date, time, code, incident, grid, neighborhood, block) | 1 |
| POST /new-incident | 1 |

### 7. About Page — 2 pts

| Feature | Pts |
|---------|-----|
| Team member bios with photos | 1 |
| Demo video | 1 |

---

## Extra Features (5 pts each)

| Feature | Pts |
|---------|-----|
| Delete incident (DELETE /remove-incident) | 5 |
| Individual crime markers on map | 5 |
| Marker clustering | 5 |
| Dark mode | 5 |
| Filter presets (save/load) | 5 |
| Export to CSV | 5 |
| Statistics/charts | 5 |
| Responsive/mobile layout | 5 |
| Search within table | 5 |
| Sortable columns | 5 |

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Framework | Vue.js 3 (Composition API) |
| Build | Vite |
| CSS | Foundation |
| Maps | Leaflet |
| Geocoding | Nominatim |

---

## Submission

1. Code pushed to GitHub
2. README with setup instructions
3. Demo video (2-3 min)
4. Team contributions documented

