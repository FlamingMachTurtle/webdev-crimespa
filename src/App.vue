<script setup>
import { reactive, ref, onMounted, computed } from 'vue'


let crime_url = ref('');
let dialog_err = ref(false);

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

// C1 + C2 + C3: address search state
let addressSearch = reactive({
    query: '',
    searching: false,
    error: '',
    lat: null,
    lng: null,
    marker: null  // store reference to search marker so we can remove it
});

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

    // Update when map is panned/zoomed (core requirement)
    map.leaflet.on('moveend', () => {
        const center = map.leaflet.getCenter();
        map.center.lat = center.lat;
        map.center.lng = center.lng;
        // Store current center address for display (but don't overwrite if user is typing)
        map.center.address = `${center.lat.toFixed(6)}, ${center.lng.toFixed(6)}`;
        // Update visible neighborhoods based on map bounds
        updateVisibleNeighborhoods();
    });

    // Get boundaries for St. Paul neighborhoods
    let district_boundary = new L.geoJson();
    district_boundary.addTo(map.leaflet);
    fetch('data/StPaulDistrictCouncil.geojson')
    .then((response) => {
        return response.json();
    })
    .then((result) => {
        result.features.forEach((value) => {
            district_boundary.addData(value);
        });
    })
    .catch((error) => {
        console.log('Error:', error);
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
            `https://nominatim.openstreetmap.org/search?q=${encodeURIComponent(addressSearch.query)}&format=json&limit=1`
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

    } catch (error) {
        console.error('Search error:', error);
        addressSearch.error = 'Failed to geocode address: ' + error.message;
    } finally {
        addressSearch.searching = false;
    }
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
}

// submit new incident to api
function submitIncident() {
    // check all fields filled
    if (!newIncident.case_number || !newIncident.date || !newIncident.time ||
        !newIncident.code || !newIncident.incident || !newIncident.police_grid ||
        !newIncident.neighborhood_number || !newIncident.block) {
        form_error.value = 'fill out all fields';
        return;
    }
    
    form_error.value = '';
    form_success.value = false;
    let api = crime_url.value.replace(/\/+$/, '');
    
    // send to api
    fetch(api + '/new-incident', {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
            case_number: newIncident.case_number,
            date: newIncident.date,
            time: newIncident.time,
            code: Number(newIncident.code),
            incident: newIncident.incident,
            police_grid: Number(newIncident.police_grid),
            neighborhood_number: Number(newIncident.neighborhood_number),
            block: newIncident.block
        })
    })
    .then(res => res.json())
    .then(data => {
        console.log('incident added:', data);
        form_success.value = true;
        // clear form
        newIncident.case_number = '';
        newIncident.date = '';
        newIncident.time = '';
        newIncident.code = '';
        newIncident.incident = '';
        newIncident.police_grid = '';
        newIncident.neighborhood_number = '';
        newIncident.block = '';
        // update crime counts
        updateNeighborhoodCrimeCounts();
        addNeighborhoodMarkers();
    })
    .catch(err => {
        form_error.value = 'failed to add incident';
        console.log('error:', err);
    });
}

// show or hide the incident form
function toggleForm() {
    form_visible.value = !form_visible.value;
}

// Function called when user presses 'OK' on dialog box
function closeDialog() {
    let dialog = document.getElementById('rest-dialog');
    let url_input = document.getElementById('dialog-url');
    if (crime_url.value !== '' && url_input.checkValidity()) {
        dialog_err.value = false;
        dialog.close();
        initializeCrimes();
    }
    else {
        dialog_err.value = true;
    }
}

