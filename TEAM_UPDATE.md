**Subject:** Crime SPA Progress - API Done, Pick Your Tasks

---

Hey Caiden & Charlotte,

I finished the API setup (13 pts) - **data is ready for you to use!** Pull the latest code and pick which tasks you want to tackle.

## ✅ What's Working Now
- API connection dialog + data fetching (codes, neighborhoods, incidents)
- New incident form (all fields, validation, POST)

## 🚀 How to Run

**1. Start the REST API server:**
```bash
cd webdev-rest
npm install  # first time only
node rest_server.mjs
```
Should see: `Now listening on port 8000`

**Important:** Database file (`stpaul_crime.sqlite3`) must be in `db/` folder. Download from [tmarrinan/webdev-rest](https://github.com/tmarrinan/webdev-rest) if missing.

**2. Start the Vue app (in this folder):**
```bash
npm install  # first time only
npm run dev
```

**3. Enter API URL:**
When the dialog pops up, enter: `http://localhost:8000`

You should see console messages confirming data loaded (codes, neighborhoods, incidents).

---

## 🎯 Remaining Work (25 pts) - **Pick Your Tasks**

### Map Features (8 pts total)
- [ ] **C1** (2 pts): Address search input UI
- [ ] **B3** (3 pts): Markers showing crime count per neighborhood
- [ ] **C2** (2 pts): Nominatim geocoding (address → lat/lng)
- [ ] **C3** (1 pt): Center map on search result

### Table + Filters (15 pts total)
- [ ] **D1** (3 pts): Display crime table
- [ ] **D2** (4 pts): All columns (case#, date, incident, etc.)
- [ ] **E1** (2 pts): Date range filter UI
- [ ] **E2** (2 pts): Crime type dropdown filter
- [ ] **E3** (1 pt): Neighborhood dropdown filter
- [ ] **Filter logic**: Wire up filters to API calls

### About Page (2 pts)
- [ ] **G1** (1 pt): Team bios + photos
- [ ] **G2** (1 pt): Demo video (Friday 6-8pm - all of us)

### Extras (5 pts each - optional)
- [ ] **X1**: Delete incident button
- [ ] **X2**: Individual crime location markers
- [ ] **X3**: Sortable table columns
- [ ] **X4**: Search within table results

**Note:** I'll do **D3** (address lookup, 3 pts) Friday morning after the table is working.

---

## 📅 Timeline
- **Thu:** Build your features (table/filters OR map/search)
  - **I'm free after 5pm if anyone needs help with setup/debugging**
- **Fri AM:** Finish + extras
- **Fri 6-8pm:** Record demo video together
- **Fri 10pm:** Submit

---

## 📖 API Details

**Key Endpoints:**
- `GET /codes` → crime types
- `GET /neighborhoods` → neighborhood list
- `GET /incidents?limit=1000&start_date=YYYY-MM-DD&end_date=YYYY-MM-DD&code=110,700&neighborhood=11,14`
- `PUT /new-incident` → add incident (note: PUT not POST!)
- `DELETE /remove-incident` → remove by case_number

**Full API docs:** Check `docs/API.md` or `docs/PROGRESS.md` for implementation hints

---

**Let me know what you want to work on!** Either reply here or just start committing - I'll help debug Thursday if needed.

-Eli
