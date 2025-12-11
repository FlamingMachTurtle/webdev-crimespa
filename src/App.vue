<script setup>
import { reactive, ref, onMounted } from 'vue'

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
        method: 'POST',
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
    height: calc(100vh - 2rem);
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
</style>