// computed property - filters incidents based on current filter selections
const filteredIncidents = computed(() => {
    // First apply filters
    let filtered = incidents.value.filter(inc => {
        // Core requirement: filter by visible map area
        if (filters.limit_to_visible && visibleNeighborhoods.value.length > 0) {
            if (!visibleNeighborhoods.value.includes(inc.neighborhood_number)) return false;
        }
        
        // date filter
        if (filters.start_date && inc.date < filters.start_date) return false;
        if (filters.end_date && inc.date > filters.end_date) return false;
        
        // crime type filter (if any selected)
        if (filters.selected_codes.length > 0) {
            if (!filters.selected_codes.includes(inc.code)) return false;
        }
        
        // neighborhood filter
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

</script>

<template>
    <dialog id="rest-dialog" open>
        <h1 class="dialog-header">St. Paul Crime REST API</h1>
        <label class="dialog-label">URL: </label>
        <input id="dialog-url" class="dialog-input" type="url" v-model="crime_url" placeholder="http://localhost:8000" />
        <p class="dialog-error" v-if="dialog_err">Error: must enter valid URL</p>
        <br/>
        <button class="button" type="button" @click="closeDialog">OK</button>
    </dialog>
    
    <!-- main layout - map + button -->
    <div class="main-layout">
        <div id="leafletmap"></div>
        
        <!-- C1: Address search input UI -->
        <div class="search-container">
            <div class="search-box">
                <input 
                    v-model="addressSearch.query" 
                    type="text" 
                    placeholder="Search for an address in St. Paul..."
                    @keyup.enter="searchAddress"
                    class="search-input"
                />
                <button 
                    class="search-button" 
                    @click="searchAddress"
                    :disabled="addressSearch.searching"
                >
                    {{ addressSearch.searching ? 'Searching...' : 'Search' }}
                </button>
                <button 
                    class="clear-search-button" 
                    @click="clearSearch"
                    :disabled="!addressSearch.marker"
                    title="Remove search marker"
                >
                    ✕
                </button>
            </div>
            <p v-if="addressSearch.error" class="search-error">{{ addressSearch.error }}</p>
            <p class="current-center">Map center: {{ map.center.address || `${map.center.lat.toFixed(4)}, ${map.center.lng.toFixed(4)}` }}</p>
        </div>
        
        <!-- button to show form - positioned over map in top right -->
        <button class="report-button" @click="toggleForm">
            {{ form_visible ? 'Close Form' : 'Report New Incident' }}
        </button>
    </div>

    <!-- filters and crime table section -->
<div class="table-section">
    <h2>Crime Incidents</h2>
    
    <!-- filter controls -->
    <div class="filters">
        <div class="filter-group">
            <label>Start Date</label>
            <input type="date" v-model="filters.start_date" />
        </div>
        <div class="filter-group">
            <label>End Date</label>
            <input type="date" v-model="filters.end_date" />
        </div>
        <div class="filter-group">
            <label>Max Incidents</label>
            <select v-model="filters.max_incidents">
                <option :value="50">50</option>
                <option :value="100">100</option>
                <option :value="250">250</option>
                <option :value="500">500</option>
                <option :value="1000">1,000</option>
            </select>
        </div>
        <div class="filter-group">
            <label>Crime Type <button class="clear-btn" @click="filters.selected_codes = []">Clear</button></label>
            <select v-model="filters.selected_codes" multiple>
                <option v-for="c in codes" :key="c.code" :value="c.code">
                    {{ c.type }}
                </option>
            </select>
        </div>
        <div class="filter-group">
            <label>Neighborhood</label>
            <select v-model="filters.selected_neighborhood">
                <option value="">All Neighborhoods</option>
                <option v-for="n in neighborhoods" :key="n.id" :value="n.id">
                    {{ n.name }}
                </option>
            </select>
        </div>
        <div class="filter-group filter-toggle">
            <label>
                <input type="checkbox" v-model="filters.limit_to_visible" />
                Only show crimes in visible map area
            </label>
        </div>
    </div>
    
    <!-- results count -->
    <p class="results-count">Showing {{ filteredIncidents.length }} of {{ incidents.length }} incidents</p>
    
    <!-- Geocoding status message -->
    <div v-if="geocodingStatus" class="geocoding-status">
        {{ geocodingStatus }}
    </div>
    
    <!-- X4: Table search input -->
    <div class="table-search-container">
        <input 
            v-model="tableSearch" 
            type="text" 
            placeholder="Search crimes (case #, date, time, incident, block, neighborhood)..."
            class="table-search-input"
        />
        <button 
            v-if="tableSearch" 
            @click="tableSearch = ''"
            class="table-search-clear"
            title="Clear search"
        >
            ✕
        </button>
    </div>
    
    <!-- color legend -->
<div class="legend">
    <span class="legend-item"><span class="dot violent"></span> Violent</span>
    <span class="legend-item"><span class="dot property"></span> Property</span>
    <span class="legend-item"><span class="dot other"></span> Other</span>
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
                <tr v-for="inc in filteredIncidents" :key="inc.case_number" :class="getCrimeCategory(inc.code)">
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
            <h3>Add New Incident</h3>
            <button class="close-form" @click="toggleForm">×</button>
        </div>
        
        <label>Case Number</label>
        <input v-model="newIncident.case_number" placeholder="24-123456" />
        
        <label>Date</label>
        <input v-model="newIncident.date" type="date" />
        
        <label>Time</label>
        <input v-model="newIncident.time" type="time" step="1" />
        
        <label>Crime Type</label>
        <select v-model="newIncident.code">
            <option value="">-- pick one --</option>
            <option v-for="c in codes" :key="c.code" :value="c.code">
                {{ c.type }}
            </option>
        </select>
        
        <label>Incident Description</label>
        <input v-model="newIncident.incident" placeholder="what happened" />
        
        <label>Police Grid</label>
        <input v-model="newIncident.police_grid" type="number" placeholder="87" />
        
        <label>Neighborhood</label>
        <select v-model="newIncident.neighborhood_number">
            <option value="">-- pick one --</option>
            <option v-for="n in neighborhoods" :key="n.id" :value="n.id">
                {{ n.name }}
            </option>
        </select>
        
        <label>Block Address</label>
        <input v-model="newIncident.block" placeholder="1XX MAIN ST" />
        
        <button class="button" @click="submitIncident">Submit</button>
        
        <p v-if="form_error" class="form-error">{{ form_error }}</p>
        <p v-if="form_success" class="form-success">incident added!</p>
    </div>
</template>

<style scoped>
/* dialog for api url */
#rest-dialog {
    width: 20rem;
    margin-top: 1rem;
    z-index: 1000;
}
.dialog-header {
    font-size: 1.2rem;
    font-weight: bold;
}
.dialog-label {
    font-size: 1rem;
}
.dialog-input {
    font-size: 1rem;
    width: 100%;
}
.dialog-error {
    font-size: 1rem;
    color: #D32323;
}

/* main layout - full screen with padding */
.main-layout {
    position: relative;
    padding: 1rem;
    height: 60vh;
}

/* map fills the space */
#leafletmap {
    width: 100%;
    height: 100%;
    border-radius: 8px;
}

/* C1: Address search container */
.search-container {
    position: absolute;
    bottom: 1.5rem;
    left: 1.5rem;
    z-index: 500;
    background: white;
    padding: 1rem;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.2);
    min-width: 300px;
}

.search-box {
    display: flex;
    gap: 0.5rem;
}

.search-input {
    flex: 1;
    padding: 0.75rem;
    border: 1px solid #ccc;
    border-radius: 4px;
    font-size: 0.95rem;
}

.search-button {
    padding: 0.75rem 1.5rem;
    background: #3b82f6;
    color: white;
    border: none;
    border-radius: 4px;
    font-weight: 600;
    cursor: pointer;
    white-space: nowrap;
}

.search-button:hover:not(:disabled) {
    background: #2563eb;
}

.search-button:disabled {
    background: #9ca3af;
    cursor: not-allowed;
}

.clear-search-button {
    padding: 0.75rem 0.75rem;
    background: #ef4444;
    color: white;
    border: none;
    border-radius: 4px;
    font-weight: 600;
    cursor: pointer;
    font-size: 1.1rem;
}

.clear-search-button:hover:not(:disabled) {
    background: #dc2626;
}

.clear-search-button:disabled {
    background: #d1d5db;
    cursor: not-allowed;
}

.search-error {
    color: #d32323;
    font-size: 0.85rem;
    margin-top: 0.5rem;
    margin-bottom: 0;
}

.current-center {
    color: #666;
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
    height: calc(100vh - 4rem); /* leave room at bottom for leaflet watermark */
    background: white;
    padding: 2rem;
    overflow-y: auto;
    box-shadow: -2px 0 10px rgba(0,0,0,0.2);
    transition: right 0.3s ease; /* smooth slide */
    z-index: 600;
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

/* form fields */
.incident-form label {
    display: block;
    margin-top: 1rem;
    font-weight: bold;
    color: #555;
}
.incident-form input, 
.incident-form select {
    width: 100%;
    padding: 0.5rem;
    margin-top: 0.25rem;
    border: 1px solid #ccc;
    border-radius: 4px;
}

/* submit button at bottom with proper spacing */
.incident-form button {
    width: 100%;
    margin-top: 1.5rem;
    padding: 0.75rem;
    background: #3b82f6;
    color: white;
    border: none;
    border-radius: 6px;
    font-weight: 600;
    cursor: pointer;
}
.incident-form button:hover {
    background: #2563eb;
}

/* error and success messages */
.form-error { 
    color: #D32323; 
    margin-top: 0.5rem;
}
.form-success { 
    color: #2E7D32;
    margin-top: 0.5rem;
}

/* table section */
.table-section {
    padding: 1rem;
    max-width: 1400px;
    margin: 0 auto;
}
.table-section h2 {
    margin-bottom: 1rem;
    color: #333;
}

/* filter controls */
.filters {
    display: flex;
    flex-wrap: wrap;
    gap: 1rem;
    margin-bottom: 1rem;
    padding: 1rem;
    background: #f5f5f5;
    border-radius: 8px;
}
.filter-group {
    display: flex;
    flex-direction: column;
    min-width: 150px;
}
.filter-group label {
    font-weight: 600;
    margin-bottom: 0.25rem;
    color: #555;
}
.filter-group input,
.filter-group select {
    padding: 0.5rem;
    border: 1px solid #ccc;
    border-radius: 4px;
}
.filter-group select[multiple] {
    height: 100px;
}
.filter-toggle {
    display: flex;
    align-items: center;
}
.filter-toggle label {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    cursor: pointer;
    white-space: nowrap;
}
.filter-toggle input[type="checkbox"] {
    width: auto;
    margin: 0;
}

.results-count {
    color: #666;
    margin-bottom: 0.5rem;
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
</style>
