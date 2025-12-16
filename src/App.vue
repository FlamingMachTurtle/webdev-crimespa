<script setup>
import { reactive, ref, onMounted, computed } from 'vue'


let crime_url = ref('');
let dialog_err = ref(false);
let dialog_err_msg = ref('');
let localhost_loading = ref(false);

// data from api - filled after user enters url
let codes = ref([]);
let neighborhoods = ref([]);
let incidents = ref([]);

// new incident form data
let newIncident = reactive({
    case_number: '',
    date: '',
    time: '',
    code: '',
    incident: '',
    police_grid: '',
    neighborhood_number: '',
    block: ''
});
let form_error = ref('');
let form_success = ref(false);
let form_visible = ref(false); // controls if form is shown or hidden
let showAdvancedFields = ref(false); // toggle for advanced form fields
let successPopup = ref(false); // show success popup
let lastAddedCase = ref(''); // track the last added case number

// filter state
let filters = reactive({
    start_date: '',
    end_date: '',
    selected_codes: [],      // array of selected crime type codes
    selected_neighborhood: '', // single neighborhood id or empty for all
    limit_to_visible: true,   // only show crimes in visible map area (core requirement)
    max_incidents: 1000       // max incidents to show
});

// track which neighborhoods are currently visible on map
let visibleNeighborhoods = ref([]);

// X2: Individual crime markers state
let crimeMarkers = reactive({});  // object to store markers by case_number

// X3: Table sorting state
let tableSort = reactive({
    field: 'date',     // current sort field
    direction: 'desc'  // 'asc' or 'desc'
});

// X4: Table search state
let tableSearch = ref('');

// Smart search state
let smartSearchOpen = ref(false);
let smartSearchIndex = ref(-1); // currently highlighted suggestion (-1 = none)

// Filter dropdown state (Option A)
let dateFilter = ref('all');  // 'all', 'today', 'week', 'month', 'custom'
let timeFilter = ref('all');  // 'all', 'morning', 'afternoon', 'evening', 'night', 'weekdays', 'weekends', 'business'
let typeFilter = ref('all');  // 'all', 'violent', 'property', 'other' (kept for backwards compat)

// Crime type panel state (Option 3)
let crimeTypePanelOpen = ref(false);

// Map search autocomplete state
let mapSearchOpen = ref(false);
let mapSearchIndex = ref(-1);

// C1 + C2 + C3: address search state
let addressSearch = reactive({
    query: '',
    searching: false,
    error: '',
    lat: null,
    lng: null,
    marker: null,  // store reference to search marker so we can remove it
    autoUpdating: false // track if we're auto-updating from map movement
});

// Manual lat/lng input state
let manualCoords = reactive({
    lat: '',
    lng: '',
    isEditing: false // track if user is currently editing the fields
});

// Debounce timer for reverse geocoding
let reverseGeocodeTimer = null;

// B3: neighborhood crime counts
let neighborhoodCrimeCounts = reactive({});

// Block-level geocoding cache - stores geocoded addresses to avoid repeat API calls
// Key: "600_MAIN_ST" -> Value: {lat, lng}
let geocodeCache = reactive({});

// Track which markers are currently being geocoded (loading state)
// Using ref so Vue properly tracks changes when we add/remove keys
let geocodingInProgress = ref({});

// Track geocoding progress for each case (0-100)
let geocodingProgress = ref({});

// Status message shown during geocoding
let geocodingStatus = ref('');

// Analytics panel state
let analyticsCollapsed = ref(false);
let districtLayers = ref({});  // map of neighborhood_id -> Leaflet layer

// Labels for charts
const hourLabels = ['12am', '3am', '6am', '9am', '12pm', '3pm', '6pm', '9pm'];
const dayLabels = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'];

// Cross-filtering state for charts
let selectedHourBucket = ref(null);  // 0-7 or null for all
let selectedDayIndex = ref(null);    // 0-6 (Sun-Sat) or null for all

let map = reactive(
    {
        leaflet: null,
        center: {
            lat: 44.955139,
            lng: -93.102222,
            address: ''
        },
        zoom: 12,
        bounds: {
            nw: {lat: 45.008206, lng: -93.217977},
            se: {lat: 44.883658, lng: -92.993787}
        },
        neighborhood_markers: [
            {location: [44.942068, -93.020521], marker: null},
            {location: [44.977413, -93.025156], marker: null},
            {location: [44.931244, -93.079578], marker: null},
            {location: [44.956192, -93.060189], marker: null},
            {location: [44.978883, -93.068163], marker: null},
            {location: [44.975766, -93.113887], marker: null},
            {location: [44.959639, -93.121271], marker: null},
            {location: [44.947700, -93.128505], marker: null},
            {location: [44.930276, -93.119911], marker: null},
            {location: [44.982752, -93.147910], marker: null},
            {location: [44.963631, -93.167548], marker: null},
            {location: [44.973971, -93.197965], marker: null},
            {location: [44.949043, -93.178261], marker: null},
            {location: [44.934848, -93.176736], marker: null},
            {location: [44.913106, -93.170779], marker: null},
            {location: [44.937705, -93.136997], marker: null},
            {location: [44.949203, -93.093739], marker: null}
        ]
    }
);

// Vue callback for once <template> HTML has been added to web page
onMounted(() => {
    // Create Leaflet map (set bounds and valied zoom levels)
    map.leaflet = L.map('leafletmap').setView([map.center.lat, map.center.lng], map.zoom);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        minZoom: 11,
        maxZoom: 18
    }).addTo(map.leaflet);
    map.leaflet.setMaxBounds([[44.883658, -93.217977], [45.008206, -92.993787]]);
    
    // Initialize manual coordinate inputs with map center
    manualCoords.lat = map.center.lat.toFixed(6);
    manualCoords.lng = map.center.lng.toFixed(6);

    // Update when map is panned/zoomed (core requirement)
    map.leaflet.on('moveend', () => {
        const center = map.leaflet.getCenter();
        map.center.lat = center.lat;
        map.center.lng = center.lng;
        // Store current center address for display (but don't overwrite if user is typing)
        map.center.address = `${center.lat.toFixed(6)}, ${center.lng.toFixed(6)}`;
        
        // Update manual coordinate inputs (only if user isn't editing them)
        if (!manualCoords.isEditing) {
            manualCoords.lat = center.lat.toFixed(6);
            manualCoords.lng = center.lng.toFixed(6);
        }
        
        // Update visible neighborhoods based on map bounds
        updateVisibleNeighborhoods();
        
        // Debounced reverse geocoding to update search bar (prevent API overload)
        updateSearchFromMapPosition();
    });

    // Get boundaries for St. Paul neighborhoods with click handlers
    fetch('data/StPaulDistrictCouncil.geojson')
    .then((response) => {
        return response.json();
    })
    .then((result) => {
        result.features.forEach((feature) => {
            // Get district number from properties (district council number matches neighborhood)
            const districtNum = parseInt(feature.properties.district);
            
            // Create layer with styling and click handler
            const layer = L.geoJSON(feature, {
                style: getPolygonStyle(districtNum),
                onEachFeature: (f, l) => {
                    // Click handler for polygon selection
                    l.on('click', (e) => {
                        L.DomEvent.stopPropagation(e);
                        selectNeighborhood(districtNum);
                    });
                    // Hover effects
                    l.on('mouseover', () => {
                        if (filters.selected_neighborhood != districtNum) {
                            l.setStyle({ fillOpacity: 0.4 });
                        }
                    });
                    l.on('mouseout', () => {
                        l.setStyle(getPolygonStyle(districtNum));
                    });
                }
            });
            
            // Store reference and add to map
            districtLayers.value[districtNum] = layer;
            layer.addTo(map.leaflet);
        });
    })
    .catch((error) => {
        console.log('Error loading district boundaries:', error);
    });
    
    // Click anywhere on map background to deselect
    map.leaflet.on('click', () => {
        if (filters.selected_neighborhood !== '') {
            clearNeighborhoodSelection();
        }
    });
});


// FUNCTIONS
// called after user enters api url and clicks ok
function initializeCrimes() {
    // remove trailing slash if user added one
    let api = crime_url.value.replace(/\/+$/, '');
    
    // grab crime codes from api
    fetch(api + '/codes')
        .then(res => res.json())
        .then(data => { 
            codes.value = data;
            console.log('codes loaded:', data.length);
        });

    // grab neighborhood list  
    fetch(api + '/neighborhoods')
        .then(res => res.json())
        .then(data => {
            neighborhoods.value = data;
            console.log('neighborhoods loaded:', data.length);
        });

    // grab last 1000 crimes - newest first
    fetch(api + '/incidents?limit=1000')
        .then(res => res.json())
        .then(data => {
            incidents.value = data;
            console.log('incidents loaded:', data.length);
            // B3: Calculate crime counts per neighborhood
            updateNeighborhoodCrimeCounts();
            // B3: Add neighborhood markers
            addNeighborhoodMarkers();
            // Initialize visible neighborhoods based on current map view
            updateVisibleNeighborhoods();
        });
}

// B3: Calculate crime counts for each neighborhood
function updateNeighborhoodCrimeCounts() {
    neighborhoodCrimeCounts.value = {};
    incidents.value.forEach(inc => {
        const nid = inc.neighborhood_number;
        if (!neighborhoodCrimeCounts.value[nid]) {
            neighborhoodCrimeCounts.value[nid] = 0;
        }
        neighborhoodCrimeCounts.value[nid]++;
    });
}

// B3: Add markers for each neighborhood showing crime count
function addNeighborhoodMarkers() {
    // Remove old markers
    map.neighborhood_markers.forEach(nm => {
        if (nm.marker) {
            map.leaflet.removeLayer(nm.marker);
            nm.marker = null;
        }
    });

    // Add new markers with crime counts
    map.neighborhood_markers.forEach((nm, idx) => {
        const nid = idx + 1; // neighborhood IDs are 1-indexed
        const count = neighborhoodCrimeCounts.value[nid] || 0;
        
        // Create a custom icon based on crime count
        const color = getColorByCount(count);
        const iconUrl = `data:image/svg+xml;base64,${btoa(`
            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="32" height="32">
                <circle cx="12" cy="12" r="10" fill="${color}" stroke="white" stroke-width="2"/>
                <text x="12" y="15" font-size="10" font-weight="bold" text-anchor="middle" fill="white">${count}</text>
            </svg>
        `)}`;
        
        const customIcon = L.icon({
            iconUrl: iconUrl,
            iconSize: [32, 32],
            iconAnchor: [16, 16],
            popupAnchor: [0, -16]
        });
        
        nm.marker = L.marker(nm.location, { icon: customIcon })
            .bindPopup(`<b>${neighborhoods.value.find(n => n.id === nid)?.name || 'Neighborhood ' + nid}</b><br/>Crimes: ${count}`)
            .addTo(map.leaflet);
    });
}

// B3: Get color based on crime count (heatmap style)
function getColorByCount(count) {
    if (count === 0) return '#90EE90'; // light green
    if (count < 50) return '#FFD700'; // gold
    if (count < 100) return '#FFA500'; // orange
    if (count < 150) return '#FF6347'; // tomato
    return '#DC143C'; // crimson
}

// Core requirement: determine which neighborhoods are visible on current map view
function updateVisibleNeighborhoods() {
    if (!map.leaflet) return;
    
    const bounds = map.leaflet.getBounds();
    const visible = [];
    
    // Check each neighborhood marker location against map bounds
    map.neighborhood_markers.forEach((nm, idx) => {
        const nid = idx + 1; // neighborhood IDs are 1-indexed
        const [lat, lng] = nm.location;
        if (bounds.contains([lat, lng])) {
            visible.push(nid);
        }
    });
    
    visibleNeighborhoods.value = visible;
}

// C1 + C2 + C3: Search for address and center map
async function searchAddress() {
    if (!addressSearch.query.trim()) {
        addressSearch.error = 'Please enter an address';
        return;
    }

    addressSearch.searching = true;
    addressSearch.error = '';
    addressSearch.lat = null;
    addressSearch.lng = null;

    try {
        // C2: Use Nominatim API to geocode address
        const response = await fetch(
            `https://nominatim.openstreetmap.org/search?q=${encodeURIComponent(addressSearch.query)}&format=json&limit=1`,
            {
                headers: {
                    'User-Agent': 'StPaulCrimeSPA/1.0'
                }
            }
        );
        
        if (!response.ok) {
            throw new Error('Geocoding service error');
        }
        
        const results = await response.json();
        
        if (!results || results.length === 0) {
            addressSearch.error = 'Address not found';
            return;
        }

        const result = results[0];
        addressSearch.lat = parseFloat(result.lat);
        addressSearch.lng = parseFloat(result.lon);

        // C3: Center map on search result
        map.leaflet.setView([addressSearch.lat, addressSearch.lng], 15);
        
        // Remove old marker if it exists
        if (addressSearch.marker) {
            map.leaflet.removeLayer(addressSearch.marker);
        }
        
        // Add a marker at the search location
        addressSearch.marker = L.marker([addressSearch.lat, addressSearch.lng], {
            title: addressSearch.query
        }).bindPopup(`<b>Search Result</b><br/>${addressSearch.query}`)
         .openPopup()
         .addTo(map.leaflet);

        // Extract street name from query for table filtering
        // e.g., "123 University Ave, St Paul" -> "UNIVERSITY"
        const streetMatch = extractStreetName(addressSearch.query);
        
        if (streetMatch) {
            // Filter the crime table to show incidents on this street
            tableSearch.value = streetMatch;
            
            // Scroll to the table section after a brief delay (let map animate first)
            setTimeout(() => {
                const tableSection = document.querySelector('.table-section');
                if (tableSection) {
                    tableSection.scrollIntoView({ behavior: 'smooth', block: 'start' });
                }
            }, 800);
        }

    } catch (error) {
        console.error('Search error:', error);
        addressSearch.error = 'Failed to geocode address: ' + error.message;
    } finally {
        addressSearch.searching = false;
    }
}

// Extract street name from a full address for table filtering
// e.g., "123 University Ave, St Paul, MN" -> "UNIVERSITY"
// e.g., "Snelling Avenue" -> "SNELLING"
function extractStreetName(address) {
    if (!address) return '';
    
    // Common street suffixes to help identify street names
    const suffixes = ['AVE', 'AVENUE', 'ST', 'STREET', 'BLVD', 'BOULEVARD', 'DR', 'DRIVE', 
                      'RD', 'ROAD', 'LN', 'LANE', 'WAY', 'CT', 'COURT', 'PL', 'PLACE',
                      'PKWY', 'PARKWAY', 'CIR', 'CIRCLE'];
    
    // Clean up the address - uppercase and remove extra spaces
    let cleaned = address.toUpperCase().trim();
    
    // Remove city/state/zip if present (everything after first comma usually)
    const commaIndex = cleaned.indexOf(',');
    if (commaIndex > 0) {
        cleaned = cleaned.substring(0, commaIndex);
    }
    
    // Split into words
    const words = cleaned.split(/\s+/);
    
    // Try to find the street name (word before a suffix, or just the main word)
    for (let i = 0; i < words.length; i++) {
        if (suffixes.includes(words[i]) && i > 0) {
            // Return the word before the suffix (the actual street name)
            // Skip numbers at the start
            let streetName = words[i - 1];
            if (!/^\d+$/.test(streetName)) {
                return streetName;
            }
        }
    }
    
    // Fallback: return the longest non-numeric word (likely the street name)
    const nonNumericWords = words.filter(w => !/^\d+X*$/.test(w) && w.length > 2);
    if (nonNumericWords.length > 0) {
        return nonNumericWords.reduce((a, b) => a.length >= b.length ? a : b);
    }
    
    return cleaned;
}

