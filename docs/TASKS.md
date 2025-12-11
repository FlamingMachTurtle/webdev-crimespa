# Tasks

**Deadline:** Fri Dec 12 @ 10pm  
**Team:** Eli, Caiden, Charlotte

---

## Dependency Tree

```
WEDNESDAY — Eli builds foundation, others prep UI shells
──────────────────────────────────────────────────────────
Eli          Caiden              Charlotte
 │            │                   │
 A1 ──────────┼───────────────────┼──── API connect
 │            │                   │
 A2,A3,A4 ────┼───────────────────┼──── Fetch all data
 │            │                   │
 F1,F2,F3     C1                  E1,E2,E3,G1
 (form)       (search input)      (filter UI, bios)
 │            │                   │
 ▼            ▼                   ▼
DATA READY ───────────────────────────────────────────────

THURSDAY — Caiden & Charlotte use the data
──────────────────────────────────────────────────────────
              Caiden              Charlotte
               │                   │
              B3                  D1,D2
              (markers+counts)    (crime table)
               │                   │
              C2,C3               
              (geocoding)         
               │                   │
               ▼                   ▼
             MAP DONE            TABLE DONE ──────────────

FRIDAY — Final pieces + polish
──────────────────────────────────────────────────────────
Eli          Caiden              Charlotte
 │            │                   │
 D3           Extras?             Polish
 (addresses)  (X2: crime markers) 
 │            │                   │
 └────────────┴───────────────────┴──── G2: Video @ 6pm
```

---

## Task List

### Eli (16 pts)

| ID | Task | Pts | When | Status |
|----|------|-----|------|--------|
| A1 | API connection dialog | 2 | Wed AM | ✅ Done |
| A2 | Fetch neighborhoods | 2 | Wed AM | ✅ Done |
| A3 | Fetch crime codes | 3 | Wed PM | ✅ Done |
| A4 | Fetch incidents | 3 | Wed PM | ✅ Done |
| F1 | New incident form UI | 1 | Wed PM | ✅ Done |
| F2 | Form required fields | 1 | Wed PM | ✅ Done |
| F3 | POST to API | 1 | Wed Eve | ✅ Done |
| D3 | Address lookup | 3 | Fri AM | To Do |

**Wed goal:** Finish A1-A4, F1-F3 (13 pts) ✅ **COMPLETE**  
**Fri goal:** D3 after Charlotte's table works (3 pts)

---

### Caiden (8 pts)

| ID | Task | Pts | When | Status |
|----|------|-----|------|--------|
| C1 | Location search input | 2 | Wed | To Do |
| B3 | Markers + crime counts | 3 | Thu | To Do |
| C2 | Nominatim geocoding | 2 | Thu | To Do |
| C3 | Center map on result | 1 | Thu | To Do |

**Wed:** Build C1 (search input UI only)  
**Thu:** B3, C2, C3 (needs Eli's data)

---

### Charlotte (14 pts)

| ID | Task | Pts | When | Status |
|----|------|-----|------|--------|
| E1 | Date range filter UI | 2 | Wed | To Do |
| E2 | Crime type filter UI | 2 | Wed | To Do |
| E3 | Neighborhood filter UI | 1 | Wed | To Do |
| G1 | Team bios + photos | 1 | Wed | To Do |
| D1 | Display crime table | 3 | Thu | To Do |
| D2 | Table columns | 4 | Thu | To Do |
| - | Filter logic | 0 | Thu | To Do |

**Wed:** E1-E3, G1 (UI shells, 6 pts)  
**Thu:** D1, D2, wire up filters (8 pts)

---

### Done (7 pts)

| ID | Task | Pts |
|----|------|-----|
| B1 | Leaflet map render | 2 |
| B2 | Neighborhood boundaries | 2 |
| B4 | Pan/zoom bounds | 3 |

---

### Extras (5 pts each, pick for A grade)

| ID | Task | Best For | Status |
|----|------|----------|--------|
| X1 | Delete incident button | Eli | To Do |
| X2 | Crime location markers | Caiden | To Do |
| X3 | Sortable table columns | Charlotte | To Do |
| X4 | Search within table | Charlotte | To Do |

---

## Schedule

| Day | Eli | Caiden | Charlotte |
|-----|-----|--------|-----------|
| **Wed AM** | A1, A2 | C1 | E1, E2, E3 |
| **Wed PM** | A3, A4, F1, F2 | (help or prep) | G1 |
| **Wed Eve** | F3 | - | - |
| **Thu** | (available) | B3, C2, C3 | D1, D2, filters |
| **Fri AM** | D3 | Extras | Polish |
| **Fri PM** | **All: Video + Submit** |||

---

## Checkpoints

| When | Verify |
|------|--------|
| Wed 6pm | Eli: API fetches all data, form submits |
| Thu 6pm | Table shows data, markers have counts |
| Fri 6pm | All 45 pts working |
| Fri 9pm | Video done |
| Fri 10pm | Submit |

---

## Progress

| Person | Done | Total |
|--------|------|-------|
| Eli | 13 | 16 |
| Caiden | 0 | 8 |
| Charlotte | 0 | 14 |
| Starter | 7 | 7 |
| **Total** | **20** | **45** |
