# Assignment 4: St. Paul Crime SPA

## Overview

Build a Single Page Application (SPA) to display St. Paul crime data using:
- **Vue.js** - Frontend framework
- **Leaflet API** - Interactive maps
- **Nominatim API** - Geocoding (address ↔ coordinates)
- **Your REST API** - Crime data from Project 3

---

## Grade Breakdown

| Grade | Points Required | Description |
|-------|-----------------|-------------|
| **C** | 45 points | All base features completed |
| **B** | 50 points | Base + 1 extra feature |
| **A** | 55 points | Base + 2 extra features |
| **A+** | 60+ points | Base + 3+ extra features |

**Total possible:** 60 points (45 base + 15 from extras)

---

## Base Features (45 Points - Required for C)

### 1. REST API Integration (10 pts)

| Feature | Points | Description |
|---------|--------|-------------|
| Connect to REST API | 2 | Dialog prompts user for API URL, validates and connects |
| Fetch neighborhoods | 2 | GET `/neighborhoods` - retrieve all 17 neighborhoods |
| Fetch crime codes | 3 | GET `/codes` - retrieve all crime type codes |
| Fetch incidents | 3 | GET `/incidents` - retrieve crime data with query params |

### 2. Map Display (10 pts)

| Feature | Points | Description |
|---------|--------|-------------|
| Leaflet map rendering | 2 | Display interactive map centered on St. Paul |
| Neighborhood boundaries | 2 | Show district council boundaries from GeoJSON |
| Neighborhood markers | 3 | 17 markers showing neighborhood names + crime counts |
| Pan and zoom | 3 | User can navigate map; stays within St. Paul bounds |

### 3. Location Search (5 pts)

| Feature | Points | Description |
|---------|--------|-------------|
| Address/lat-lng input | 2 | Input field for location search |
| Nominatim geocoding | 2 | Convert address → coordinates (or reverse) |
| Map updates | 1 | Center map on searched location |

### 4. Crime Table (10 pts)

| Feature | Points | Description |
|---------|--------|-------------|
| Display crime list | 3 | Table showing crime incidents |
| Required columns | 4 | Case number, date/time, incident type, address, neighborhood |
| Address lookup | 3 | Convert block coordinates to readable addresses via Nominatim |

### 5. Filtering (5 pts)

| Feature | Points | Description |
|---------|--------|-------------|
| Date range filter | 2 | Filter crimes by start/end date |
| Crime type filter | 2 | Filter by incident type (codes) |
| Neighborhood filter | 1 | Filter by neighborhood(s) |

### 6. New Incident Form (3 pts)

| Feature | Points | Description |
|---------|--------|-------------|
| Input form | 1 | Form to submit new crime incident |
| Required fields | 1 | Case number, date, time, code, incident, police_grid, neighborhood_number, block |
| POST to API | 1 | Submit data via `POST /new-incident` |

### 7. About Page (2 pts)

| Feature | Points | Description |
|---------|--------|-------------|
| Team bios | 1 | Photo and short bio for each team member |
| Demo video | 1 | Embedded video demonstrating app functionality |

---

## Extra Features (5 Points Each)

Implement additional features beyond the base requirements for higher grades:

| Feature | Points | Description |
|---------|--------|-------------|
| Delete incident UI | 5 | Button to delete incidents via `DELETE /remove-incident` |
| Crime location markers | 5 | Show individual crime markers on map (with popups) |
| Marker clustering | 5 | Group nearby markers when zoomed out |
| Dark mode toggle | 5 | Switch between light/dark themes |
| Filter presets | 5 | Save and load custom filter configurations |
| Export to CSV | 5 | Download filtered crime data as CSV |
| Statistics dashboard | 5 | Charts/graphs showing crime trends |
| Responsive design | 5 | Mobile-friendly layout |
| Search within results | 5 | Text search across displayed crimes |
| Sort table columns | 5 | Click column headers to sort |

---

## API Endpoints Reference

Your REST API from Project 3 should support:

```
GET  /codes                  - List all crime codes
GET  /neighborhoods          - List all neighborhoods  
GET  /incidents              - List incidents (supports query params)
POST /new-incident           - Create new incident
DELETE /remove-incident      - Delete incident by case number
```

### Query Parameters for `/incidents`:

| Parameter | Type | Description |
|-----------|------|-------------|
| `start_date` | string | Filter by start date (YYYY-MM-DD) |
| `end_date` | string | Filter by end date (YYYY-MM-DD) |
| `code` | string | Filter by crime code(s), comma-separated |
| `neighborhood` | string | Filter by neighborhood number(s), comma-separated |
| `limit` | number | Max number of results (default: 1000) |

---

## External APIs

### Nominatim (OpenStreetMap Geocoding)

**Forward geocoding** (address → coordinates):
```
GET https://nominatim.openstreetmap.org/search?q={address}&format=json
```

**Reverse geocoding** (coordinates → address):
```
GET https://nominatim.openstreetmap.org/reverse?lat={lat}&lon={lng}&format=json
```

> **Note:** Respect Nominatim usage policy - max 1 request/second, include User-Agent header

### Leaflet Map

- Tile layer: OpenStreetMap
- Bounds: St. Paul city limits
- Min zoom: 11, Max zoom: 18

---

## Technical Requirements

- **Framework:** Vue.js 3 (Composition API)
- **Build tool:** Vite
- **CSS Framework:** Foundation CSS
- **Map:** Leaflet.js
- **Data:** GeoJSON for district boundaries

### File Structure

```
├── index.html          # Entry point
├── src/
│   ├── main.js         # Vue app initialization
│   └── App.vue         # Main component
├── public/
│   ├── css/            # Stylesheets
│   ├── js/vendor/      # Third-party scripts
│   └── data/           # GeoJSON data
├── package.json        # Dependencies
└── vite.config.js      # Vite configuration
```

---

## Submission Requirements

1. **Code:** Push all source code to GitHub repository
2. **README:** Document setup instructions and features
3. **Demo video:** Record 2-3 minute walkthrough of functionality
4. **Team contributions:** Document who worked on what

---

## Grading Rubric Summary

| Category | Points |
|----------|--------|
| REST API Integration | 10 |
| Map Display | 10 |
| Location Search | 5 |
| Crime Table | 10 |
| Filtering | 5 |
| New Incident Form | 3 |
| About Page | 2 |
| **Base Total** | **45** |
| Extra Features | 5 each (max 15) |
| **Maximum Total** | **60** |

---

## Resources

- [Vue.js Documentation](https://vuejs.org/guide/introduction.html)
- [Leaflet Documentation](https://leafletjs.com/reference.html)
- [Nominatim API](https://nominatim.org/release-docs/develop/api/Overview/)
- [Foundation CSS](https://get.foundation/sites/docs/)