// Clear search marker and reset search
function clearSearch() {
    if (addressSearch.marker) {
        map.leaflet.removeLayer(addressSearch.marker);
        addressSearch.marker = null;
    }
    addressSearch.query = '';
    addressSearch.error = '';
    addressSearch.lat = null;
    addressSearch.lng = null;
    
    // Trigger an immediate reverse geocode to fill in current location
    setTimeout(() => {
        updateSearchFromMapPosition();
    }, 100);
}

// Debounced reverse geocoding - updates search bar as map moves (throttled to prevent API overload)
function updateSearchFromMapPosition() {
    // Clear any pending reverse geocode requests
    if (reverseGeocodeTimer) {
        clearTimeout(reverseGeocodeTimer);
    }
    
    // Wait 2 seconds after user stops moving the map before making API call
    reverseGeocodeTimer = setTimeout(async () => {
        if (!map.leaflet) return;
        
        // Don't override if user is actively searching
        if (addressSearch.searching) {
            console.log('Skipping reverse geocode - search in progress');
            return;
        }
        
        const center = map.leaflet.getCenter();
        
        try {
            addressSearch.autoUpdating = true;
            
            // Reverse geocode the center position with proper headers
            const response = await fetch(
                `https://nominatim.openstreetmap.org/reverse?lat=${center.lat}&lon=${center.lng}&format=json&zoom=16`,
                {
                    headers: {
                        'User-Agent': 'StPaulCrimeSPA/1.0'
                    }
                }
            );
            
            if (response.ok) {
                const result = await response.json();
                
                // Build a concise address string
                let displayAddress = '';
                
                if (result.address) {
                    // Try to build a short, useful address
                    const addr = result.address;
                    const parts = [];
                    
                    if (addr.road) {
                        parts.push(addr.road);
                    } else if (addr.neighbourhood) {
                        parts.push(addr.neighbourhood);
                    }
                    
                    if (addr.suburb && addr.suburb !== addr.neighbourhood) {
                        parts.push(addr.suburb);
                    }
                    
                    displayAddress = parts.join(', ');
                    
                    // Fallback to full display name if we couldn't build a good address
                    if (!displayAddress) {
                        displayAddress = result.display_name.split(',').slice(0, 2).join(',');
                    }
                } else {
                    displayAddress = result.display_name || `${center.lat.toFixed(6)}, ${center.lng.toFixed(6)}`;
                }
                
                // Always update with the new location
                addressSearch.query = displayAddress;
                console.log('Reverse geocoded:', displayAddress);
            }
        } catch (error) {
            console.log('Reverse geocoding failed:', error);
            // Silently fail - this is a convenience feature
        } finally {
            addressSearch.autoUpdating = false;
        }
    }, 2000); // 2 second debounce to reduce API calls
}

// Handle manual latitude/longitude input
function updateMapFromCoords() {
    const lat = parseFloat(manualCoords.lat);
    const lng = parseFloat(manualCoords.lng);
    
    // Validate coordinates
    if (isNaN(lat) || isNaN(lng)) {
        addressSearch.error = 'Invalid coordinates';
        return;
    }
    
    if (lat < -90 || lat > 90) {
        addressSearch.error = 'Latitude must be between -90 and 90';
        return;
    }
    
    if (lng < -180 || lng > 180) {
        addressSearch.error = 'Longitude must be between -180 and 180';
        return;
    }
    
    addressSearch.error = '';
    
    // Update map center
    map.leaflet.setView([lat, lng], map.leaflet.getZoom());
    
    // Add a marker at the specified location
    if (addressSearch.marker) {
        map.leaflet.removeLayer(addressSearch.marker);
    }
    
    addressSearch.marker = L.marker([lat, lng], {
        title: `${lat}, ${lng}`
    }).bindPopup(`<b>Manual Location</b><br/>${lat.toFixed(6)}, ${lng.toFixed(6)}`)
     .openPopup()
     .addTo(map.leaflet);
}

// Handle when user starts/stops editing coordinate fields
function handleCoordFocus() {
    manualCoords.isEditing = true;
}

function handleCoordBlur() {
    manualCoords.isEditing = false;
}

// submit new incident to api
function submitIncident() {
    // Check required fields only
    if (!newIncident.code || !newIncident.neighborhood_number || !newIncident.block) {
        form_error.value = 'Please fill out Crime Type, Neighborhood, and Address';
        return;
    }
    
    // Auto-generate case number if not provided
    let caseNumber = newIncident.case_number;
    if (!caseNumber) {
        const year = new Date().getFullYear().toString().slice(-2);
        const randomNum = Math.floor(100000 + Math.random() * 900000);
        caseNumber = `${year}-${randomNum}`;
    }
    
    // Auto-generate police grid if not set
    let policeGrid = newIncident.police_grid;
    if (!policeGrid) {
        policeGrid = neighborhoodGridMap[newIncident.neighborhood_number] || 87;
    }
    
    // Auto-fill incident description if empty
    let incidentDesc = newIncident.incident;
    if (!incidentDesc) {
        const selectedCode = codes.value.find(c => c.code == newIncident.code);
        incidentDesc = selectedCode ? selectedCode.type : 'Incident';
    }
    
    form_error.value = '';
    form_success.value = false;
    let api = crime_url.value.replace(/\/+$/, '');
    
    // Ensure time has seconds (HH:MM:SS format)
    let timeStr = newIncident.time;
    if (timeStr && timeStr.length === 5) {
        timeStr = timeStr + ':00'; // Add seconds if missing
    }
    
    const payload = {
        case_number: caseNumber,
        date: newIncident.date,
        time: timeStr,
        code: Number(newIncident.code),
        incident: incidentDesc,
        police_grid: Number(policeGrid),
        neighborhood_number: Number(newIncident.neighborhood_number),
        block: newIncident.block.toUpperCase()
    };
    
    console.log('Submitting incident:', payload);
    
    // send to api
    fetch(api + '/new-incident', {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
    })
    .then(res => {
        if (!res.ok) {
            return res.text().then(text => {
                throw new Error(`Server error: ${res.status} - ${text}`);
            });
        }
        // Server may return "OK" as text or JSON
        return res.text();
    })
    .then(data => {
        console.log('incident added:', data);
        form_success.value = true;
        lastAddedCase.value = caseNumber;
        
        // Show success popup
        successPopup.value = true;
        
        // clear form
        newIncident.case_number = '';
        newIncident.date = '';
        newIncident.time = '';
        newIncident.code = '';
        newIncident.incident = '';
        newIncident.police_grid = '';
        newIncident.neighborhood_number = '';
        newIncident.block = '';
        
        // Refresh the incidents list
        initializeCrimes();
    })
    .catch(err => {
        form_error.value = err.message || 'Failed to add incident';
        console.error('Submit error:', err);
    });
}

// show or hide the incident form
function toggleForm() {
    form_visible.value = !form_visible.value;
    
    // Auto-populate date and time when opening form
    if (form_visible.value) {
        const now = new Date();
        if (!newIncident.date) {
            newIncident.date = now.toISOString().split('T')[0]; // YYYY-MM-DD
        }
        if (!newIncident.time) {
            newIncident.time = now.toTimeString().slice(0, 5); // HH:MM
        }
        form_error.value = '';
        form_success.value = false;
    }
}

// Auto-fill incident description based on crime type
function autofillIncidentType() {
    if (newIncident.code && !newIncident.incident) {
        const selectedCode = codes.value.find(c => c.code == newIncident.code);
        if (selectedCode) {
            newIncident.incident = selectedCode.type;
        }
    }
}

// Auto-assign a reasonable police grid based on neighborhood
// These are approximate mappings - not exact but good defaults
const neighborhoodGridMap = {
    1: 75, 2: 61, 3: 82, 4: 63, 5: 78,
    6: 85, 7: 93, 8: 48, 9: 52, 10: 51,
    11: 65, 12: 86, 13: 99, 14: 107, 15: 112,
    16: 116, 17: 121
};

function autofillPoliceGrid() {
    if (newIncident.neighborhood_number && !newIncident.police_grid) {
        newIncident.police_grid = neighborhoodGridMap[newIncident.neighborhood_number] || 87;
    }
}

// Jump to the newly added incident in the table
function jumpToIncident() {
    successPopup.value = false;
    form_visible.value = false;
    
    // Retry mechanism to find the row (data might still be loading)
    let attempts = 0;
    const maxAttempts = 10;
    
    function findAndHighlight() {
        attempts++;
        const caseNum = lastAddedCase.value;
        const row = document.querySelector(`tr[data-case="${caseNum}"]`);
        
        if (row) {
            console.log('Found row for case:', caseNum);
            row.scrollIntoView({ behavior: 'smooth', block: 'center' });
            row.classList.add('highlight-row');
            // Keep highlight for 5 seconds
            setTimeout(() => row.classList.remove('highlight-row'), 5000);
        } else if (attempts < maxAttempts) {
            // Try again in 300ms
            console.log(`Attempt ${attempts}: Row not found yet, retrying...`);
            setTimeout(findAndHighlight, 300);
        } else {
            // Final fallback: just scroll to table
            console.log('Could not find row, scrolling to table');
            const table = document.querySelector('.crime-table');
            if (table) {
                table.scrollIntoView({ behavior: 'smooth', block: 'start' });
            }
        }
    }
    
    // Start searching after a short delay
    setTimeout(findAndHighlight, 500);
}

// Close success popup
function closeSuccessPopup() {
    successPopup.value = false;
}

// Function called when user presses 'OK' on dialog box
function closeDialog() {
    let dialog = document.getElementById('rest-dialog');
    let url_input = document.getElementById('dialog-url');
    if (crime_url.value !== '' && url_input.checkValidity()) {
        dialog_err.value = false;
        dialog_err_msg.value = '';
        dialog.close();
        initializeCrimes();
    }
    else {
        dialog_err.value = true;
        dialog_err_msg.value = 'Error: must enter valid URL';
    }
}

// Quick-fill localhost:8000 with connection test
async function useLocalhost() {
    const testUrl = 'http://localhost:8000';
    localhost_loading.value = true;
    dialog_err.value = false;
    dialog_err_msg.value = '';
    
    try {
        // Test connection with a quick fetch to /codes endpoint
        const controller = new AbortController();
        const timeoutId = setTimeout(() => controller.abort(), 3000);
        
        const response = await fetch(`${testUrl}/codes`, { signal: controller.signal });
        clearTimeout(timeoutId);
        
        if (response.ok) {
            // Connection successful - close dialog and load data
            crime_url.value = testUrl;
            let dialog = document.getElementById('rest-dialog');
            dialog.close();
            initializeCrimes();
        } else {
            throw new Error('Server returned error');
        }
    } catch (error) {
        // Show error message
        dialog_err.value = true;
        if (error.name === 'AbortError') {
            dialog_err_msg.value = '⚠️ Connection timed out. Is the server running?';
        } else {
            dialog_err_msg.value = '⚠️ Cannot connect to localhost:8000. Start the server with: node rest_server.mjs';
        }
        console.log('Localhost connection failed:', error);
    } finally {
        localhost_loading.value = false;
    }
}

// Helper: check if a time falls within a period
function isTimeInPeriod(timeStr, period) {
    if (!timeStr) return true; // No time data, include it
    
    // Parse HH:MM:SS or HH:MM format
    const timeParts = timeStr.split(':');
    const hour = parseInt(timeParts[0]);
    
    switch(period) {
        case 'morning':
            return hour >= 6 && hour < 12;
        case 'afternoon':
            return hour >= 12 && hour < 18;
        case 'evening':
            return hour >= 18 && hour < 22;
        case 'night':
            return hour >= 22 || hour < 6;
        case 'business':
            return hour >= 9 && hour < 17;
        default:
            return true;
    }
}

// Helper: check if a date is a weekend or weekday
function isDayType(dateStr, type) {
    if (!dateStr) return true;
    
    const date = new Date(dateStr);
    const dayOfWeek = date.getDay(); // 0 = Sunday, 6 = Saturday
    
    if (type === 'weekends') {
        return dayOfWeek === 0 || dayOfWeek === 6;
    } else if (type === 'weekdays') {
        return dayOfWeek >= 1 && dayOfWeek <= 5;
    }
    
    return true;
}

// computed property - filters incidents based on current filter selections
const filteredIncidents = computed(() => {
    // Helper: get date string for comparisons
    const today = new Date();
    const todayStr = today.toISOString().split('T')[0];
    const weekAgo = new Date(today.getTime() - 7 * 24 * 60 * 60 * 1000).toISOString().split('T')[0];
    const monthAgo = new Date(today.getTime() - 30 * 24 * 60 * 60 * 1000).toISOString().split('T')[0];
    
    // First apply filters
    let filtered = incidents.value.filter(inc => {
        // Core requirement: filter by visible map area
        if (filters.limit_to_visible && visibleNeighborhoods.value.length > 0) {
            // Use Number() to ensure consistent comparison (API may return string)
            if (!visibleNeighborhoods.value.includes(Number(inc.neighborhood_number))) return false;
        }
        
        // Date filter dropdown (new Option A style)
        if (dateFilter.value === 'today') {
            if (inc.date !== todayStr) return false;
        } else if (dateFilter.value === 'week') {
            if (inc.date < weekAgo) return false;
        } else if (dateFilter.value === 'month') {
            if (inc.date < monthAgo) return false;
        } else if (dateFilter.value === 'custom') {
            // Use custom date range inputs
            if (filters.start_date && inc.date < filters.start_date) return false;
            if (filters.end_date && inc.date > filters.end_date) return false;
        }
        
        // Time filter (morning, afternoon, evening, night, weekdays, weekends, business hours)
        if (timeFilter.value !== 'all') {
            if (timeFilter.value === 'weekdays' || timeFilter.value === 'weekends') {
                if (!isDayType(inc.date, timeFilter.value)) return false;
            } else {
                if (!isTimeInPeriod(inc.time, timeFilter.value)) return false;
            }
        }
        
        // Crime type filter (Option 3: specific codes or categories)
        if (filters.selected_codes.length > 0) {
            if (!filters.selected_codes.includes(inc.code)) return false;
        }
        
        // Neighborhood filter
        if (filters.selected_neighborhood !== '') {
            if (inc.neighborhood_number != filters.selected_neighborhood) return false;
        }
        
        // X4: table search filter
        if (tableSearch.value.trim()) {
            const searchLower = tableSearch.value.toLowerCase();
            const neighborhood = neighborhoods.value.find(n => n.id === inc.neighborhood_number)?.name || '';
            const searchFields = [
                inc.case_number.toString(),
                inc.date,
                inc.time,
                inc.incident.toLowerCase(),
                inc.block.toLowerCase(),
                neighborhood.toLowerCase()
            ].join(' ');
            
            if (!searchFields.includes(searchLower)) return false;
        }
        
        return true;
    });
    
    // Apply max incidents limit
    if (filters.max_incidents && filters.max_incidents < filtered.length) {
        filtered = filtered.slice(0, filters.max_incidents);
    }
    
    // X3: Apply sorting
    filtered.sort((a, b) => {
        let aVal, bVal;
        
        switch(tableSort.field) {
            case 'case_number':
                aVal = a.case_number.toString();
                bVal = b.case_number.toString();
                break;
            case 'date':
                aVal = a.date;
                bVal = b.date;
                break;
            case 'time':
                aVal = a.time;
                bVal = b.time;
                break;
            case 'incident':
                aVal = a.incident.toLowerCase();
                bVal = b.incident.toLowerCase();
                break;
            case 'block':
                aVal = a.block.toLowerCase();
                bVal = b.block.toLowerCase();
                break;
            case 'neighborhood':
                aVal = neighborhoods.value.find(n => n.id === a.neighborhood_number)?.name || '';
                bVal = neighborhoods.value.find(n => n.id === b.neighborhood_number)?.name || '';
                aVal = aVal.toLowerCase();
                bVal = bVal.toLowerCase();
                break;
            default:
                aVal = a.date;
                bVal = b.date;
        }
        
        if (aVal < bVal) return tableSort.direction === 'asc' ? -1 : 1;
        if (aVal > bVal) return tableSort.direction === 'asc' ? 1 : -1;
        return 0;
    });
    
    return filtered;
});

