# API Reference

## REST API (Your Server)

Base URL: `http://localhost:8000` (or as configured)

---

### GET /codes

Returns all crime type codes.

**Request:**
```
GET /codes
```

**Response:**
```json
[
  {"code": 110, "type": "Murder, Non Negligent Manslaughter"},
  {"code": 120, "type": "Murder, Manslaughter By Negligence"},
  {"code": 210, "type": "Rape"},
  {"code": 220, "type": "Robbery"},
  ...
]
```

---

### GET /neighborhoods

Returns all 17 St. Paul neighborhoods.

**Request:**
```
GET /neighborhoods
```

**Response:**
```json
[
  {"id": 1, "name": "Conway/Battlecreek/Highwood"},
  {"id": 2, "name": "Greater East Side"},
  {"id": 3, "name": "West Side"},
  ...
]
```

---

### GET /incidents

Returns crime incidents with optional filters.

**Request:**
```
GET /incidents
GET /incidents?start_date=2024-01-01&end_date=2024-12-31
GET /incidents?code=110,120&neighborhood=1,2,3&limit=500
```

**Query Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| start_date | string | YYYY-MM-DD format |
| end_date | string | YYYY-MM-DD format |
| code | string | Comma-separated crime codes |
| neighborhood | string | Comma-separated neighborhood IDs |
| limit | number | Max results (default: 1000) |

**Response:**
```json
[
  {
    "case_number": "24-123456",
    "date": "2024-03-15",
    "time": "14:30:00",
    "code": 220,
    "incident": "Robbery",
    "police_grid": 87,
    "neighborhood_number": 4,
    "block": "2XX UNIVERSITY AVE W"
  },
  ...
]
```

---

### POST /new-incident

Creates a new crime incident.

**Request:**
```
POST /new-incident
Content-Type: application/json

{
  "case_number": "24-999999",
  "date": "2024-12-10",
  "time": "09:00:00",
  "code": 220,
  "incident": "Robbery",
  "police_grid": 87,
  "neighborhood_number": 4,
  "block": "1XX MAIN ST"
}
```

**Required Fields:**

| Field | Type | Description |
|-------|------|-------------|
| case_number | string | Unique case ID |
| date | string | YYYY-MM-DD |
| time | string | HH:MM:SS |
| code | number | Crime code |
| incident | string | Crime description |
| police_grid | number | Grid number |
| neighborhood_number | number | 1-17 |
| block | string | Street address block |

**Response:**
```json
{"message": "success"}
```

---

### DELETE /remove-incident

Deletes a crime incident by case number.

**Request:**
```
DELETE /remove-incident
Content-Type: application/json

{
  "case_number": "24-999999"
}
```

**Response:**
```json
{"message": "success"}
```

---

## Nominatim API (Geocoding)

Free geocoding service from OpenStreetMap.

**Rate Limit:** 1 request per second  
**Required Header:** User-Agent

---

### Forward Geocoding (Address → Coordinates)

**Request:**
```
GET https://nominatim.openstreetmap.org/search?q=200+University+Ave+St+Paul+MN&format=json
```

**Response:**
```json
[
  {
    "lat": "44.9537",
    "lon": "-93.0900",
    "display_name": "200 University Avenue West, St Paul, MN 55103, USA",
    ...
  }
]
```

**Usage in Vue:**
```javascript
async function geocodeAddress(address) {
  const url = `https://nominatim.openstreetmap.org/search?q=${encodeURIComponent(address)}&format=json`;
  const response = await fetch(url, {
    headers: { 'User-Agent': 'StPaulCrimeSPA/1.0' }
  });
  const data = await response.json();
  if (data.length > 0) {
    return { lat: parseFloat(data[0].lat), lng: parseFloat(data[0].lon) };
  }
  return null;
}
```

---

### Reverse Geocoding (Coordinates → Address)

**Request:**
```
GET https://nominatim.openstreetmap.org/reverse?lat=44.9537&lon=-93.0900&format=json
```

**Response:**
```json
{
  "display_name": "200 University Avenue West, St Paul, MN 55103, USA",
  "address": {
    "house_number": "200",
    "road": "University Avenue West",
    "city": "St Paul",
    "state": "Minnesota",
    "postcode": "55103"
  }
}
```

**Usage in Vue:**
```javascript
async function reverseGeocode(lat, lng) {
  const url = `https://nominatim.openstreetmap.org/reverse?lat=${lat}&lon=${lng}&format=json`;
  const response = await fetch(url, {
    headers: { 'User-Agent': 'StPaulCrimeSPA/1.0' }
  });
  const data = await response.json();
  return data.display_name;
}
```

---

## Data Shapes

### Crime Incident Object
```typescript
{
  case_number: string;        // "24-123456"
  date: string;               // "2024-03-15"
  time: string;               // "14:30:00"
  code: number;               // 220
  incident: string;           // "Robbery"
  police_grid: number;        // 87
  neighborhood_number: number; // 4
  block: string;              // "2XX UNIVERSITY AVE W"
}
```

### Neighborhood Object
```typescript
{
  id: number;    // 1-17
  name: string;  // "Conway/Battlecreek/Highwood"
}
```

### Crime Code Object
```typescript
{
  code: number;  // 110
  type: string;  // "Murder, Non Negligent Manslaughter"
}
```

---

## Example Fetch Calls

### Fetch all codes
```javascript
const codes = await fetch(`${crime_url}/codes`).then(r => r.json());
```

### Fetch all neighborhoods
```javascript
const neighborhoods = await fetch(`${crime_url}/neighborhoods`).then(r => r.json());
```

### Fetch filtered incidents
```javascript
const params = new URLSearchParams({
  start_date: '2024-01-01',
  end_date: '2024-12-31',
  limit: '1000'
});
const incidents = await fetch(`${crime_url}/incidents?${params}`).then(r => r.json());
```

### Submit new incident
```javascript
await fetch(`${crime_url}/new-incident`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    case_number: '24-999999',
    date: '2024-12-10',
    time: '09:00:00',
    code: 220,
    incident: 'Robbery',
    police_grid: 87,
    neighborhood_number: 4,
    block: '1XX MAIN ST'
  })
});
```

