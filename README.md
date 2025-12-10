# St. Paul Crime SPA

A Single Page Application for viewing and managing St. Paul crime data, built with Vue.js and Leaflet maps.

## Quick Links

- [Assignment Requirements](assignmnet.md) - Full assignment specs and grading rubric
- [Project Tracker](PROJECT_TRACKER.md) - Team task tracking and progress

---

## Project Overview

This application displays crime incident data for St. Paul, Minnesota using:
- **Vue.js 3** - Reactive frontend framework
- **Leaflet** - Interactive mapping
- **Nominatim** - Geocoding (address lookup)
- **Foundation CSS** - Responsive styling
- **Your REST API** - Crime data backend (from Project 3)

---

## Getting Started

### Prerequisites
- Node.js (v16+)
- npm
- Your REST API running (from Project 3)

### Installation

```bash
# Clone the repository
git clone <repo-url>
cd webdev-crimespa

# Install dependencies
npm install

# Start development server
npm run dev
```

The app will be available at `http://localhost:5173` (Vite default).

### Running with Your API

1. Start your REST API server from Project 3
2. Open the app in your browser
3. Enter your API URL in the dialog (e.g., `http://localhost:8000`)
4. Crime data will load automatically

---

## Project Structure

```
webdev-crimespa/
├── index.html              # Entry point
├── package.json            # Dependencies and scripts
├── vite.config.js          # Vite build configuration
│
├── src/
│   ├── main.js             # Vue app initialization
│   └── App.vue             # Main application component
│
├── public/
│   ├── css/
│   │   ├── foundation.css  # Foundation framework
│   │   └── style.css       # Custom styles
│   ├── data/
│   │   └── StPaulDistrictCouncil.geojson  # Neighborhood boundaries
│   └── js/vendor/          # Third-party scripts
│
├── assignmnet.md           # Assignment requirements
├── PROJECT_TRACKER.md      # Team task tracking
└── README.md               # This file
```

---

## Features

### Core Features (Required)
- [ ] REST API integration with crime data
- [ ] Interactive Leaflet map with neighborhood boundaries
- [ ] Neighborhood markers showing crime counts
- [ ] Location search (address ↔ coordinates)
- [ ] Crime incident table with details
- [ ] Filtering by date, crime type, neighborhood
- [ ] New incident submission form
- [ ] About page with team info

### Extra Features (For higher grades)
- [ ] Delete incident functionality
- [ ] Individual crime markers on map
- [ ] Dark mode toggle
- [ ] And more... (see [PROJECT_TRACKER.md](PROJECT_TRACKER.md))

---

## API Endpoints

Your REST API should provide:

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/codes` | List crime codes |
| GET | `/neighborhoods` | List neighborhoods |
| GET | `/incidents` | List crime incidents |
| POST | `/new-incident` | Add new incident |
| DELETE | `/remove-incident` | Remove incident |

---

## Team

| Member | Role | Responsibilities |
|--------|------|------------------|
| TBD | Backend/API | REST integration, data handling |
| TBD | Map/UI | Leaflet, markers, location search |
| TBD | Frontend | Table, filters, About page |

---

## Development

### Available Scripts

```bash
npm run dev      # Start dev server with hot reload
npm run build    # Build for production
npm run preview  # Preview production build
```

### Tech Stack
- **Vue 3** with Composition API
- **Vite** for fast builds
- **Leaflet** for maps
- **Foundation CSS** for layout
- **Nominatim API** for geocoding

---

## Contributing

1. Check [PROJECT_TRACKER.md](PROJECT_TRACKER.md) for available tasks
2. Assign yourself to a task
3. Create a branch for your feature
4. Make changes and test
5. Submit a pull request

---

## Resources

- [Vue.js Docs](https://vuejs.org/guide/)
- [Leaflet Docs](https://leafletjs.com/reference.html)
- [Nominatim API](https://nominatim.org/release-docs/develop/api/Overview/)
- [Foundation CSS](https://get.foundation/sites/docs/)