// Smart search: generate categorized suggestions based on search input
const smartSearchSuggestions = computed(() => {
    const query = tableSearch.value.trim().toLowerCase();
    if (!query || query.length < 2) return [];
    
    const suggestions = [];
    const MAX_PER_CATEGORY = 3;
    
    // Category 1: Case numbers (exact or partial match)
    const caseMatches = incidents.value
        .filter(inc => inc.case_number.toString().toLowerCase().includes(query))
        .slice(0, MAX_PER_CATEGORY)
        .map(inc => ({
            type: 'case',
            label: inc.case_number,
            display: `Case #${inc.case_number}`,
            incident: inc
        }));
    suggestions.push(...caseMatches);
    
    // Category 2: Addresses/Blocks (unique blocks that match)
    const seenBlocks = new Set();
    const blockMatches = incidents.value
        .filter(inc => {
            const block = inc.block.toLowerCase();
            if (seenBlocks.has(block)) return false;
            if (!block.includes(query)) return false;
            seenBlocks.add(block);
            return true;
        })
        .slice(0, MAX_PER_CATEGORY)
        .map(inc => ({
            type: 'address',
            label: inc.block,
            display: inc.block,
            incident: inc
        }));
    suggestions.push(...blockMatches);
    
    // Category 3: Crime types (from codes that match, linked to first incident with that code)
    const crimeTypeMatches = codes.value
        .filter(c => c.type.toLowerCase().includes(query))
        .slice(0, MAX_PER_CATEGORY)
        .map(c => {
            const firstIncident = incidents.value.find(inc => inc.code === c.code);
            return {
                type: 'crime',
                label: c.type,
                display: c.type,
                code: c.code,
                incident: firstIncident || null
            };
        })
        .filter(s => s.incident !== null); // only include if there's a matching incident
    suggestions.push(...crimeTypeMatches);
    
    return suggestions.slice(0, 8); // max 8 total suggestions
});

// Active filters: generate list of active filter chips for display
const activeFilters = computed(() => {
    const chips = [];
    
    // Date range
    if (filters.start_date && filters.end_date) {
        chips.push({
            id: 'date-range',
            label: `${filters.start_date} to ${filters.end_date}`,
            type: 'date',
            clear: () => { filters.start_date = ''; filters.end_date = ''; }
        });
    } else if (filters.start_date) {
        chips.push({
            id: 'start-date',
            label: `From ${filters.start_date}`,
            type: 'date',
            clear: () => { filters.start_date = ''; }
        });
    } else if (filters.end_date) {
        chips.push({
            id: 'end-date',
            label: `Until ${filters.end_date}`,
            type: 'date',
            clear: () => { filters.end_date = ''; }
        });
    }
    
    // Selected crime types
    filters.selected_codes.forEach(code => {
        const crimeType = codes.value.find(c => c.code === code);
        if (crimeType) {
            chips.push({
                id: `crime-${code}`,
                label: crimeType.type,
                type: 'crime',
                clear: () => { filters.selected_codes = filters.selected_codes.filter(c => c !== code); }
            });
        }
    });
    
    // Selected neighborhood
    if (filters.selected_neighborhood) {
        const neighborhood = neighborhoods.value.find(n => n.id == filters.selected_neighborhood);
        if (neighborhood) {
            chips.push({
                id: 'neighborhood',
                label: neighborhood.name,
                type: 'neighborhood',
                clear: () => { filters.selected_neighborhood = ''; }
            });
        }
    }
    
    return chips;
});

// Check if any filters are active (for showing Clear button)
const hasActiveFilters = computed(() => {
    return dateFilter.value !== 'all' || 
           timeFilter.value !== 'all' ||
           filters.selected_codes.length > 0 ||
           filters.selected_neighborhood !== '' ||
           tableSearch.value.trim() !== '';
});

// Computed: crime types by category
const violentCrimes = computed(() => {
    return codes.value.filter(c => getCrimeCategory(c.code) === 'violent');
});

const propertyCrimes = computed(() => {
    return codes.value.filter(c => getCrimeCategory(c.code) === 'property');
});

const otherCrimes = computed(() => {
    return codes.value.filter(c => getCrimeCategory(c.code) === 'other');
});

// Map search: generate suggestions from neighborhoods and common addresses
const mapSearchSuggestions = computed(() => {
    const query = addressSearch.query.trim().toLowerCase();
    if (!query || query.length < 2) return [];
    
    const suggestions = [];
    const MAX_PER_CATEGORY = 4;
    
    // Category 1: Neighborhoods that match
    const neighborhoodMatches = neighborhoods.value
        .filter(n => n.name.toLowerCase().includes(query))
        .slice(0, MAX_PER_CATEGORY)
        .map(n => ({
            type: 'neighborhood',
            label: n.name,
            display: n.name,
            id: n.id
        }));
    suggestions.push(...neighborhoodMatches);
    
    // Category 2: Unique street names from incidents
    const seenStreets = new Set();
    const streetMatches = incidents.value
        .filter(inc => {
            // Extract street name from block (e.g., "7XX SNELLING AVE" -> "SNELLING AVE")
            const streetPart = inc.block.replace(/^\d+X*\s+/, '').toUpperCase();
            if (seenStreets.has(streetPart)) return false;
            if (!streetPart.toLowerCase().includes(query)) return false;
            seenStreets.add(streetPart);
            return true;
        })
        .slice(0, MAX_PER_CATEGORY)
        .map(inc => {
            const streetPart = inc.block.replace(/^\d+X*\s+/, '').toUpperCase();
            return {
                type: 'street',
                label: streetPart,
                display: streetPart + ', St Paul',
                block: inc.block
            };
        });
    suggestions.push(...streetMatches);
    
    return suggestions.slice(0, 6);
});

// categorize crime by type for color coding
function getCrimeCategory(code) {
    // violent crimes: homicide, rape, robbery, assault (typically codes < 500)
    if (code < 500) return 'violent';
    // property crimes: burglary, theft, auto theft, arson (typically 500-999)
    if (code >= 500 && code < 1000) return 'property';
    // other: everything else
    return 'other';
}
// delete incident from database
function deleteIncident(case_number) {
    if (!confirm(`Delete case ${case_number}?`)) return;
    
    let api = crime_url.value.replace(/\/+$/, '');
    
    fetch(api + '/remove-incident', {
        method: 'DELETE',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ case_number: String(case_number) })
    })
    .then(res => {
        if (!res.ok) {
            throw new Error('Server returned ' + res.status);
        }
        // remove from local array immediately - server returned OK
        incidents.value = incidents.value.filter(inc => inc.case_number !== case_number);
        // X2: remove marker if it exists
        if (crimeMarkers[case_number]) {
            map.leaflet.removeLayer(crimeMarkers[case_number]);
            delete crimeMarkers[case_number];
        }
        // update crime counts
        updateNeighborhoodCrimeCounts();
        addNeighborhoodMarkers();
        console.log('deleted case:', case_number);
    })
    .catch(err => {
        console.log('delete error:', err);
        alert('Failed to delete incident: ' + err.message);
    });
}

// X2: Handle address obscuring - replace X in address number with 0
function cleanAddress(address) {
    // Replace X in the first numeric part of the address
    // e.g., "90X MAIN ST" becomes "900 MAIN ST"
    return address.replace(/^(\d+)X\s/, '$10 ');
}

// Block-level geocoding: converts "6XX MAIN ST" to coordinates on the 600 block
async function geocodeBlock(address, neighborhoodNumber) {
    // Parse: "6XX MAIN ST" -> "600 MAIN ST" (replace X's with 0's in block number)
    const cleanedAddr = address.replace(/^(\d+)X+\s/i, (match, prefix) => prefix + '00 ');
    const cacheKey = cleanedAddr.replace(/\s+/g, '_').toUpperCase();
    
    // Return cached result if available (instant)
    if (geocodeCache[cacheKey]) {
        console.log('Geocode cache hit:', cacheKey);
        return { ...geocodeCache[cacheKey], cached: true };
    }
    
    try {
        // Geocode with "St Paul, MN" suffix for accuracy
        const query = `${cleanedAddr}, St Paul, MN`;
        console.log('Geocoding:', query);
        
        // Create abort controller for timeout (3 seconds max)
        const controller = new AbortController();
        const timeoutId = setTimeout(() => controller.abort(), 3000);
        
        const response = await fetch(
            `https://nominatim.openstreetmap.org/search?q=${encodeURIComponent(query)}&format=json&limit=1`,
            { signal: controller.signal }
        );
        
        clearTimeout(timeoutId);
        
        if (!response.ok) {
            throw new Error('Geocoding service error');
        }
        
        const results = await response.json();
        
        if (results && results.length > 0) {
            const result = results[0];
            const location = {
                lat: parseFloat(result.lat),
                lng: parseFloat(result.lon),
                accuracy: 'block'
            };
            
            // Cache the result
            geocodeCache[cacheKey] = location;
            console.log('Geocoded successfully:', cleanedAddr, location);
            return { ...location, cached: false };
        }
        
        // Geocoding returned no results - fall back to neighborhood
        console.log('Geocoding returned no results for:', query);
        return getFallbackLocation(neighborhoodNumber);
        
    } catch (error) {
        // Handle timeout or other errors gracefully
        if (error.name === 'AbortError') {
            console.log('Geocoding timed out, using neighborhood fallback');
        } else {
            console.error('Geocoding failed:', error);
        }
        return getFallbackLocation(neighborhoodNumber);
    }
}

// Fallback: return neighborhood center when geocoding fails
function getFallbackLocation(neighborhoodNumber) {
    const nIdx = neighborhoodNumber - 1;
    if (nIdx >= 0 && nIdx < map.neighborhood_markers.length) {
        const [lat, lng] = map.neighborhood_markers[nIdx].location;
        return { lat, lng, accuracy: 'neighborhood' };
    }
    // Ultimate fallback: city center
    return { lat: map.center.lat, lng: map.center.lng, accuracy: 'city' };
}

// X2: Add or toggle marker for a single crime incident (with block-level geocoding)
async function toggleCrimeMarker(incident) {
    const caseNum = incident.case_number;
    
    // If marker exists, remove it
    if (crimeMarkers[caseNum]) {
        map.leaflet.removeLayer(crimeMarkers[caseNum]);
        delete crimeMarkers[caseNum];
        return;
    }
    
    // Prevent double-clicks while geocoding
    if (geocodingInProgress.value[caseNum]) {
        return;
    }
    
    // Set loading state - use spread to trigger Vue reactivity
    geocodingInProgress.value = { ...geocodingInProgress.value, [caseNum]: true };
    
    // Initialize progress at 0
    geocodingProgress.value = { ...geocodingProgress.value, [caseNum]: 0 };
    
    // Animate progress from 0 to 85% over ~2.5 seconds (simulated progress during API call)
    const progressInterval = setInterval(() => {
        const current = geocodingProgress.value[caseNum] || 0;
        if (current < 85) {
            // Ease-out: slow down as we approach 85%
            const increment = Math.max(1, (85 - current) / 10);
            geocodingProgress.value = { ...geocodingProgress.value, [caseNum]: Math.min(85, current + increment) };
        }
    }, 100);
    
    // Show status message
    const cleanAddr = cleanAddress(incident.block);
    geocodingStatus.value = `📍 Locating "${cleanAddr}" on map...`;
    
    try {
        // Geocode the block address for accurate placement
        const location = await geocodeBlock(incident.block, incident.neighborhood_number);
        
        // Stop the simulated progress and jump to 100%
        clearInterval(progressInterval);
        geocodingProgress.value = { ...geocodingProgress.value, [caseNum]: 100 };
        
        // Add small random offset (~50m for block-level, ~200m for neighborhood fallback)
        // This prevents markers from stacking exactly on top of each other
        const offsetScale = location.accuracy === 'block' ? 0.0008 : 0.003;
        const lat = location.lat + (Math.random() - 0.5) * offsetScale;
        const lng = location.lng + (Math.random() - 0.5) * offsetScale;
        
        // Update status based on result
        if (location.accuracy === 'block') {
            geocodingStatus.value = `✅ Found "${cleanAddr}" - placing marker...`;
        } else {
            geocodingStatus.value = `⚠️ Couldn't find exact location, using neighborhood center...`;
        }
        
        // Create custom purple marker
        const crimeIcon = L.icon({
            iconUrl: 'data:image/svg+xml;base64,' + btoa(`
                <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 32" width="24" height="32">
                    <path d="M12 0C5.4 0 0 5.4 0 12c0 10 12 20 12 20s12-10 12-20c0-6.6-5.4-12-12-12z" fill="#9333ea" stroke="white" stroke-width="1"/>
                    <circle cx="12" cy="13" r="5" fill="white"/>
                </svg>
            `),
            iconSize: [24, 32],
            iconAnchor: [12, 32],
            popupAnchor: [0, -32]
        });
        
        // Build popup content with accuracy indicator
        const accuracyText = location.accuracy === 'block' 
            ? '📍 Block-level accuracy' 
            : '⚠️ Approximate (neighborhood center)';
        const accuracyColor = location.accuracy === 'block' ? '#2e7d32' : '#f57c00';
        
        const popupContent = `
            <div style="width: 240px;">
                <b>Crime Details</b><br/>
                <em style="color:${accuracyColor}; font-size: 0.85em;">${accuracyText}</em><br/>
                <strong>Case:</strong> ${incident.case_number}<br/>
                <strong>Date:</strong> ${incident.date}<br/>
                <strong>Time:</strong> ${incident.time}<br/>
                <strong>Incident:</strong> ${incident.incident}<br/>
                <strong>Address:</strong> ${cleanAddr}
            </div>
        `;
        
        const marker = L.marker([lat, lng], { icon: crimeIcon })
            .bindPopup(popupContent)
            .addTo(map.leaflet);
        
        crimeMarkers[caseNum] = marker;
        
        // Pan map to marker and open popup immediately
        map.leaflet.setView([lat, lng], 15, { animate: true });
        setTimeout(() => marker.openPopup(), 300);
        
        // Scroll page to show map
        window.scrollTo({ top: 0, behavior: 'smooth' });
        
    } catch (error) {
        console.error('Failed to place marker:', error);
        geocodingStatus.value = '❌ Failed to place marker';
        clearInterval(progressInterval);
    } finally {
        // Clear loading state - create new object without the key to trigger reactivity
        const { [caseNum]: _, ...rest } = geocodingInProgress.value;
        geocodingInProgress.value = rest;
        
        // Clear progress state after a brief delay (let user see 100%)
        setTimeout(() => {
            const { [caseNum]: __, ...progressRest } = geocodingProgress.value;
            geocodingProgress.value = progressRest;
        }, 300);
        
        // Clear status message after a short delay
        setTimeout(() => {
            geocodingStatus.value = '';
        }, 2000);
    }
}

