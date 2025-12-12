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
    selected_neighborhood: '' // single neighborhood id or empty for all
});

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
        });
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
    return incidents.value.filter(inc => {
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
        
        return true;
    });
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
        console.log('deleted case:', case_number);
    })
    .catch(err => {
        console.log('delete error:', err);
        alert('Failed to delete incident: ' + err.message);
    });
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
        
        <!-- button to show form - positioned over map -->
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
    </div>
    
    <!-- results count -->
    <p class="results-count">Showing {{ filteredIncidents.length }} of {{ incidents.length }} incidents</p>
    
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
                    <th>Case #</th>
                    <th>Date</th>
                    <th>Time</th>
                    <th>Incident</th>
                    <th>Block</th>
                    <th>Neighborhood</th>
                    <th>Delete</th>
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
                    <td><button class="delete-btn" @click="deleteIncident(inc.case_number)">🗑️</button></td>
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

.results-count {
    color: #666;
    margin-bottom: 0.5rem;
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
