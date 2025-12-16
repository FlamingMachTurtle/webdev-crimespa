# St. Paul Crime SPA

## Setup

```bash
npm install
npm run dev
```

Open `http://localhost:5173`

## Features

### Map Interface
- **Interactive Leaflet Map** - Pan and zoom around St. Paul
- **Coordinate Display & Input** - View and manually enter latitude/longitude coordinates
  - Coordinates update automatically as you move the map
  - Enter custom coordinates to jump to a specific location
  - Click "Go" button to navigate to entered coordinates
- **Smart Address Search** - Search by address, street, or neighborhood
  - **Auto-updating**: Address bar automatically updates as you move the map (with 2.5s debounce to prevent API overload)
  - **Visual feedback**: Green pulse animation when reverse geocoding is active
  - **Autocomplete suggestions**: Properly aligned dropdown with categorized results (Neighborhood, Street)
  - **Smart behavior**: Won't override your manual searches - only updates when idle
  - Clear button (✕) to remove search marker and reset
- **Neighborhood Markers** - Visual indicators showing crime counts per neighborhood
- **Crime Location Markers** - Click any crime to see its precise location on the map

### Search Behavior Details
- **After panning/zooming**: Wait 2.5 seconds → Reverse geocode → Update address bar
- **After searching**: Your search is preserved, auto-update pauses
- **After clearing**: Auto-update resumes immediately
- **Typing in search**: Auto-update pauses while you have an active search

### Filtering System
A comprehensive, clean filtering interface with:
- **Date Filtering**
  - All dates, Today, Last 7 days, Last 30 days, Custom range
- **Time Filtering** *(NEW)*
  - **Time of Day**: Morning (6am-12pm), Afternoon (12pm-6pm), Evening (6pm-10pm), Night (10pm-6am)
  - **Schedule**: Business Hours (9am-5pm), Weekdays, Weekends
- **Crime Type Filtering**
  - Multi-select panel with categories: Violent, Property, Other
  - Select individual types or entire categories
- **Neighborhood Filtering**
  - Filter by specific St. Paul neighborhood
- **Results Badge**: Live count of filtered incidents
- **Clear All Button**: One-click reset of all filters

### Design Improvements
- Modern gradient filter bar with organized grid layout
- Clean labeled filter groups for quick identification
- Active filter highlighting with blue accents
- Smooth animations when opening/closing panels
- Responsive grid that adapts to screen size