// X3: Handle table column sorting
function sortBy(field) {
    if (tableSort.field === field) {
        // Toggle direction if same field clicked
        tableSort.direction = tableSort.direction === 'asc' ? 'desc' : 'asc';
    } else {
        // Change field and reset to ascending
        tableSort.field = field;
        tableSort.direction = 'asc';
    }
}

// X3: Get sort indicator for column headers
function getSortIndicator(field) {
    if (tableSort.field !== field) return '';
    return tableSort.direction === 'asc' ? ' ▲' : ' ▼';
}

// Smart search: get category label for display
function getCategoryLabel(type) {
    switch(type) {
        case 'case': return 'Case #';
        case 'address': return 'Address';
        case 'crime': return 'Crime Type';
        default: return '';
    }
}

// Smart search: highlight matching text in suggestion
function highlightMatch(text, query) {
    if (!query.trim()) return text;
    const regex = new RegExp(`(${query.trim().replace(/[.*+?^${}()|[\]\\]/g, '\\$&')})`, 'gi');
    return text.replace(regex, '<mark>$1</mark>');
}

// Smart search: handle blur to close dropdown (with delay for click to register)
function handleSearchBlur() {
    setTimeout(() => {
        smartSearchOpen.value = false;
        smartSearchIndex.value = -1;
    }, 150);
}

// Smart search: clear search and reset state
function clearSmartSearch() {
    tableSearch.value = '';
    smartSearchOpen.value = false;
    smartSearchIndex.value = -1;
}

// Smart search: handle keyboard navigation
function handleSearchKeydown(event) {
    const suggestions = smartSearchSuggestions.value;
    
    if (!smartSearchOpen.value || suggestions.length === 0) {
        // If dropdown is closed and user presses down, open it
        if (event.key === 'ArrowDown' && tableSearch.value.trim().length >= 2) {
            smartSearchOpen.value = true;
            smartSearchIndex.value = 0;
            event.preventDefault();
        }
        return;
    }
    
    switch(event.key) {
        case 'ArrowDown':
            event.preventDefault();
            smartSearchIndex.value = (smartSearchIndex.value + 1) % suggestions.length;
            break;
        case 'ArrowUp':
            event.preventDefault();
            smartSearchIndex.value = smartSearchIndex.value <= 0 
                ? suggestions.length - 1 
                : smartSearchIndex.value - 1;
            break;
        case 'Enter':
            event.preventDefault();
            if (smartSearchIndex.value >= 0 && smartSearchIndex.value < suggestions.length) {
                selectSuggestion(suggestions[smartSearchIndex.value]);
            }
            break;
        case 'Escape':
            smartSearchOpen.value = false;
            smartSearchIndex.value = -1;
            break;
    }
}

// Smart search: select a suggestion and jump to the incident
function selectSuggestion(suggestion) {
    smartSearchOpen.value = false;
    smartSearchIndex.value = -1;
    
    if (!suggestion.incident) return;
    
    const caseNum = suggestion.incident.case_number;
    
    // Clear search to show all results (so we can find the row)
    tableSearch.value = '';
    
    // Jump to the incident in the table
    jumpToIncidentByCase(caseNum);
}

// Jump to a specific incident by case number
function jumpToIncidentByCase(caseNum) {
    // Retry mechanism to find the row (data might still be rendering)
    let attempts = 0;
    const maxAttempts = 10;
    
    function findAndHighlight() {
        attempts++;
        const row = document.querySelector(`tr[data-case="${caseNum}"]`);
        
        if (row) {
            row.scrollIntoView({ behavior: 'smooth', block: 'center' });
            row.classList.add('highlight-row');
            // Keep highlight for 5 seconds
            setTimeout(() => row.classList.remove('highlight-row'), 5000);
        } else if (attempts < maxAttempts) {
            // Try again in 100ms
            setTimeout(findAndHighlight, 100);
        } else {
            // Final fallback: scroll to table
            const table = document.querySelector('.crime-table');
            if (table) {
                table.scrollIntoView({ behavior: 'smooth', block: 'start' });
            }
        }
    }
    
    // Start searching after a short delay to let Vue re-render
    setTimeout(findAndHighlight, 50);
}

// Filter panel: clear all filters
function clearAllFilters() {
    // Reset dropdowns
    dateFilter.value = 'all';
    timeFilter.value = 'all';
    filters.selected_neighborhood = '';
    // Clear crime type selections
    filters.selected_codes = [];
    // Reset custom date range
    filters.start_date = '';
    filters.end_date = '';
    // Clear search
    tableSearch.value = '';
    // Close panel
    crimeTypePanelOpen.value = false;
}

// Toggle entire category (for quick buttons)
function toggleCategory(category) {
    const categoryCodes = codes.value
        .filter(c => getCrimeCategory(c.code) === category)
        .map(c => c.code);
    
    const allSelected = categoryCodes.every(code => filters.selected_codes.includes(code));
    
    if (allSelected) {
        // Remove all codes in this category
        filters.selected_codes = filters.selected_codes.filter(code => !categoryCodes.includes(code));
    } else {
        // Add all codes in this category (without duplicates)
        filters.selected_codes = [...new Set([...filters.selected_codes, ...categoryCodes])];
    }
}

// Check if a category is fully active (all types selected)
function isCategoryActive(category) {
    const categoryCodes = codes.value
        .filter(c => getCrimeCategory(c.code) === category)
        .map(c => c.code);
    return categoryCodes.length > 0 && categoryCodes.every(code => filters.selected_codes.includes(code));
}

// Check if category is fully selected (for panel checkbox)
function isCategoryFullySelected(category) {
    const categoryCodes = codes.value
        .filter(c => getCrimeCategory(c.code) === category)
        .map(c => c.code);
    return categoryCodes.length > 0 && categoryCodes.every(code => filters.selected_codes.includes(code));
}

// Check if category is partially selected (for indeterminate state)
function isCategoryPartiallySelected(category) {
    const categoryCodes = codes.value
        .filter(c => getCrimeCategory(c.code) === category)
        .map(c => c.code);
    const selectedCount = categoryCodes.filter(code => filters.selected_codes.includes(code)).length;
    return selectedCount > 0 && selectedCount < categoryCodes.length;
}

// Toggle all types in a category (from panel header checkbox)
function toggleCategoryAll(category) {
    const categoryCodes = codes.value
        .filter(c => getCrimeCategory(c.code) === category)
        .map(c => c.code);
    
    const allSelected = categoryCodes.every(code => filters.selected_codes.includes(code));
    
    if (allSelected) {
        filters.selected_codes = filters.selected_codes.filter(code => !categoryCodes.includes(code));
    } else {
        filters.selected_codes = [...new Set([...filters.selected_codes, ...categoryCodes])];
    }
}

// Select all crime types
function selectAllCrimeTypes() {
    filters.selected_codes = codes.value.map(c => c.code);
}

// Clear all crime type selections
function clearCrimeTypes() {
    filters.selected_codes = [];
}

// Map search: handle blur to close dropdown
function handleMapSearchBlur() {
    setTimeout(() => {
        mapSearchOpen.value = false;
        mapSearchIndex.value = -1;
    }, 150);
}

// Map search: handle keyboard navigation
function handleMapSearchKeydown(event) {
    const suggestions = mapSearchSuggestions.value;
    
    if (!mapSearchOpen.value || suggestions.length === 0) {
        if (event.key === 'ArrowDown' && addressSearch.query.trim().length >= 2) {
            mapSearchOpen.value = true;
            mapSearchIndex.value = 0;
            event.preventDefault();
        }
        return;
    }
    
    switch(event.key) {
        case 'ArrowDown':
            event.preventDefault();
            mapSearchIndex.value = (mapSearchIndex.value + 1) % suggestions.length;
            break;
        case 'ArrowUp':
            event.preventDefault();
            mapSearchIndex.value = mapSearchIndex.value <= 0 
                ? suggestions.length - 1 
                : mapSearchIndex.value - 1;
            break;
        case 'Enter':
            event.preventDefault();
            if (mapSearchIndex.value >= 0 && mapSearchIndex.value < suggestions.length) {
                selectMapSuggestion(suggestions[mapSearchIndex.value]);
            } else {
                searchAddress(); // Normal search if no suggestion selected
            }
            break;
        case 'Escape':
            mapSearchOpen.value = false;
            mapSearchIndex.value = -1;
            break;
    }
}

// Map search: select a suggestion
function selectMapSuggestion(suggestion) {
    mapSearchOpen.value = false;
    mapSearchIndex.value = -1;
    
    if (suggestion.type === 'neighborhood') {
        // Pan to neighborhood center and filter by neighborhood
        const nIdx = suggestion.id - 1;
        if (nIdx >= 0 && nIdx < map.neighborhood_markers.length) {
            const [lat, lng] = map.neighborhood_markers[nIdx].location;
            map.leaflet.setView([lat, lng], 14, { animate: true });
            
            // Also filter the table to this neighborhood
            filters.selected_neighborhood = suggestion.id;
            
            // Scroll to table
            setTimeout(() => {
                const tableSection = document.querySelector('.table-section');
                if (tableSection) {
                    tableSection.scrollIntoView({ behavior: 'smooth', block: 'start' });
                }
            }, 800);
        }
        addressSearch.query = suggestion.display;
    } else if (suggestion.type === 'street') {
        // Search for this street and filter table
        addressSearch.query = suggestion.display;
        searchAddress();
    }
}

// Get map search category label
function getMapCategoryLabel(type) {
    switch(type) {
        case 'neighborhood': return 'Neighborhood';
        case 'street': return 'Street';
        default: return '';
    }
}

// Polygon styling function - blue default, yellow when selected
function getPolygonStyle(districtId) {
    const isSelected = filters.selected_neighborhood == districtId;
    return {
        fillColor: isSelected ? '#fbbf24' : '#3b82f6',  // yellow or blue
        fillOpacity: isSelected ? 0.5 : 0.2,
        color: isSelected ? '#f59e0b' : '#2563eb',      // border
        weight: isSelected ? 3 : 1
    };
}

// Select a neighborhood (from polygon click or hotspot chip)
function selectNeighborhood(neighborhoodId) {
    // If clicking same one, deselect
    if (filters.selected_neighborhood == neighborhoodId) {
        clearNeighborhoodSelection();
        return;
    }
    
    // Set the filter (syncs with Area dropdown)
    filters.selected_neighborhood = neighborhoodId;
    
    // Update polygon styles
    updatePolygonStyles();
}

// Clear neighborhood selection
function clearNeighborhoodSelection() {
    filters.selected_neighborhood = '';
    updatePolygonStyles();
}

// Update all polygon styles based on current selection
function updatePolygonStyles() {
    Object.entries(districtLayers.value).forEach(([id, layer]) => {
        layer.setStyle(getPolygonStyle(parseInt(id)));
    });
}

// Calculate bar width as percentage for charts
function barWidth(count, distribution) {
    const max = Math.max(...distribution, 1);
    const percent = (count / max) * 100;
    return `${percent}%`;
}

// Hour distribution with crime type breakdown (8 buckets: 12am, 3am, 6am, 9am, 12pm, 3pm, 6pm, 9pm)
// Filters by selected day when cross-filtering is active
const hourDistribution = computed(() => {
    const buckets = Array(8).fill(null).map(() => ({ violent: 0, property: 0, other: 0, total: 0 }));
    filteredIncidents.value.forEach(inc => {
        // If a day is selected, only count incidents from that day
        if (selectedDayIndex.value !== null) {
            const incDayIndex = new Date(inc.date).getDay();
            if (incDayIndex !== selectedDayIndex.value) return;
        }
        const hour = parseInt(inc.time.split(':')[0]);
        const bucket = Math.floor(hour / 3);
        const category = getCrimeCategory(inc.code);
        buckets[bucket][category]++;
        buckets[bucket].total++;
    });
    return buckets;
});

// Simple hour totals for peak calculation
const hourTotals = computed(() => hourDistribution.value.map(b => b.total));

// Day of week distribution with crime type breakdown
// Filters by selected hour bucket when cross-filtering is active
const dayDistribution = computed(() => {
    const days = Array(7).fill(null).map(() => ({ violent: 0, property: 0, other: 0, total: 0 })); // Sun-Sat
    filteredIncidents.value.forEach(inc => {
        // If an hour is selected, only count incidents from that hour bucket
        if (selectedHourBucket.value !== null) {
            const incHour = parseInt(inc.time.split(':')[0]);
            const incBucket = Math.floor(incHour / 3);
            if (incBucket !== selectedHourBucket.value) return;
        }
        const dayIndex = new Date(inc.date).getDay();
        const category = getCrimeCategory(inc.code);
        days[dayIndex][category]++;
        days[dayIndex].total++;
    });
    return days;
});

// Simple day totals for peak calculation
const dayTotals = computed(() => dayDistribution.value.map(d => d.total));

// Toggle hour bucket selection for cross-filtering
function toggleHourFilter(bucketIndex) {
    if (selectedHourBucket.value === bucketIndex) {
        selectedHourBucket.value = null; // Deselect
    } else {
        selectedHourBucket.value = bucketIndex;
    }
}

// Toggle day selection for cross-filtering
function toggleDayFilter(dayIndex) {
    if (selectedDayIndex.value === dayIndex) {
        selectedDayIndex.value = null; // Deselect
    } else {
        selectedDayIndex.value = dayIndex;
    }
}

// Clear all chart filters
function clearChartFilters() {
    selectedHourBucket.value = null;
    selectedDayIndex.value = null;
}

// Check if any chart filter is active
const hasChartFilter = computed(() => selectedHourBucket.value !== null || selectedDayIndex.value !== null);

// Calculate stacked bar widths as percentages
function stackedBarWidths(bucket, maxTotal) {
    if (!bucket || maxTotal === 0) return { violent: '0%', property: '0%', other: '0%' };
    const scale = 100 / maxTotal;
    return {
        violent: `${bucket.violent * scale}%`,
        property: `${bucket.property * scale}%`,
        other: `${bucket.other * scale}%`
    };
}

// Weekend vs weekday stats
const weekendStats = computed(() => {
    const d = dayTotals.value;
    const weekend = d[0] + d[6];  // Sun + Sat
    const weekday = d[1] + d[2] + d[3] + d[4] + d[5];
    const total = weekend + weekday;
    return {
        weekend,
        weekday,
        total,
        weekendPercent: total ? Math.round(weekend / total * 100) : 0
    };
});

// Peak hour and day
const peakStats = computed(() => {
    const hours = hourTotals.value;
    const days = dayTotals.value;
    const peakHourIndex = hours.indexOf(Math.max(...hours));
    const peakDayIndex = days.indexOf(Math.max(...days));
    return {
        peakHour: hourLabels[peakHourIndex],
        peakDay: dayLabels[peakDayIndex],
        peakHourCount: hours[peakHourIndex],
        peakDayCount: days[peakDayIndex]
    };
});

// Max values for scaling stacked bars
const maxHourTotal = computed(() => Math.max(...hourTotals.value, 1));
const maxDayTotal = computed(() => Math.max(...dayTotals.value, 1));

// Top neighborhoods in current view (for "All Areas" mode)
const topNeighborhoods = computed(() => {
    const counts = {};
    filteredIncidents.value.forEach(inc => {
        counts[inc.neighborhood_number] = (counts[inc.neighborhood_number] || 0) + 1;
    });
    return Object.entries(counts)
        .map(([id, count]) => ({
            id: Number(id),
            name: neighborhoods.value.find(n => n.id == id)?.name || 'Unknown',
            count
        }))
        .sort((a, b) => b.count - a.count)
        .slice(0, 5);
});

</script>

<template>
    <dialog id="rest-dialog" open>
        <div class="dialog-content">
            <h1 class="dialog-header">St. Paul Crime Map</h1>
            <p class="dialog-subtitle">Connect to the REST API to load crime data</p>
            
            <div class="dialog-input-group">
                <label class="dialog-label">API Server URL</label>
                <input id="dialog-url" class="dialog-input" type="url" v-model="crime_url" placeholder="http://localhost:8000" />
            </div>
            
            <p class="dialog-error" v-if="dialog_err">{{ dialog_err_msg }}</p>
            
            <div class="dialog-buttons">
                <button class="button localhost-btn" type="button" @click="useLocalhost" :disabled="localhost_loading">
                    {{ localhost_loading ? 'Connecting...' : 'Quick Connect' }}
                </button>
                <button class="button ok-btn" type="button" @click="closeDialog">Connect</button>
            </div>
            
            <p class="dialog-hint">Quick Connect uses localhost:8000</p>
        </div>
    </dialog>
    
    <!-- main layout - map + button -->
    <div class="main-layout">
        <div id="leafletmap"></div>
        
        <!-- C1: Coordinate inputs and Address search with autocomplete -->
        <div class="search-container">
            <!-- Coordinate input fields -->
            <div class="coord-inputs">
                <div class="coord-field">
                    <label class="coord-label">Latitude</label>
                    <input 
                        v-model="manualCoords.lat" 
                        type="number" 
                        step="0.000001"
                        class="coord-input"
                        @focus="handleCoordFocus"
                        @blur="handleCoordBlur"
                        @keydown.enter="updateMapFromCoords"
                        placeholder="44.955139"
                    />
                </div>
                <div class="coord-field">
                    <label class="coord-label">Longitude</label>
                    <input 
                        v-model="manualCoords.lng" 
                        type="number" 
                        step="0.000001"
                        class="coord-input"
                        @focus="handleCoordFocus"
                        @blur="handleCoordBlur"
                        @keydown.enter="updateMapFromCoords"
                        placeholder="-93.102222"
                    />
                </div>
                <button 
                    class="coord-go-button" 
                    @click="updateMapFromCoords"
                    title="Go to coordinates"
                >
                    Go
                </button>
            </div>
            
            <!-- Address search box -->
            <div class="search-box">
                <input 
                    v-model="addressSearch.query" 
                    type="text" 
                    :placeholder="addressSearch.autoUpdating ? 'Updating location...' : 'Auto-updates as map moves, or search manually...'"
                    @input="mapSearchOpen = addressSearch.query.length >= 2"
                    @focus="mapSearchOpen = addressSearch.query.length >= 2"
                    @blur="handleMapSearchBlur"
                    @keydown="handleMapSearchKeydown"
                    :class="['search-input', { 'auto-updating': addressSearch.autoUpdating }]"
                    autocomplete="off"
                />
                <button 
                    class="search-button" 
                    @click="searchAddress"
                    :disabled="addressSearch.searching || addressSearch.autoUpdating"
                >
                    {{ addressSearch.searching ? '...' : 'Go' }}
                </button>
                <button 
                    v-if="addressSearch.marker"
                    class="clear-search-button" 
                    @click="clearSearch"
                    title="Remove search marker"
                >
                    ✕
                </button>
                
                <!-- Map search dropdown -->
                <div 
                    v-if="mapSearchOpen && mapSearchSuggestions.length > 0" 
                    class="map-search-dropdown"
                >
                    <div 
                        v-for="(suggestion, index) in mapSearchSuggestions" 
                        :key="`${suggestion.type}-${suggestion.label}`"
                        class="map-search-item"
                        :class="{ 
                            'selected': index === mapSearchIndex,
                            [`type-${suggestion.type}`]: true
                        }"
                        @mousedown.prevent="selectMapSuggestion(suggestion)"
                        @mouseenter="mapSearchIndex = index"
                    >
                        <span class="map-suggestion-category">{{ getMapCategoryLabel(suggestion.type) }}</span>
                        <span class="map-suggestion-text" v-html="highlightMatch(suggestion.display, addressSearch.query)"></span>
                    </div>
                </div>
            </div>
            <p v-if="addressSearch.error" class="search-error">{{ addressSearch.error }}</p>
        </div>
        
        <!-- button to show form - positioned over map in top right -->
        <button class="report-button" @click="toggleForm">
            {{ form_visible ? 'Close Form' : 'Report New Incident' }}
        </button>
        <a href="about.html" class="about-link">About</a>
    </div>

    <!-- Analytics Panel -->
    <div class="analytics-section">
        <div class="analytics-panel" v-if="!analyticsCollapsed && incidents.length > 0">
            <div class="analytics-header">
                <h3 class="analytics-title">
                    Crime Analytics: 
                    <span class="analytics-area-name">
                        {{ filters.selected_neighborhood 
                            ? neighborhoods.find(n => n.id == filters.selected_neighborhood)?.name 
                            : 'All Visible Areas' }}
                    </span>
                </h3>
                <div class="analytics-actions">
                    <button 
                        v-if="filters.selected_neighborhood" 
                        class="show-all-btn"
                        @click="clearNeighborhoodSelection"
                    >
                        ↺ Show All
                    </button>
                    <button class="collapse-btn" @click="analyticsCollapsed = true">
                        Collapse ▼
                    </button>
                </div>
            </div>
            
            <p class="analytics-summary">
                <strong>{{ filteredIncidents.length }}</strong> incidents
                <span v-if="filters.selected_neighborhood"> in selected area</span>
                <span v-else> currently visible</span>
            </p>
            
            <!-- Compact crime type legend -->
            <div class="crime-legend">
                <span class="legend-item"><span class="legend-dot violent"></span>Violent</span>
                <span class="legend-item"><span class="legend-dot property"></span>Property</span>
                <span class="legend-item"><span class="legend-dot other"></span>Other</span>
            </div>
            
            <div class="charts-row">
                <!-- Time of Day Chart -->
                <div class="chart-container">
                    <h4 class="chart-title">
                        Time of Day
                        <span v-if="selectedDayIndex !== null" class="filter-badge">
                            {{ dayLabels[selectedDayIndex] }} only
                            <button class="clear-filter-x" @click.stop="selectedDayIndex = null">×</button>
                        </span>
                    </h4>
                    <div class="bar-chart">
                        <div 
                            v-for="(bucket, i) in hourDistribution" 
                            :key="'hour-'+i" 
                            class="bar-row clickable-bar"
                            :class="{ 'selected-bar': selectedHourBucket === i }"
                            :title="`Click to filter by ${hourLabels[i]} | ${bucket.violent} violent, ${bucket.property} property, ${bucket.other} other`"
                            @click="toggleHourFilter(i)"
                        >
                            <span class="bar-label">{{ hourLabels[i] }}</span>
                            <div class="bar-track stacked">
                                <div 
                                    class="bar-segment violent" 
                                    :style="{ width: stackedBarWidths(bucket, maxHourTotal).violent }"
                                ></div>
                                <div 
                                    class="bar-segment property" 
                                    :style="{ width: stackedBarWidths(bucket, maxHourTotal).property }"
                                ></div>
                                <div 
                                    class="bar-segment other" 
                                    :style="{ width: stackedBarWidths(bucket, maxHourTotal).other }"
                                ></div>
                            </div>
                            <span class="bar-count">{{ bucket.total }}</span>
                        </div>
                    </div>
                    <p class="chart-insight">
                        <span class="insight-icon">⏰</span> Peak: <strong>{{ peakStats.peakHour }}</strong>
                        <span class="insight-count">({{ peakStats.peakHourCount }})</span>
                    </p>
                </div>
                
                <!-- Day of Week Chart -->
                <div class="chart-container">
                    <h4 class="chart-title">
                        Day of Week
                        <span v-if="selectedHourBucket !== null" class="filter-badge">
                            {{ hourLabels[selectedHourBucket] }} only
                            <button class="clear-filter-x" @click.stop="selectedHourBucket = null">×</button>
                        </span>
                    </h4>
                    <div class="bar-chart">
                        <div 
                            v-for="(bucket, i) in dayDistribution" 
                            :key="'day-'+i" 
                            class="bar-row clickable-bar"
                            :class="{ 
                                'weekend-row': i === 0 || i === 6,
                                'selected-bar': selectedDayIndex === i
                            }"
                            :title="`Click to filter by ${dayLabels[i]} | ${bucket.violent} violent, ${bucket.property} property, ${bucket.other} other`"
                            @click="toggleDayFilter(i)"
                        >
                            <span class="bar-label">{{ dayLabels[i] }}</span>
                            <div class="bar-track stacked">
                                <div 
                                    class="bar-segment violent" 
                                    :style="{ width: stackedBarWidths(bucket, maxDayTotal).violent }"
                                ></div>
                                <div 
                                    class="bar-segment property" 
                                    :style="{ width: stackedBarWidths(bucket, maxDayTotal).property }"
                                ></div>
                                <div 
                                    class="bar-segment other" 
                                    :style="{ width: stackedBarWidths(bucket, maxDayTotal).other }"
                                ></div>
                            </div>
                            <span class="bar-count">{{ bucket.total }}</span>
                        </div>
                    </div>
                    <p class="chart-insight">
                        <span class="insight-icon">📅</span> Weekends: <strong>{{ weekendStats.weekend }}</strong>
                        <span class="insight-percent">({{ weekendStats.weekendPercent }}%)</span>
                    </p>
                </div>
            </div>
            
            <!-- Top Neighborhoods (only when no specific area selected) -->
            <div v-if="!filters.selected_neighborhood && topNeighborhoods.length > 0" class="top-areas">
                <h4 class="chart-title">Top Areas in View</h4>
                <div class="hotspot-list">
                    <button 
                        v-for="n in topNeighborhoods" 
                        :key="n.id"
                        class="hotspot-chip"
                        @click="selectNeighborhood(n.id)"
                    >
                        <span class="hotspot-name">{{ n.name }}</span>
                        <span class="hotspot-count">{{ n.count }}</span>
                    </button>
                </div>
            </div>
        </div>
        
        <!-- Collapsed state -->
        <button 
            v-else-if="analyticsCollapsed && incidents.length > 0" 
            class="analytics-expand" 
            @click="analyticsCollapsed = false"
        >
            📊 Show Analytics ▲
        </button>
    </div>

    <!-- Crime table section -->
<div class="table-section">
    
    <!-- Redesigned filter bar with better organization -->
    <div class="filter-bar">
        <!-- Top row: Results and clear button -->
        <div class="filter-bar-header">
            <div class="results-badge">
                <span class="results-count">{{ filteredIncidents.length }}</span>
                <span class="results-text">incidents</span>
            </div>
            
            <button 
                v-if="hasActiveFilters"
                class="clear-all-btn"
                @click="clearAllFilters"
                title="Clear all filters"
            >
                <span>✕</span> Clear All
            </button>
        </div>
        
        <!-- Filter controls grid -->
        <div class="filter-controls-grid">
            <!-- Date filter -->
            <div class="filter-group">
                <label class="filter-label">Date</label>
                <select v-model="dateFilter" class="filter-select" :class="{ active: dateFilter !== 'all' }">
                    <option value="all">All dates</option>
                    <option value="today">Today</option>
                    <option value="week">Last 7 days</option>
                    <option value="month">Last 30 days</option>
                    <option value="custom">Custom range...</option>
                </select>
            </div>
            
            <!-- Time filter (NEW) -->
            <div class="filter-group">
                <label class="filter-label">Time</label>
                <select v-model="timeFilter" class="filter-select" :class="{ active: timeFilter !== 'all' }">
                    <option value="all">All times</option>
                    <optgroup label="Time of Day">
                        <option value="morning">Morning (6am-12pm)</option>
                        <option value="afternoon">Afternoon (12pm-6pm)</option>
                        <option value="evening">Evening (6pm-10pm)</option>
                        <option value="night">Night (10pm-6am)</option>
                    </optgroup>
                    <optgroup label="Schedule">
                        <option value="business">Business Hours (9am-5pm)</option>
                        <option value="weekdays">Weekdays</option>
                        <option value="weekends">Weekends</option>
                    </optgroup>
                </select>
            </div>
            
            <!-- Type button (opens panel) -->
            <div class="filter-group">
                <label class="filter-label">Type</label>
                <button 
                    class="filter-select type-btn"
                    :class="{ active: filters.selected_codes.length > 0 || crimeTypePanelOpen }"
                    @click="crimeTypePanelOpen = !crimeTypePanelOpen"
                >
                    <span class="type-btn-text">{{ filters.selected_codes.length > 0 ? `${filters.selected_codes.length} selected` : 'All types' }}</span>
                    <span class="dropdown-arrow">{{ crimeTypePanelOpen ? '▲' : '▼' }}</span>
                </button>
            </div>
            
            <!-- Area dropdown -->
            <div class="filter-group">
                <label class="filter-label">Area</label>
                <select v-model="filters.selected_neighborhood" class="filter-select" :class="{ active: filters.selected_neighborhood !== '' }">
                    <option value="">All neighborhoods</option>
                    <option v-for="n in neighborhoods" :key="n.id" :value="n.id">
                        {{ n.name }}
                    </option>
                </select>
            </div>
        </div>
        
        <!-- Expandable crime type panel -->
        <div v-if="crimeTypePanelOpen" class="crime-type-panel">
            <div class="crime-type-columns">
                <!-- Violent crimes column -->
                <div class="crime-column">
                    <div class="column-header violent">
                        <input 
                            type="checkbox" 
                            :checked="isCategoryFullySelected('violent')"
                            :indeterminate.prop="isCategoryPartiallySelected('violent')"
                            @change="toggleCategoryAll('violent')"
                        />
                        <span>Violent</span>
                    </div>
                    <label v-for="c in violentCrimes" :key="c.code" class="crime-checkbox">
                        <input 
                            type="checkbox" 
                            :value="c.code" 
                            v-model="filters.selected_codes"
                        />
                        <span>{{ c.type }}</span>
                    </label>
                </div>
                
                <!-- Property crimes column -->
                <div class="crime-column">
                    <div class="column-header property">
                        <input 
                            type="checkbox" 
                            :checked="isCategoryFullySelected('property')"
                            :indeterminate.prop="isCategoryPartiallySelected('property')"
                            @change="toggleCategoryAll('property')"
                        />
                        <span>Property</span>
                    </div>
                    <label v-for="c in propertyCrimes" :key="c.code" class="crime-checkbox">
                        <input 
                            type="checkbox" 
                            :value="c.code" 
                            v-model="filters.selected_codes"
                        />
                        <span>{{ c.type }}</span>
                    </label>
                </div>
                
                <!-- Other crimes column -->
                <div class="crime-column">
                    <div class="column-header other">
                        <input 
                            type="checkbox" 
                            :checked="isCategoryFullySelected('other')"
                            :indeterminate.prop="isCategoryPartiallySelected('other')"
                            @change="toggleCategoryAll('other')"
                        />
                        <span>Other</span>
                    </div>
                    <label v-for="c in otherCrimes" :key="c.code" class="crime-checkbox">
                        <input 
                            type="checkbox" 
                            :value="c.code" 
                            v-model="filters.selected_codes"
                        />
                        <span>{{ c.type }}</span>
                    </label>
                </div>
            </div>
            
            <!-- Panel actions -->
            <div class="panel-actions">
                <span class="selection-count">{{ filters.selected_codes.length }} types selected</span>
                <button class="panel-btn" @click="selectAllCrimeTypes">Select All</button>
                <button class="panel-btn" @click="clearCrimeTypes">Clear</button>
                <button class="panel-btn primary" @click="crimeTypePanelOpen = false">Done</button>
            </div>
        </div>
        
        <!-- Custom date range (shows when Date: Custom is selected) -->
        <div v-if="dateFilter === 'custom'" class="custom-date-row">
            <label>From:</label>
            <input type="date" v-model="filters.start_date" class="date-input" />
            <label>To:</label>
            <input type="date" v-model="filters.end_date" class="date-input" />
        </div>
    </div>
    
    <!-- Geocoding status message -->
    <div v-if="geocodingStatus" class="geocoding-status">
        {{ geocodingStatus }}
    </div>
    
    <!-- crime table -->
    <div class="table-wrapper">
        <table class="crime-table">
            <thead>
                <tr>
                    <th class="sortable" @click="sortBy('case_number')">
                        Case #{{ getSortIndicator('case_number') }}
                    </th>
                    <th class="sortable" @click="sortBy('date')">
                        Date{{ getSortIndicator('date') }}
                    </th>
                    <th class="sortable" @click="sortBy('time')">
                        Time{{ getSortIndicator('time') }}
                    </th>
                    <th class="sortable" @click="sortBy('incident')">
                        Incident{{ getSortIndicator('incident') }}
                    </th>
                    <th class="sortable" @click="sortBy('block')">
                        Block{{ getSortIndicator('block') }}
                    </th>
                    <th class="sortable" @click="sortBy('neighborhood')">
                        Neighborhood{{ getSortIndicator('neighborhood') }}
                    </th>
                    <th>Actions</th>
                </tr>
            </thead>
            <tbody>
                <tr v-for="inc in filteredIncidents" :key="inc.case_number" :class="getCrimeCategory(inc.code)" :data-case="inc.case_number">
                    <td>{{ inc.case_number }}</td>
                    <td>{{ inc.date }}</td>
                    <td>{{ inc.time }}</td>
                    <td>{{ inc.incident }}</td>
                    <td>{{ inc.block }}</td>
                    <td>{{ neighborhoods.find(n => n.id === inc.neighborhood_number)?.name || inc.neighborhood_number }}</td>
<td>
                                        <button 
                                            class="marker-btn" 
                                            :class="{ 
                                                active: crimeMarkers[inc.case_number],
                                                loading: geocodingInProgress[inc.case_number]
                                            }"
                                            @click="toggleCrimeMarker(inc)"
                                            :disabled="geocodingInProgress[inc.case_number]"
                                            :title="geocodingInProgress[inc.case_number] ? 'Locating...' : (crimeMarkers[inc.case_number] ? 'Remove from map' : 'Show on map')"
                                        >
                                            <!-- Circular progress ring during loading -->
                                            <svg v-if="geocodingInProgress[inc.case_number]" 
                                                 class="progress-ring" 
                                                 width="28" height="28" viewBox="0 0 28 28">
                                                <!-- Grey background ring -->
                                                <circle 
                                                    class="progress-ring-bg"
                                                    cx="14" cy="14" r="11"
                                                    fill="none"
                                                    stroke="#d0d0d0"
                                                    stroke-width="3"
                                                />
                                                <!-- Green progress ring -->
                                                <circle 
                                                    class="progress-ring-fill"
                                                    cx="14" cy="14" r="11"
                                                    fill="none"
                                                    stroke="#4caf50"
                                                    stroke-width="3"
                                                    stroke-linecap="round"
                                                    :stroke-dasharray="69.1"
                                                    :stroke-dashoffset="69.1 - (69.1 * (geocodingProgress[inc.case_number] || 0) / 100)"
                                                    transform="rotate(-90 14 14)"
                                                />
                                                <!-- Number in center (no % to keep it centered) -->
                                                <text x="14" y="15" 
                                                      class="progress-text"
                                                      text-anchor="middle" 
                                                      dominant-baseline="middle"
                                                      font-size="9" 
                                                      font-weight="bold"
                                                      font-family="system-ui, -apple-system, sans-serif"
                                                      :fill="(geocodingProgress[inc.case_number] || 0) >= 100 ? '#4caf50' : '#555'">
                                                    {{ Math.round(geocodingProgress[inc.case_number] || 0) }}
                                                </text>
                                            </svg>
                                            <!-- Normal state: pin or checkmark -->
                                            <span v-else>{{ crimeMarkers[inc.case_number] ? '✓' : '📍' }}</span>
                                        </button>
                                        <button class="delete-btn" @click="deleteIncident(inc.case_number)">🗑️</button>
                                    </td>
                </tr>
            </tbody>
        </table>
    </div>
</div>

    
    <!-- new incident form - slides in from right when visible -->
    <div class="incident-form" :class="{ visible: form_visible }">
        <div class="form-header">
            <h3>Report New Incident</h3>
            <button class="close-form" @click="toggleForm">×</button>
        </div>
        
        <div class="form-section">
            <div class="form-row">
                <div class="form-field">
                    <label>Date</label>
                    <input v-model="newIncident.date" type="date" />
                </div>
                <div class="form-field">
                    <label>Time</label>
                    <input v-model="newIncident.time" type="time" />
                </div>
            </div>
            
            <label>Crime Type <span class="required">*</span></label>
            <select v-model="newIncident.code" @change="autofillIncidentType">
                <option value="">-- Select crime type --</option>
                <option v-for="c in codes" :key="c.code" :value="c.code">
                    {{ c.type }}
                </option>
            </select>
            
            <label>Neighborhood <span class="required">*</span></label>
            <select v-model="newIncident.neighborhood_number" @change="autofillPoliceGrid">
                <option value="">-- Select neighborhood --</option>
                <option v-for="n in neighborhoods" :key="n.id" :value="n.id">
                    {{ n.name }}
                </option>
            </select>
            
            <label>Block Address <span class="required">*</span></label>
            <input v-model="newIncident.block" placeholder="e.g. 7XX UNIVERSITY AVE" />
            <span class="field-hint">Use XX for privacy (e.g. 12XX MAIN ST)</span>
            
            <label>Description</label>
            <input v-model="newIncident.incident" placeholder="Brief description of incident" />
        </div>
        
        <div class="form-section form-advanced" v-if="showAdvancedFields">
            <label>Case Number</label>
            <input v-model="newIncident.case_number" placeholder="Auto-generated if blank" />
            
            <label>Police Grid</label>
            <input v-model="newIncident.police_grid" type="number" placeholder="Auto-assigned based on neighborhood" />
        </div>
        
        <button type="button" class="toggle-advanced" @click="showAdvancedFields = !showAdvancedFields">
            {{ showAdvancedFields ? '▼ Hide advanced fields' : '▶ Show advanced fields' }}
        </button>
        
        <button class="button submit-btn" @click="submitIncident">Submit Report</button>
        
        <p v-if="form_error" class="form-error">{{ form_error }}</p>
    </div>
    
    <!-- Success popup overlay -->
    <div class="success-overlay" v-if="successPopup" @click.self="closeSuccessPopup">
        <div class="success-popup">
            <div class="success-icon">✓</div>
            <h2>Incident Reported!</h2>
            <p class="success-case">Case #{{ lastAddedCase }}</p>
            <p class="success-message">Your incident has been added to the database and will appear in the Crime Incidents table below.</p>
            <div class="success-buttons">
                <button class="btn-jump" @click="jumpToIncident">Jump to Incident</button>
                <button class="btn-close" @click="closeSuccessPopup">Close</button>
            </div>
        </div>
    </div>
</template>

<style scoped>
.about-link {
    position: absolute;
    top: 1.5rem;
    right: 16rem;
    padding: 0.75rem 1.5rem;
    background: #1e3a5f;
    color: white;
    text-decoration: none;
    border-radius: 6px;
    font-weight: 600;
    font-size: 1rem;
    line-height: 1.2;
    display: inline-block;
    z-index: 500;
    box-shadow: 0 2px 8px rgba(0,0,0,0.3);
}
.about-link:hover {
    background: #2d5a87;
}
/* dialog for api url */
#rest-dialog {
    width: 340px;
    margin: 0;
    position: fixed;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    z-index: 1000;
    padding: 0;
    border-radius: 16px;
    border: none;
    box-shadow: 0 8px 32px rgba(0,0,0,0.2);
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    overflow: hidden;
}

#rest-dialog::backdrop {
    background: rgba(0,0,0,0.5);
    backdrop-filter: blur(4px);
}

.dialog-content {
    background: white;
    margin: 3px;
    border-radius: 14px;
    padding: 1.25rem;
    text-align: center;
}

.dialog-header {
    font-size: 1.25rem;
    font-weight: 700;
    color: #1f2937;
    margin: 0 0 0.25rem 0;
}

.dialog-subtitle {
    font-size: 0.85rem;
    color: #6b7280;
    margin: 0 0 1rem 0;
}

.dialog-input-group {
    text-align: left;
    margin-bottom: 0.75rem;
}

.dialog-label {
    font-size: 0.8rem;
    font-weight: 600;
    color: #374151;
    display: block;
    margin-bottom: 0.25rem;
}

.dialog-input {
    font-size: 0.9rem;
    width: 100%;
    padding: 0.6rem 0.75rem;
    border: 2px solid #e5e7eb;
    border-radius: 8px;
    transition: border-color 0.2s;
    box-sizing: border-box;
}

.dialog-input:focus {
    outline: none;
    border-color: #667eea;
}

.dialog-error {
    font-size: 0.8rem;
    color: #dc2626;
    background: #fef2f2;
    padding: 0.5rem 0.75rem;
    border-radius: 6px;
    margin: 0 0 0.75rem 0;
    text-align: left;
}

/* Dialog buttons container */
.dialog-buttons {
    display: flex;
    gap: 0.5rem;
}

.dialog-buttons .button {
    flex: 1;
    padding: 0.65rem 0.5rem;
    font-size: 0.85rem;
    border-radius: 8px;
    cursor: pointer;
    font-weight: 600;
    transition: all 0.2s ease;
    border: none;
}

/* Quick connect button */
.localhost-btn {
    background: linear-gradient(135deg, #10b981 0%, #059669 100%) !important;
    color: white !important;
}

.localhost-btn:hover:not(:disabled) {
    transform: translateY(-1px);
    box-shadow: 0 4px 12px rgba(16, 185, 129, 0.4);
}

.localhost-btn:disabled {
    background: #9ca3af !important;
    cursor: wait;
    transform: none;
    box-shadow: none;
}

/* Connect button */
.ok-btn {
    background: linear-gradient(135deg, #3b82f6 0%, #2563eb 100%);
    color: white;
}

.ok-btn:hover {
    transform: translateY(-1px);
    box-shadow: 0 4px 12px rgba(59, 130, 246, 0.4);
}

.dialog-hint {
    font-size: 0.75rem;
    color: #9ca3af;
    margin: 0.75rem 0 0 0;
}

/* main layout - full screen with padding */
.main-layout {
    position: relative;
    padding: 1rem 1rem 0;
    height: 55vh;
}

/* map fills the space */
#leafletmap {
    width: 100%;
    height: 100%;
    border-radius: 12px 12px 0 0;
}

/* C1: Address search container - conservative light theme */
.search-container {
    position: absolute;
    bottom: 1.5rem;
    left: 1.5rem;
    z-index: 500;
    background: #f8f9fa;
    padding: 0.75rem;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.15);
    min-width: 380px;
    border: 1px solid #e5e7eb;
}

/* Coordinate inputs */
.coord-inputs {
    display: flex;
    gap: 0.5rem;
    margin-bottom: 0.75rem;
    align-items: flex-end;
}

.coord-field {
    flex: 1;
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
}

.coord-label {
    font-size: 0.7rem;
    font-weight: 600;
    color: #6b7280;
    text-transform: uppercase;
    letter-spacing: 0.3px;
}

.coord-input {
    padding: 0.5rem 0.6rem;
    background: white;
    border: 1px solid #d1d5db;
    border-radius: 6px;
    font-size: 0.85rem;
    color: #374151;
    font-family: 'Courier New', monospace;
    transition: border-color 0.15s;
}

.coord-input:focus {
    outline: none;
    border-color: #6b7280;
    box-shadow: 0 0 0 2px rgba(107, 114, 128, 0.1);
}

.coord-input::placeholder {
    color: #9ca3af;
    font-family: 'Courier New', monospace;
}

.coord-go-button {
    padding: 0.5rem 0.9rem;
    background: #10b981;
    color: white;
    border: none;
    border-radius: 6px;
    font-weight: 600;
    font-size: 0.85rem;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.15s;
    height: fit-content;
}

.coord-go-button:hover {
    background: #059669;
}

.search-box {
    display: flex;
    gap: 0.5rem;
    align-items: center;
    position: relative;
}

.search-input {
    flex: 1;
    padding: 0.5rem 0.75rem;
    background: white;
    border: 1px solid #d1d5db;
    border-radius: 6px;
    font-size: 0.9rem;
    color: #374151;
    transition: all 0.3s ease;
}

.search-input::placeholder {
    color: #9ca3af;
    font-size: 0.85rem;
}

.search-input:focus {
    outline: none;
    border-color: #6b7280;
    box-shadow: 0 0 0 2px rgba(107, 114, 128, 0.1);
}

.search-input.auto-updating {
    border-color: #10b981;
    background: #f0fdf4;
    animation: pulseGreen 2s ease-in-out infinite;
}

@keyframes pulseGreen {
    0%, 100% {
        border-color: #10b981;
        box-shadow: 0 0 0 0 rgba(16, 185, 129, 0.4);
    }
    50% {
        border-color: #059669;
        box-shadow: 0 0 0 3px rgba(16, 185, 129, 0.2);
    }
}

.search-button {
    padding: 0.5rem 0.9rem;
    background: #4b5563;
    color: white;
    border: none;
    border-radius: 6px;
    font-weight: 500;
    font-size: 0.85rem;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.15s;
}

.search-button:hover:not(:disabled) {
    background: #374151;
}

.search-button:disabled {
    background: #d1d5db;
    color: #9ca3af;
    cursor: not-allowed;
}

.clear-search-button {
    padding: 0.5rem 0.6rem;
    background: #ef4444;
    color: white;
    border: none;
    border-radius: 6px;
    font-weight: 500;
    cursor: pointer;
    font-size: 0.85rem;
    line-height: 1;
}

.clear-search-button:hover {
    background: #dc2626;
}

.search-error {
    color: #fca5a5;
    font-size: 0.8rem;
    margin-top: 0.5rem;
    margin-bottom: 0;
}

/* button to open form - floats over map in top right */
.report-button {
    position: absolute;
    top: 1.5rem;
    right: 1.5rem;
    padding: 0.75rem 1.5rem;
    background: #3b82f6;
    color: white;
    border: none;
    border-radius: 6px;
    font-weight: 600;
    font-size: 1rem;
    line-height: 1.2;
    cursor: pointer;
    z-index: 500;
    box-shadow: 0 2px 8px rgba(0,0,0,0.3);
}
.report-button:hover {
    background: #2563eb;
}

/* incident form - slides in from right */
.incident-form {
    position: fixed;
    top: 1rem;
    right: -450px; /* hidden off screen */
    width: 400px;
    height: calc(100vh - 4rem);
    background: white;
    padding: 2rem;
    overflow-y: auto;
    box-shadow: -2px 0 10px rgba(0,0,0,0.2);
    transition: right 0.3s ease; /* smooth slide */
    z-index: 1500; /* above leaflet controls */
    border-radius: 8px;
}
.incident-form.visible {
    right: 1rem; /* slides in when visible */
}

/* form header with title and close button */
.form-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 1rem;
}
.form-header h3 {
    margin: 0;
    color: #333;
}
.close-form {
    background: #f5f5f5 !important;
    border: 1px solid #ddd !important;
    font-size: 1.25rem;
    color: #666 !important;
    cursor: pointer;
    padding: 0.25rem 0.75rem !important;
    border-radius: 4px;
    line-height: 1;
    width: auto !important;
    margin: 0 !important;
}
.close-form:hover {
    background: #e5e5e5 !important;
    color: #333 !important;
}

/* form sections */
.form-section {
    margin-bottom: 0.5rem;
}

.form-row {
    display: flex;
    gap: 0.75rem;
}

.form-field {
    flex: 1;
}

.form-field label {
    margin-top: 0;
}

/* form fields */
.incident-form label {
    display: block;
    margin-top: 0.75rem;
    margin-bottom: 0.25rem;
    font-weight: 600;
    color: #374151;
    font-size: 0.9rem;
}

.incident-form label .required {
    color: #dc2626;
}

.incident-form input, 
.incident-form select {
    width: 100%;
    padding: 0.6rem 0.75rem;
    border: 2px solid #e5e7eb;
    border-radius: 6px;
    font-size: 0.9rem;
    transition: border-color 0.2s;
    box-sizing: border-box;
}

.incident-form input:focus,
.incident-form select:focus {
    outline: none;
    border-color: #3b82f6;
}

.field-hint {
    display: block;
    font-size: 0.75rem;
    color: #6b7280;
    margin-top: 0.25rem;
}

/* advanced fields section */
.form-advanced {
    background: #f9fafb;
    padding: 0.75rem;
    border-radius: 6px;
    margin-top: 0.5rem;
}

.toggle-advanced {
    width: 100%;
    background: none;
    border: none;
    color: #6b7280;
    font-size: 0.8rem;
    cursor: pointer;
    padding: 0.5rem;
    margin-top: 0.5rem;
    text-align: left;
}

.toggle-advanced:hover {
    color: #374151;
    background: #f3f4f6;
    border-radius: 4px;
}

/* submit button */
.submit-btn {
    width: 100%;
    margin-top: 1rem !important;
    padding: 0.75rem;
    background: linear-gradient(135deg, #3b82f6 0%, #2563eb 100%);
    color: white;
    border: none;
    border-radius: 6px;
    font-weight: 600;
    font-size: 1rem;
    cursor: pointer;
    transition: all 0.2s;
}

.submit-btn:hover {
    transform: translateY(-1px);
    box-shadow: 0 4px 12px rgba(59, 130, 246, 0.4);
}

/* error and success messages */
.form-error { 
    color: #D32323; 
    margin-top: 0.5rem;
}

/* Success popup overlay */
.success-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.5);
    backdrop-filter: blur(4px);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 2000;
    animation: fadeIn 0.2s ease;
}

@keyframes fadeIn {
    from { opacity: 0; }
    to { opacity: 1; }
}

.success-popup {
    background: white;
    border-radius: 16px;
    padding: 2rem;
    text-align: center;
    max-width: 400px;
    box-shadow: 0 10px 40px rgba(0, 0, 0, 0.2);
    animation: popIn 0.3s ease;
}

@keyframes popIn {
    from { 
        transform: scale(0.8);
        opacity: 0;
    }
    to { 
        transform: scale(1);
        opacity: 1;
    }
}

.success-icon {
    width: 64px;
    height: 64px;
    background: linear-gradient(135deg, #10b981 0%, #059669 100%);
    color: white;
    font-size: 2rem;
    font-weight: bold;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    margin: 0 auto 1rem;
}

.success-popup h2 {
    color: #1f2937;
    margin: 0 0 0.5rem;
    font-size: 1.5rem;
}

.success-case {
    color: #6b7280;
    font-size: 0.9rem;
    margin: 0 0 0.75rem;
    font-family: monospace;
    background: #f3f4f6;
    padding: 0.25rem 0.75rem;
    border-radius: 4px;
    display: inline-block;
}

.success-message {
    color: #4b5563;
    font-size: 0.95rem;
    margin: 0 0 1.5rem;
    line-height: 1.5;
}

.success-buttons {
    display: flex;
    gap: 0.75rem;
}

.success-buttons button {
    flex: 1;
    padding: 0.75rem 1rem;
    border-radius: 8px;
    font-weight: 600;
    font-size: 0.95rem;
    cursor: pointer;
    transition: all 0.2s;
    border: none;
}

.btn-jump {
    background: linear-gradient(135deg, #3b82f6 0%, #2563eb 100%);
    color: white;
}

.btn-jump:hover {
    transform: translateY(-1px);
    box-shadow: 0 4px 12px rgba(59, 130, 246, 0.4);
}

.btn-close {
    background: #f3f4f6;
    color: #374151;
}

.btn-close:hover {
    background: #e5e7eb;
}

/* Highlight newly added row */
.crime-table tbody tr.highlight-row {
    animation: highlightPulse 0.5s ease infinite alternate;
    background-color: #4ade80 !important;
    outline: 4px solid #16a34a;
    outline-offset: -2px;
}

.crime-table tbody tr.highlight-row td {
    font-weight: 700 !important;
    color: #14532d !important;
}

@keyframes highlightPulse {
    from { 
        background-color: #4ade80;
        outline-color: #16a34a;
    }
    to { 
        background-color: #22c55e;
        outline-color: #15803d;
    }
}

/* table section - connects directly with map */
.table-section {
    padding: 0 1rem 1rem;
    max-width: 1400px;
    margin: -1px auto 0;
}

/* ========== REDESIGNED FILTER BAR ========== */
.filter-bar {
    background: linear-gradient(to bottom, #ffffff 0%, #f9fafb 100%);
    border: 1px solid #e5e7eb;
    border-radius: 12px 12px 0 0;
    padding: 1.25rem 1.5rem;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

/* Top header with results and clear button */
.filter-bar-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 1rem;
    padding-bottom: 0.75rem;
    border-bottom: 2px solid #e5e7eb;
}

/* Results badge */
.results-badge {
    display: flex;
    align-items: baseline;
    gap: 0.5rem;
    background: linear-gradient(135deg, #3b82f6 0%, #2563eb 100%);
    padding: 0.5rem 1rem;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(59, 130, 246, 0.3);
}

.results-count {
    font-size: 1.5rem;
    font-weight: 800;
    color: white;
    line-height: 1;
}

.results-text {
    font-size: 0.9rem;
    font-weight: 600;
    color: rgba(255, 255, 255, 0.9);
    text-transform: lowercase;
}

/* Clear all button */
.clear-all-btn {
    display: flex;
    align-items: center;
    gap: 0.4rem;
    padding: 0.5rem 1rem;
    background: white;
    border: 2px solid #ef4444;
    border-radius: 8px;
    color: #ef4444;
    font-size: 0.85rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.2s ease;
}

.clear-all-btn:hover {
    background: #ef4444;
    color: white;
    transform: translateY(-1px);
    box-shadow: 0 4px 12px rgba(239, 68, 68, 0.3);
}

.clear-all-btn span:first-child {
    font-size: 1rem;
    font-weight: bold;
}

/* Filter controls grid */
.filter-controls-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
    gap: 1rem;
    align-items: start;
}

/* Individual filter group */
.filter-group {
    display: flex;
    flex-direction: column;
    gap: 0.4rem;
}

/* Filter labels */
.filter-label {
    font-size: 0.75rem;
    font-weight: 700;
    color: #6b7280;
    text-transform: uppercase;
    letter-spacing: 0.5px;
}

/* Select styling - cleaner and more modern */
.filter-select {
    width: 100%;
    padding: 0.65rem 0.85rem;
    padding-right: 2rem;
    background: white;
    border: 2px solid #e5e7eb;
    border-radius: 8px;
    color: #374151;
    font-size: 0.875rem;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.2s ease;
    appearance: none;
    background-image: url("data:image/svg+xml,%3csvg xmlns='http://www.w3.org/2000/svg' fill='none' viewBox='0 0 20 20'%3e%3cpath stroke='%236b7280' stroke-linecap='round' stroke-linejoin='round' stroke-width='2' d='M6 8l4 4 4-4'/%3e%3c/svg%3e");
    background-position: right 0.65rem center;
    background-repeat: no-repeat;
    background-size: 1.2em 1.2em;
}

.filter-select:hover {
    border-color: #d1d5db;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.filter-select:focus {
    outline: none;
    border-color: #3b82f6;
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

/* Active filter indicator */
.filter-select.active {
    border-color: #3b82f6;
    background-color: #eff6ff;
    font-weight: 600;
    color: #1e40af;
}

.filter-select option {
    background: white;
    color: #374151;
    padding: 0.5rem;
}

.filter-select optgroup {
    font-weight: 700;
    color: #6b7280;
    font-size: 0.8rem;
}

/* ========== TYPE BUTTON AND PANEL ========== */

/* Type button (looks like a dropdown) */
.type-btn {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 0.75rem;
    width: 100%;
    text-align: left;
}

.type-btn-text {
    flex: 1;
}

.type-btn .dropdown-arrow {
    font-size: 0.75rem;
    color: #6b7280;
    transition: transform 0.2s ease;
}

.type-btn.active .dropdown-arrow {
    color: #1e40af;
}

/* Crime type panel */
.crime-type-panel {
    background: #f9fafb;
    border-top: 2px solid #e5e7eb;
    padding: 1.25rem 1.5rem 1rem;
    margin-top: 1rem;
    border-radius: 0 0 8px 8px;
    animation: slideDown 0.2s ease-out;
}

@keyframes slideDown {
    from {
        opacity: 0;
        transform: translateY(-10px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

.crime-type-columns {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 2rem;
    max-height: 280px;
    overflow-y: auto;
    padding: 0.5rem 0;
}

.crime-column {
    display: flex;
    flex-direction: column;
    gap: 0.15rem;
}

.column-header {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.4rem 0.5rem;
    border-bottom: 2px solid #e5e7eb;
    margin-bottom: 0.4rem;
    font-weight: 600;
    font-size: 0.8rem;
    text-transform: uppercase;
    letter-spacing: 0.3px;
    color: #374151;
    background: #f9fafb;
    border-radius: 4px 4px 0 0;
}

.column-header.violent {
    border-bottom-color: #fca5a5;
    color: #dc2626;
    background: #fef2f2;
}

.column-header.property {
    border-bottom-color: #fcd34d;
    color: #d97706;
    background: #fefce8;
}

.column-header.other {
    border-bottom-color: #d1d5db;
    color: #4b5563;
    background: #f9fafb;
}

.column-header input[type="checkbox"] {
    width: 15px;
    height: 15px;
    accent-color: #4b5563;
    cursor: pointer;
}

.crime-checkbox {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.35rem 0.5rem;
    cursor: pointer;
    font-size: 0.775rem;
    color: #4b5563;
    border-radius: 4px;
    transition: background 0.1s ease;
}

.crime-checkbox:hover {
    background: #f3f4f6;
}

.crime-checkbox input[type="checkbox"] {
    width: 13px;
    height: 13px;
    accent-color: #4b5563;
    cursor: pointer;
}

.crime-checkbox span {
    flex: 1;
    line-height: 1.3;
}

/* Panel actions */
.panel-actions {
    display: flex;
    align-items: center;
    gap: 0.6rem;
    padding: 0.75rem 0 0.5rem;
    margin-top: 0.75rem;
    border-top: 1px solid #e5e7eb;
}

.selection-count {
    font-size: 0.775rem;
    color: #6b7280;
    margin-right: auto;
    font-weight: 500;
}

.panel-btn {
    padding: 0.35rem 0.7rem;
    background: white;
    border: 1px solid #d1d5db;
    border-radius: 6px;
    color: #6b7280;
    font-size: 0.75rem;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.12s ease;
}

.panel-btn:hover {
    background: #f9fafb;
    border-color: #9ca3af;
    color: #4b5563;
}

.panel-btn.primary {
    background: #4b5563;
    border-color: #4b5563;
    color: white;
}

.panel-btn.primary:hover {
    background: #374151;
    border-color: #374151;
}

/* Custom date range row */
.custom-date-row {
    display: flex;
    align-items: center;
    justify-content: flex-start;
    gap: 1rem;
    padding: 1rem 1.5rem;
    background: #eff6ff;
    border: 2px solid #3b82f6;
    border-radius: 8px;
    margin-top: 1rem;
    animation: slideDown 0.2s ease-out;
}

.custom-date-row label {
    font-size: 0.75rem;
    font-weight: 700;
    color: #1e40af;
    text-transform: uppercase;
    letter-spacing: 0.5px;
}

.date-input {
    padding: 0.5rem 0.75rem;
    background: white;
    border: 2px solid #e5e7eb;
    border-radius: 6px;
    color: #374151;
    font-size: 0.875rem;
    font-weight: 500;
    transition: all 0.2s ease;
}

.date-input:hover {
    border-color: #d1d5db;
}

.date-input:focus {
    outline: none;
    border-color: #3b82f6;
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

/* Smart search dropdown (used in filter bar) */
.table-search-wrapper .smart-search-dropdown {
    position: absolute;
    top: 100%;
    left: 0;
    right: 0;
    background: white;
    border-radius: 0 0 8px 8px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
    z-index: 100;
    max-height: 280px;
    overflow-y: auto;
}

/* Map search dropdown - dark theme */
.search-box {
    position: relative;
}

.map-search-dropdown {
    position: absolute;
    top: calc(100% + 4px);
    left: 0;
    right: 0;
    background: white;
    border: 1px solid #d1d5db;
    border-radius: 6px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
    z-index: 100;
    max-height: 250px;
    overflow-y: auto;
}

.map-search-item {
    display: flex;
    align-items: center;
    gap: 0.75rem;
    padding: 0.65rem 0.875rem;
    cursor: pointer;
    border-bottom: 1px solid #f3f4f6;
    transition: background 0.15s ease;
}

.map-search-item:last-child {
    border-bottom: none;
}

.map-search-item:hover,
.map-search-item.selected {
    background: #f3f4f6;
}

.map-suggestion-category {
    font-size: 0.65rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
    white-space: nowrap;
    flex-shrink: 0;
    min-width: 95px;
    text-align: center;
}

.map-search-item.type-neighborhood .map-suggestion-category {
    background: #dcfce7;
    color: #166534;
}

.map-search-item.type-street .map-suggestion-category {
    background: #dbeafe;
    color: #1e40af;
}

.map-suggestion-text {
    flex: 1;
    font-size: 0.875rem;
    color: #374151;
    line-height: 1.4;
}

.map-suggestion-text :deep(mark) {
    background: #fef08a;
    color: #374151;
    padding: 0 2px;
    border-radius: 2px;
    font-weight: 600;
}

/* Geocoding status message - prominent notification */
.geocoding-status {
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    padding: 0.75rem 1.25rem;
    border-radius: 8px;
    margin-bottom: 1rem;
    font-weight: 600;
    font-size: 0.95rem;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    box-shadow: 0 4px 15px rgba(102, 126, 234, 0.4);
    animation: slideIn 0.3s ease-out;
}

@keyframes slideIn {
    from {
        opacity: 0;
        transform: translateY(-10px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

/* X4: Table search input */
.table-search-container {
    position: relative;
    margin-bottom: 1rem;
}

.table-search-input {
    width: 100%;
    padding: 0.75rem;
    border: 1px solid #ccc;
    border-radius: 4px;
    font-size: 0.95rem;
    font-weight: 500;
}

.table-search-clear {
    position: absolute;
    right: 0.75rem;
    top: 50%;
    transform: translateY(-50%);
    background: #ef4444;
    color: white;
    border: none;
    border-radius: 4px;
    padding: 0.4rem 0.8rem;
    cursor: pointer;
    font-size: 1rem;
}

.table-search-clear:hover {
    background: #dc2626;
}

/* table wrapper for horizontal scroll on small screens */
.table-wrapper {
    overflow-x: auto;
}

/* crime table */
.crime-table {
    width: 100%;
    border-collapse: collapse;
    font-size: 0.9rem;
}
.crime-table th,
.crime-table td {
    padding: 0.75rem;
    text-align: left;
    border-bottom: 1px solid #ddd;
}
.crime-table th {
    background: #3b82f6;
    color: white;
    font-weight: 600;
    position: sticky;
    top: 0;
}

/* X3: Sortable column headers */
.crime-table th.sortable {
    cursor: pointer;
    user-select: none;
}

.crime-table th.sortable:hover {
    background: #2563eb;
}

.crime-table tbody tr:hover {
    background: #f0f7ff;
}
.clear-btn {
    font-size: 0.75rem;
    padding: 0.1rem 0.4rem;
    margin-left: 0.5rem;
    background: #e5e5e5;
    border: 1px solid #ccc;
    border-radius: 3px;
    cursor: pointer;
}
.clear-btn:hover {
    background: #ddd;
}
/* color coded rows */
.crime-table tbody tr.violent {
    background-color: #ffcdd2;
}
.crime-table tbody tr.property {
    background-color: #fff9c4;
}
.crime-table tbody tr.other {
    background-color: #e0e0e0;
}

/* keep hover effect */
.crime-table tbody tr.violent:hover { background-color: #ef9a9a; }
.crime-table tbody tr.property:hover { background-color: #fff176; }
.crime-table tbody tr.other:hover { background-color: #bdbdbd; }

/* legend */
.legend {
    display: flex;
    gap: 1.5rem;
    margin-bottom: 0.75rem;
}
.legend-item {
    display: flex;
    align-items: center;
    gap: 0.4rem;
    font-size: 0.85rem;
    color: #555;
}
.dot {
    width: 14px;
    height: 14px;
    border-radius: 3px;
}
.dot.violent { background: #ffcdd2; border: 1px solid #e57373; }
.dot.property { background: #fff9c4; border: 1px solid #ffd54f; }
.dot.other { background: #e0e0e0; border: 1px solid #bdbdbd; }

/* marker button - show on map */
.marker-btn {
    background: #e3f2fd;
    border: 2px solid #2196f3;
    cursor: pointer;
    font-size: 1rem;
    padding: 0.3rem;
    border-radius: 4px;
    transition: all 0.2s ease;
    width: 36px;
    height: 36px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    box-sizing: border-box;
}
.marker-btn:hover:not(:disabled) {
    background: #bbdefb;
    transform: scale(1.1);
}
.marker-btn:disabled {
    cursor: wait;
    opacity: 0.8;
}
.marker-btn.active {
    background: #4caf50;
    border-color: #388e3c;
    color: white;
}
.marker-btn.active:hover:not(:disabled) {
    background: #43a047;
}
.marker-btn.loading {
    background: #f1f8e9;
    border-color: #81c784;
}

/* Circular progress ring */
.progress-ring {
    display: block;
    flex-shrink: 0;
}

.progress-ring-bg {
    opacity: 1;
}

.progress-ring-fill {
    transition: stroke-dashoffset 0.1s ease-out;
}

.progress-text {
    font-family: system-ui, -apple-system, sans-serif;
}

/* Subtle pulse on the loading button */
.marker-btn.loading {
    animation: softPulse 2s ease-in-out infinite;
}

@keyframes softPulse {
    0%, 100% { box-shadow: 0 0 0 0 rgba(76, 175, 80, 0.4); }
    50% { box-shadow: 0 0 0 4px rgba(76, 175, 80, 0.2); }
}

/* delete button */
.delete-btn {
    background: none;
    border: none;
    cursor: pointer;
    font-size: 1.1rem;
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
}
.delete-btn:hover {
    background: #ffcdd2;
}

/* Smart search dropdown */
.table-search-container {
    position: relative;
}

.smart-search-dropdown {
    position: absolute;
    top: 100%;
    left: 0;
    right: 0;
    background: white;
    border: 2px solid #3b82f6;
    border-top: none;
    border-radius: 0 0 8px 8px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
    z-index: 100;
    max-height: 320px;
    overflow-y: auto;
}

.smart-search-item {
    display: flex;
    align-items: center;
    padding: 0.75rem 1rem;
    cursor: pointer;
    border-bottom: 1px solid #f0f0f0;
    transition: background 0.15s ease;
}

.smart-search-item:last-child {
    border-bottom: none;
}

.smart-search-item:hover,
.smart-search-item.selected {
    background: #f0f7ff;
}

.smart-search-item.selected {
    background: #e0efff;
}

.suggestion-category {
    font-size: 0.7rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    padding: 0.2rem 0.5rem;
    border-radius: 4px;
    margin-right: 0.75rem;
    white-space: nowrap;
    min-width: 70px;
    text-align: center;
}

/* Category colors */
.smart-search-item.type-case .suggestion-category {
    background: #dbeafe;
    color: #1e40af;
}

.smart-search-item.type-address .suggestion-category {
    background: #dcfce7;
    color: #166534;
}

.smart-search-item.type-crime .suggestion-category {
    background: #fef3c7;
    color: #92400e;
}

.suggestion-text {
    flex: 1;
    font-size: 0.95rem;
    color: #374151;
}

.suggestion-text :deep(mark) {
    background: #fef08a;
    color: inherit;
    padding: 0 2px;
    border-radius: 2px;
}

/* Update input styling when dropdown is open */
.table-search-input:focus {
    border-color: #3b82f6;
    outline: none;
}

.table-search-container:has(.smart-search-dropdown) .table-search-input {
    border-radius: 8px 8px 0 0;
    border-color: #3b82f6;
}

/* ========== ANALYTICS PANEL STYLES ========== */
.analytics-section {
    padding: 0 1rem;
    max-width: 1400px;
    margin: 0 auto;
}

.analytics-panel {
    background: linear-gradient(135deg, #f8fafc 0%, #f1f5f9 100%);
    border: 1px solid #e2e8f0;
    border-radius: 12px;
    padding: 1.25rem;
    margin-bottom: 1rem;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
    color: #334155;
}

.analytics-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 0.75rem;
    flex-wrap: wrap;
    gap: 0.5rem;
}

.analytics-title {
    margin: 0;
    font-size: 1.1rem;
    font-weight: 600;
    color: #1e293b;
}

.analytics-area-name {
    color: #2563eb;
    font-weight: 700;
}

.analytics-actions {
    display: flex;
    gap: 0.5rem;
}

.show-all-btn {
    padding: 0.4rem 0.75rem;
    background: #fbbf24;
    color: #1e293b;
    border: none;
    border-radius: 6px;
    font-size: 0.8rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.15s ease;
}

.show-all-btn:hover {
    background: #f59e0b;
    transform: translateY(-1px);
}

.collapse-btn {
    padding: 0.4rem 0.75rem;
    background: #e2e8f0;
    color: #64748b;
    border: 1px solid #cbd5e1;
    border-radius: 6px;
    font-size: 0.8rem;
    cursor: pointer;
    transition: all 0.15s ease;
}

.collapse-btn:hover {
    background: #cbd5e1;
    color: #334155;
}

.analytics-summary {
    margin: 0 0 1rem 0;
    font-size: 0.95rem;
    color: #64748b;
}

.analytics-summary strong {
    color: #2563eb;
    font-size: 1.1rem;
}

.charts-row {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
    gap: 1.5rem;
    margin-bottom: 1rem;
}

.chart-container {
    background: rgba(255, 255, 255, 0.05);
    border-radius: 10px;
    padding: 1rem;
}

.chart-title {
    margin: 0 0 0.75rem 0;
    font-size: 0.9rem;
    font-weight: 600;
    color: #475569;
    text-transform: uppercase;
    letter-spacing: 0.5px;
}

.bar-chart {
    display: flex;
    flex-direction: column;
    gap: 0.35rem;
}

.bar-row {
    display: flex;
    align-items: center;
    gap: 0.5rem;
}

.bar-label {
    width: 40px;
    font-size: 0.75rem;
    color: #64748b;
    text-align: right;
    flex-shrink: 0;
}

.bar-track {
    flex: 1;
    height: 18px;
    background: #e2e8f0;
    border-radius: 4px;
    overflow: hidden;
}

.bar-fill {
    height: 100%;
    border-radius: 4px;
    transition: width 0.3s ease;
    min-width: 2px;
}

.bar-fill.hour-bar {
    background: linear-gradient(90deg, #3b82f6 0%, #60a5fa 100%);
}

.bar-fill.day-bar {
    background: linear-gradient(90deg, #10b981 0%, #34d399 100%);
}

.bar-fill.day-bar.weekend {
    background: linear-gradient(90deg, #8b5cf6 0%, #a78bfa 100%);
}

.bar-fill.peak {
    background: linear-gradient(90deg, #f59e0b 0%, #fbbf24 100%) !important;
    box-shadow: 0 0 8px rgba(251, 191, 36, 0.5);
}

/* Stacked bar segments for crime type breakdown */
.bar-track.stacked {
    display: flex;
    overflow: hidden;
}

.bar-segment {
    height: 100%;
    transition: width 0.3s ease;
    min-width: 0;
}

.bar-segment.violent {
    background: #ef4444;
}

.bar-segment.property {
    background: #eab308;
}

.bar-segment.other {
    background: #6b7280;
}

/* Clickable bar rows */
.bar-row.clickable-bar {
    cursor: pointer;
    transition: all 0.15s ease;
    border-radius: 4px;
    padding: 2px 4px;
    margin: 0 -4px;
}

.bar-row.clickable-bar:hover {
    background: rgba(59, 130, 246, 0.08);
}

.bar-row.clickable-bar:hover .bar-label {
    color: #60a5fa;
}

/* Selected bar state */
.bar-row.selected-bar {
    background: rgba(59, 130, 246, 0.2);
    border-left: 3px solid #3b82f6;
    padding-left: 6px;
}

.bar-row.selected-bar .bar-label {
    color: #60a5fa;
    font-weight: 600;
}

/* Filter badge in chart title */
.filter-badge {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    background: #dbeafe;
    color: #1d4ed8;
    font-size: 0.7rem;
    font-weight: 500;
    padding: 2px 8px;
    border-radius: 10px;
    margin-left: 8px;
}

.clear-filter-x {
    background: none;
    border: none;
    color: #1d4ed8;
    font-size: 0.85rem;
    cursor: pointer;
    padding: 0;
    margin-left: 2px;
    line-height: 1;
    opacity: 0.7;
}

.clear-filter-x:hover {
    opacity: 1;
    color: #1e40af;
}

/* Weekend row indicator */
.bar-row.weekend-row .bar-label {
    color: #a78bfa;
}

/* Compact crime type legend */
.crime-legend {
    display: flex;
    justify-content: center;
    gap: 1rem;
    margin-bottom: 0.75rem;
    padding: 0.4rem 0.75rem;
    background: rgba(0, 0, 0, 0.03);
    border-radius: 6px;
}

.crime-legend .legend-item {
    display: flex;
    align-items: center;
    gap: 0.3rem;
    font-size: 0.7rem;
    color: #64748b;
    text-transform: uppercase;
    letter-spacing: 0.3px;
}

.legend-dot {
    width: 8px;
    height: 8px;
    border-radius: 2px;
}

.legend-dot.violent {
    background: #ef4444;
}

.legend-dot.property {
    background: #eab308;
}

.legend-dot.other {
    background: #6b7280;
}

.bar-count {
    width: 35px;
    font-size: 0.8rem;
    font-weight: 600;
    color: #334155;
    text-align: right;
    flex-shrink: 0;
}

.chart-insight {
    margin: 0.75rem 0 0 0;
    font-size: 0.85rem;
    color: #64748b;
    display: flex;
    align-items: center;
    gap: 0.35rem;
}

.insight-icon {
    font-size: 1rem;
}

.chart-insight strong {
    color: #2563eb;
}

.insight-count,
.insight-percent {
    color: #64748b;
    font-size: 0.8rem;
}

.top-areas {
    background: rgba(255, 255, 255, 0.05);
    border-radius: 10px;
    padding: 1rem;
}

.hotspot-list {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
}

.hotspot-chip {
    display: inline-flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.5rem 0.75rem;
    background: rgba(59, 130, 246, 0.2);
    border: 1px solid rgba(59, 130, 246, 0.3);
    border-radius: 20px;
    cursor: pointer;
    transition: all 0.15s ease;
}

.hotspot-chip:hover {
    background: rgba(59, 130, 246, 0.3);
    border-color: rgba(59, 130, 246, 0.5);
    transform: translateY(-2px);
}

.hotspot-name {
    font-size: 0.85rem;
    color: #1e40af;
    font-weight: 500;
}

.hotspot-count {
    font-size: 0.75rem;
    font-weight: 700;
    color: #0369a1;
    background: #e0f2fe;
    padding: 0.15rem 0.4rem;
    border-radius: 10px;
}

.analytics-expand {
    width: 100%;
    padding: 0.75rem;
    background: linear-gradient(135deg, #f8fafc 0%, #f1f5f9 100%);
    color: #64748b;
    border: 1px solid #e2e8f0;
    border-radius: 8px;
    font-size: 0.9rem;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.15s ease;
    margin-bottom: 1rem;
}

.analytics-expand:hover {
    background: linear-gradient(135deg, #f1f5f9 0%, #e2e8f0 100%);
    color: #e2e8f0;
}

/* Responsive adjustments for analytics */
@media (max-width: 640px) {
    .charts-row {
        grid-template-columns: 1fr;
    }
    
    .analytics-header {
        flex-direction: column;
        align-items: flex-start;
    }
    
    .analytics-actions {
        width: 100%;
    }
    
    .analytics-actions button {
        flex: 1;
    }
}
</style>
