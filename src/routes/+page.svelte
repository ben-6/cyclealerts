<svelte:head>
    <link href='https://api.mapbox.com/mapbox-gl-js/v2.14.1/mapbox-gl.css' rel='stylesheet' />
</svelte:head>

<script>
    import { onMount } from 'svelte';
    import { initializeApp, getApps } from 'firebase/app' 
    import { getDatabase, ref, onValue, query, orderByChild, update, push, runTransaction, serverTimestamp, remove, set, off, child } from 'firebase/database'
    import { GoogleAuthProvider, getAuth, signInWithPopup, onAuthStateChanged, signOut } from 'firebase/auth';
    import mapboxgl from 'mapbox-gl';
    import '../global.css';


    const firebaseConfig = {
        apiKey: import.meta.env.VITE_FIREBASE_API_KEY,
        authDomain: import.meta.env.VITE_FIREBASE_AUTH_DOMAIN,
        databaseURL: import.meta.env.VITE_FIREBASE_DATABASE_URL,
        projectId: import.meta.env.VITE_FIREBASE_PROJECT_ID,
        storageBucket: import.meta.env.VITE_FIREBASE_STORAGE_BUCKET,
        messagingSenderId: import.meta.env.VITE_FIREBASE_MESSAGING_SENDER_ID,
        appId: import.meta.env.VITE_FIREBASE_APP_ID,
        measurementId: import.meta.env.VITE_FIREBASE_MEASUREMENT_ID
    };

    const app = getApps().length === 0 ? initializeApp(firebaseConfig) : getApps()[0];
    const database = getDatabase(app);

    const auth = getAuth(app);
    let user = undefined;

    const signInWithGoogle = async () => {
        const provider = new GoogleAuthProvider();
        try {
            const result = await signInWithPopup(auth, provider);   
            user = result.user; 
        } catch (error) {
            console.error("error signing in with Google:", error);
        }
    };

    const signOutUser = async () => {
        try {
            await signOut(auth);
            user = undefined;
        } catch (error) {
            console.error("error signing out:", error);
        }
    };
    onAuthStateChanged(auth, (currentUser) => {
        user = currentUser;
        if (user) {
            onValue(ref(database, `users/${user.uid}/contributions`), (snapshot) => {
                contributions = snapshot.val() || 0;
            });
        }
    });


    let map;
    let mapMarkers = {};
    let contributions = 0;
    let selectedHazard = null;
    let tempMarker = null;
    let currentWindowOpen = null;

    let newMarkerCoords = null;
    let newMarkerName = '';
    let newMarkerDescription = '';
    let newMarkerCorridor = '';

    let commentListener = null;
    let comments = [];

    let amend_input = '';

    let showMenu = true;
    let isMobileSize = false;
    let innerWidth = 0
    let innerHeight = 0
    
    let loadingCurrentLocation = false;
    let pageLoaded = false;


    class Hazard {
        constructor(id, longitude, latitude, name, description, status, confirmed, resolved, report_date, corridor, username) {
            this.id = id;
            this.longitude = longitude;
            this.latitude = latitude;            
            this.name = name;
            this.description = description;
            this.status = status;
            this.confirmed = confirmed;
            this.resolved = resolved;
            this.report_date = report_date;
            this.corridor = corridor;
            this.username = username;
        }

        getCoords() {
            return {lng: this.longitude, lat: this.latitude};
        }
    }

    const hazardTypes = ["bad road design", "bike lane ends", "pothole/bad pavement", "closure", "railroad crossing", "bad visibility", "door zone hazard"];
    const corridorList = ["", "520 trail", "i-90 trail", "alki trail/east marginal way", "burke gilman trail", "downtown bellevue (108th)", "downtown seattle (2nd/bell)", "eastrail/cross kirkland corridor", "east lake sammamish trail", "sammamish river trail", "ship canal trail/westlake cycletrack", "lake washington loop (kirkland)", "lake washington loop (south bellevue/renton)", "lake washington loop (south seattle/central district)"];

    mapboxgl.accessToken = import.meta.env.VITE_MAPBOX_API_KEY;

   
    function handleHazardCreation() {
        // remove temp marker (the blue one)
        if (!newMarkerName) {
            alert("please enter a hazard name");
            return;
        }
        if (!user) {
            alert("please sign in to use this feature.");
            return
        }

        tempMarker.remove();
        tempMarker = null;
        const hazardData = {
            longitude: newMarkerCoords.lng,
            latitude: newMarkerCoords.lat,
            name: newMarkerName,
            description: newMarkerDescription,
            status: 1,
            confirmed: 0,
            resolved: 0,
            report_date: null,
            corridor: newMarkerCorridor,
            username: user.displayName
        };

        const hazardsRef = ref(database, 'hazards');
        push(hazardsRef, hazardData)
            .then((snapshot) => {
                console.log("hazard data saved successfully!");
                contributions ++;
                set(ref(database, `users/${user.uid}/contributions`), contributions);

                const newHazard = new Hazard(snapshot.key, hazardData["longitude"], hazardData["latitude"], hazardData["name"], hazardData["description"], hazardData["status"], hazardData['confirmed'], hazardData['resolved'], hazardData['report_date'], hazardData['corridor'], hazardData['username']);
                selectedHazard = newHazard;
                currentWindowOpen = "selected marker"
                newMarkerCoords = null;
                newMarkerName = "";
                newMarkerDescription = "";
                newMarkerCorridor = "";
            })
            .catch((error) => {
                console.error("error writing hazard data:", error);
            });
        
        
    }

    function createMarker(hazard, color) {
        mapMarkers[hazard.id] = new mapboxgl.Marker({color: color})
            .setLngLat(hazard.getCoords())
            .addTo(map);
        
        mapMarkers[hazard.id].getElement().addEventListener('click', (e) => {
            map.flyTo({
                center: mapMarkers[hazard.id].getLngLat(),
                zoom: 16,
            });

            selectedHazard = hazard;
            currentWindowOpen = "selected marker";
            newMarkerCoords = null;
            if (tempMarker) {
                tempMarker.remove();
            }
            e.stopPropagation();
        });        
    }

    function manageCommentListener(id) {
        const commentsRef = ref(database, 'comments/'+id);

        if (id && !commentListener) {
            commentListener = onValue(commentsRef, (snapshot) => {
                comments = [];
                snapshot.forEach((childSnapshot) => {
                    comments.push([childSnapshot.val().username + " - " + childSnapshot.val().date, childSnapshot.val().comment]);
                });
            });
        } else if (!id && commentListener) {
            off(commentsRef, 'value', commentListener);
            commentListener = null;
        }
    }

    $: if (selectedHazard) {
        manageCommentListener(selectedHazard.id);
    } else {
        manageCommentListener(null);
    }

    function flagHazard() {
        const hazardRef = ref(database, 'hazards/' + selectedHazard.id);

        // Update a single property
        update(hazardRef, { status: 0, confirmed: 0, resolved: 0, report_date: new Date().toISOString().slice(0, 10)})
            .then(() => {
                console.log("hazard data updated successfully!");
            })
            .catch((error) => {
                console.error("error updating hazard data:", error);
            });

        selectedHazard = null;
        currentWindowOpen = null;
        
    }

    function unflagHazard() {
        const hazardRef = ref(database, 'hazards/' + selectedHazard.id);

        // Update a single property
        update(hazardRef, { status: 1, confirmed: 0, resolved: 0})
            .then(() => {
                console.log("hazard data updated successfully!");
            })
            .catch((error) => {
                console.error("error updating hazard data:", error);
            });

        selectedHazard = null;
        currentWindowOpen = null;
        
    }

    function submitAmendment() {
        const hazardRef = ref(database, 'comments/' + selectedHazard.id);
        
        push(hazardRef, {"username": user.displayName, "date": new Date().toISOString().slice(0, 10), "comment": amend_input, "timestamp": serverTimestamp()})
            .then(() => {
                console.log("key-value pair added successfully!");
            })
            .catch((error) => {
                console.error("error adding key-value pair:", error);
            });

        selectedHazard = null;
        currentWindowOpen = null;

        amend_input = "";
    }

    function confirmedIncrement() {
        const hazardRef = ref(database, 'hazards/' + selectedHazard.id + '/confirmed');

        runTransaction(hazardRef, (currentCount) => {
            return (currentCount || 0) + 1; // Increment, or start at 1 if it doesn't exist
        })
            .then(() => {
                console.log("value incremented successfully!");
            })
            .catch((error) => {
                console.error("error incrementing value:", error);
            });
        selectedHazard = null;
        currentWindowOpen = null;
    }

    function resolvedIncrement() {
        const hazardRef = ref(database, 'hazards/' + selectedHazard.id + '/resolved');

        runTransaction(hazardRef, (currentCount) => {
            return (currentCount || 0) + 1; // Increment, or start at 1 if it doesn't exist
        })
            .then(() => {
                console.log("value incremented successfully!");
            })
            .catch((error) => {
                console.error("error incrementing value:", error);
            });
        selectedHazard = null;
        currentWindowOpen = null;
    }

    function deleteHazard() {
        const hazardRef = ref(database, 'hazards/' + selectedHazard.id);
        remove(hazardRef);
        const commentsRef = ref(database, 'comments/' + selectedHazard.id);
        remove(commentsRef);
        
        mapMarkers[selectedHazard.id].remove();
        delete mapMarkers[selectedHazard.id];
        selectedHazard = null;
        currentWindowOpen = null;
    }

    function displayCorridor(map, fileName, layerId, checkboxId) {
        const filePath = new URL(`../lib/corridor_geojson/${fileName}`, import.meta.url).href;
        fetch(filePath)
            .then(response => response.json())
            .then(data => {
                map.addSource(layerId, {
                    type: 'geojson',
                    data: data
                });

                const layer = {
                    id: layerId,
                    type: 'line',
                    source: layerId,
                    layout: {
                        visibility: 'none'
                    },
                    paint: {
                        'line-color': '#0000ff',
                        'line-width': 2
                    }
                };
                map.addLayer(layer);

                // Checkbox control
                const checkbox = document.getElementById(checkboxId);
                checkbox.addEventListener('change', () => {
                    map.setLayoutProperty(layerId, 'visibility', checkbox.checked ? 'visible' : 'none');
                });
            })
            .catch(error => console.error('Error loading GeoJSON:', error));
        }

    onMount(() => {
        pageLoaded = true;
        map = new mapboxgl.Map({
            container: "map-container",
            style: 'mapbox://styles/mapbox/streets-v12',
            center: [-122.308954, 47.608027],
            zoom: 12
        });

        //const bikeLayer = new google.maps.BicyclingLayer();

        //bikeLayer.setMap(map);
        
        if (navigator.geolocation) {
            navigator.geolocation.getCurrentPosition(
            (position) => {
                const { latitude, longitude } = position.coords; 
                map.flyTo({
                    center: [longitude, latitude],
                    zoom: 12
                });
            });
        }
        
        map.addControl(
            new mapboxgl.GeolocateControl({
                positionOptions: {
                    enableHighAccuracy: true
                },
                trackUserLocation: true,
                showUserHeading: true,
                showAccuracyCircle: true
            }),
            'bottom-right'
        );

        map.addControl(new mapboxgl.NavigationControl(), 'bottom-right');
        
        const hazardsRef = ref(database, 'hazards');
        onValue(hazardsRef, (snapshot) => {
            snapshot.forEach((childSnapshot) => {
                
                if (!Object.keys(mapMarkers).includes(childSnapshot.key)) {
                //     const hazard = new Hazard(childSnapshot.key, markerData.longitude, markerData.latitude, markerData.name, markerData.description, markerData.status, markerData.confirmed, markerData.resolved, markerData.report_date, markerData.corridor, markerData.username)
                //     createMarker(hazard, markerData.status == 1 ? "black" : "red");
                
                
                    const markerRef = child(hazardsRef, childSnapshot.key);
                    onValue(markerRef, (markerSnapshot) => {
                        const markerData = markerSnapshot.val();
                        if (markerData) {
                            if (mapMarkers[markerSnapshot.key]) {
                                mapMarkers[markerSnapshot.key].remove();
                                delete mapMarkers[markerSnapshot.key];
                            } 
                            
                            const hazard = new Hazard(markerSnapshot.key, markerData.longitude, markerData.latitude, markerData.name, markerData.description, markerData.status, markerData.confirmed, markerData.resolved, markerData.report_date, markerData.corridor, markerData.username)
                            createMarker(hazard, markerData.status == 1 ? "black" : "red");
                        }
                        
                    });

                    //off(hazardsRef);
                }

            });
        });

        map.on('click', (e) => {
            openHazardForm(e.lngLat);
        });

        map.on('load', () => {
            displayCorridor(map, "520.geojson", 'layer-520', 'checkbox-520');
            displayCorridor(map, "i-90.geojson", 'layer-90', 'checkbox-90');
            displayCorridor(map, "alki.geojson", 'layer-alki', 'checkbox-alki');
            displayCorridor(map, "bgt.geojson", 'layer-bgt', 'checkbox-bgt');
            displayCorridor(map, "dt-bellevue.geojson", 'layer-dt-bellevue', 'checkbox-dt-bellevue');
            displayCorridor(map, "dt-seattle.geojson", 'layer-dt-seattle', 'checkbox-dt-seattle');
            displayCorridor(map, "ckc.geojson", 'layer-ckc', 'checkbox-ckc');
            displayCorridor(map, "elst.geojson", 'layer-elst', 'checkbox-elst');
            displayCorridor(map, "srt.geojson", 'layer-srt', 'checkbox-srt');
            displayCorridor(map, "westlake-cycle.geojson", 'layer-westlake-cycle', 'checkbox-westlake-cycle');
            displayCorridor(map, "lwl-kirkland.geojson", 'layer-lwl-kirkland', 'checkbox-lwl-kirkland');
            displayCorridor(map, "lwl-renton.geojson", 'layer-lwl-renton', 'checkbox-lwl-renton');
            displayCorridor(map, "lwl-south-seattle.geojson", 'layer-lwl-south-seattle', 'checkbox-lwl-south-seattle');
        });

        
    });

    function openHazardForm(useLocation) {
        if (useLocation != null) {
            newMarkerCoords = useLocation;
            setTempMarker();
        } else {
            if (navigator.geolocation) {
                loadingCurrentLocation = true;
                navigator.geolocation.getCurrentPosition(
                    (position) => {
                        const { longitude, latitude } = position.coords;
                        newMarkerCoords = new mapboxgl.LngLat(longitude, latitude);
                        setTempMarker();
                        loadingCurrentLocation = false;
                    });
            }
        }
        
    }

    function setTempMarker() {
        selectedHazard = null;
        currentWindowOpen = "create hazard";
        if (tempMarker) {
            tempMarker.remove();
        }

        tempMarker = new mapboxgl.Marker({ color: 'blue', draggable: true })
            .setLngLat(newMarkerCoords)
            .addTo(map);

        map.flyTo({
            center: tempMarker.getLngLat(),
            zoom: 16,
        });
    }


    function closeFloatingWindow() {
        currentWindowOpen = null;
        selectedHazard = null;
        newMarkerCoords = null;
        if (tempMarker) {
            tempMarker.remove();
        }
    }

    function toggleMenu() {
        showMenu = !showMenu;
    }

    $: if (innerWidth > 768) {
        isMobileSize = false;
        showMenu = true;
    } else {
        isMobileSize = true;
        showMenu = false;
    }

</script>

<svelte:window bind:innerWidth bind:innerHeight />

{#if isMobileSize}
<div class="top-nav">
    <button class="mobile-menu-button" on:click={toggleMenu}>Menu</button>
    {#if user}
        <p>welcome, {user.displayName}!</p>
        <p>[{contributions}] contributions</p>
        <button class="user-auth-button" on:click={signOutUser}>sign out.</button>
    {:else}
        <button class="user-auth-button" on:click={signInWithGoogle}>sign in with google</button>
    {/if}
</div>
{/if}

<div id="map-container"></div>

{#if !isMobileSize}
    <div class="sidebar" style="display: {showMenu ? 'block' : 'none'}">
        {#if isMobileSize}
            <button class="close-button" on:click={toggleMenu}>X</button>
        {:else}
            <h1>click on the map to mark a hazard.</h1>
            {#if user}
                <p>welcome, {user.displayName}!</p>
                <p>[{contributions}] contributions</p>
                <button class="user-auth-button" on:click={signOutUser}>sign out.</button>
            {:else}
                <button class="user-auth-button" on:click={signInWithGoogle}>sign in with google</button>
            {/if}
            <hr>
        {/if}

        <h3>info:</h3>
        <p>click on the map to mark a hazard.</p>
        <p><b>black markers</b> denote ongoing alerts.</p>
        <p><b>red markers</b> denote alerts that are flagged as resolved. users can vote on whether the hazard is still present or gone by clicking on the marker.</p>
        

        <hr>
        <h3>regional trails and bicycle corridors:</h3>
        <input type="checkbox" id="checkbox-520"> 
        <label for="checkbox-520">520 trail</label><br>
        <input type="checkbox" id="checkbox-90"> 
        <label for="checkbox-90">i-90 trail</label><br>
        <input type="checkbox" id="checkbox-alki"> 
        <label for="checkbox-alki">alki trail/east marginal way</label><br>
        <input type="checkbox" id="checkbox-bgt"> 
        <label for="checkbox-bgt">burke gilman trail</label><br>
        <input type="checkbox" id="checkbox-dt-bellevue"> 
        <label for="checkbox-dt-bellevue">downtown bellevue (108th)</label><br>
        <input type="checkbox" id="checkbox-dt-seattle"> 
        <label for="checkbox-dt-seattle">downtown seattle (2nd, bell)</label><br>
        <input type="checkbox" id="checkbox-ckc"> 
        <label for="checkbox-ckc">eastrail/cross kirkland corridor</label><br>
        <input type="checkbox" id="checkbox-elst"> 
        <label for="checkbox-elst">east lake sammamish trail</label><br>
        <input type="checkbox" id="checkbox-srt"> 
        <label for="checkbox-srt">sammamish river trail</label><br>
        <input type="checkbox" id="checkbox-westlake-cycle"> 
        <label for="checkbox-westlake-cycle">ship canal trail/westlake cycletrack</label><br>
        <input type="checkbox" id="checkbox-lwl-kirkland"> 
        <label for="checkbox-lwl-kirkland">lake washington loop (kirkland)</label><br>
        <input type="checkbox" id="checkbox-lwl-renton"> 
        <label for="checkbox-lwl-renton">lake washington loop (south bellevue/renton)</label><br>
        <input type="checkbox" id="checkbox-lwl-south-seattle"> 
        <label for="checkbox-lwl-south-seattle">lake washington loop (south seattle/central district)</label>
    </div>
{/if}

{#if currentWindowOpen} 
    <div class="floating-window">
        <button class="close-button" on:click={closeFloatingWindow}>X</button>

        {#if currentWindowOpen == "selected marker" && selectedHazard}
            <h1>hazard: {selectedHazard.name}</h1>
            <h5>reported by: {selectedHazard.username}</h5>
            {#if selectedHazard.corridor}
                <h5>corridor: {selectedHazard.corridor}</h5>
            {/if}
            
            <p style="line-height: 1.6;">{selectedHazard.description != null ? selectedHazard.description : "no description provided"}</p>
            
            {#if !selectedHazard.status}
                <div class="warning-box">
                    <p><strong>⚠ This hazard may be resolved</strong></p>
                    <p>This hazard was flagged for review by a community member or is past the expected end date. As a crowdsourced tool, we rely on your reports to verify the presence of hazards.</p>
                    <p><strong>Last reported:</strong> {selectedHazard.report_date}</p>
                </div>
            {/if}

            <h5>comments</h5>
            <div class="comments-section">
                {#if comments.length == 0}
                    <p class="no-comments">no comments yet</p>
                {:else}
                    {#each comments as commentTuple}
                        <div class="comment">
                            <p class="comment-meta">{commentTuple[0]}</p>
                            <p class="comment-text">{commentTuple[1]}</p>
                        </div>
                    {/each}
                {/if}
            </div>

            {#if user}
                <textarea bind:value={amend_input} placeholder="add a comment"></textarea>
                <button on:click={submitAmendment} class="btn-full">submit comment</button>
                {#if !selectedHazard.status}
                    <h5>if you have been here, is this hazard...</h5>
                    <div class="button-group">
                        <button on:click={confirmedIncrement} class="btn-half btn-success">still present ({selectedHazard.confirmed})</button>
                        <button on:click={resolvedIncrement} class="btn-half btn-danger">not present ({selectedHazard.resolved})</button>
                    </div>
                {:else}
                    <button on:click={flagHazard} class="btn-full btn-warning">flag for review</button>
                {/if}
                {#if user.displayName == selectedHazard.username}
                    {#if !selectedHazard.status}
                        <button on:click={unflagHazard} class="btn-full btn-secondary">withdraw flag</button>
                    {/if}
                    <button on:click={deleteHazard} class="btn-full btn-danger">delete hazard</button>
                {/if}

            {/if}

        {:else if currentWindowOpen == "create hazard" && newMarkerCoords}
            <h1 class="hazard-title">add hazard</h1>
            <h3>give a name for the hazard (or use one of the dropdown options) *</h3>
            <input type="text" list="hazardTypeDataList" bind:value={newMarkerName}>
            <datalist id="hazardTypeDataList">
                {#each hazardTypes as type}
                    <option value={type}></option>
                {/each}
            </datalist>
            <h3>description</h3>
            <textarea placeholder="describe the hazard" bind:value={newMarkerDescription}></textarea>
            <h3>select bike corridor (if applicable)</h3>
            <select id="corridorSelect" bind:value={newMarkerCorridor}>
                {#each corridorList as type}
                    <option value={type}>{type}</option>
                {/each}
            </select>
            {#if user}
                <h3>posting as: {user.displayName}</h3>
            {/if}
            <button on:click={handleHazardCreation} class="btn-full btn-success">add hazard</button>

        {/if}
    </div>
{/if}

{#if isMobileSize && pageLoaded}
    <div class="quick-add-marker-container">
        {#if !loadingCurrentLocation}
            <button class="quick-add-marker" on:click={() => openHazardForm(null)}>add marker at current location</button>
        {:else}
            <button class="quick-add-marker">getting location...</button>
        {/if}
    </div>
{/if}

<style>
    .top-nav {
        background-color: var(--background);
        border-bottom: 1px solid var(--border-color);
        box-sizing: border-box;
        padding: 0.75rem 1.25rem;
        display: flex;
        justify-content: space-between;
        align-items: center;
        gap: 0.75rem;
        z-index: 5;
        left: 0;
        width: 100%;
        top: 0;
        height: 60px;
        position: fixed;
        box-shadow: var(--shadow-sm);
    }

    #map-container {
        height: 100vh;
        position: absolute;
        top: 0;
        left: 25%;
        width: 75%;
    }
    .sidebar {
        position: fixed;
        top: 0;
        left: 0;
        height: 100%;
        width: 25%;
        z-index: 4;
        padding: 1.5rem;
        box-sizing: border-box;
        overflow-x: hidden;
        overflow-y: auto;
        box-shadow: var(--shadow-lg);
        background-color: var(--background);
        border-right: 1px solid var(--border-color);
    }

    .floating-window {
        position: absolute;
        top: 50%;
        left: 27%;
        transform: translateY(-50%);
        width: 23%;
        max-height: 80%;
        overflow-x: hidden;
        overflow-y: auto;
        background-color: var(--background);
        padding: 1.75rem;
        box-shadow: var(--shadow-xl);
        border-radius: var(--radius-lg);
        border: 1px solid var(--border-color);
        box-sizing: border-box;
        z-index: 3;
    }

    .close-button {
        position: absolute;
        top: 1rem;
        right: 1rem;
        cursor: pointer;
        background-color: var(--surface);
        color: var(--text-secondary);
        padding: 0.5rem 0.75rem;
        border-radius: var(--radius-md);
        font-weight: 600;
        font-size: 0.875rem;
        border: 1px solid var(--border-color);
    }

    .close-button:hover {
        background-color: var(--danger-color);
        color: white;
        border-color: var(--danger-color);
    }

    .mobile-menu-button {
        background-color: var(--surface);
        color: var(--text-primary);
        border: 1px solid var(--border-color);
    }

    .mobile-menu-button:hover {
        background-color: var(--border-color);
    }

    .user-auth-button {
        background-color: var(--success-color);
        font-size: 0.8rem;
        padding: 0.5rem 1rem;
    }

    .user-auth-button:hover {
        background-color: #15803d;
    }

    @media (max-width: 768px) {
        #map-container {
            left: 0;
            width: 100%;
            top: 0;
            height: 100%;
        }

        .sidebar {
            left: 0;
            width: 100%;
            top: 60px;
            height: calc(100% - 60px);
            z-index: 3;
            padding: 1rem;
        }

        .floating-window {
            left: 0;
            width: 100%;
            top: auto;
            bottom: 0;
            transform: none;
            height: 60vh;
            max-height: 60vh;
            border-radius: var(--radius-lg) var(--radius-lg) 0 0;
            padding: 1.25rem;
        }

        .floating-window h1 {
            font-size: 1.5rem;
        }

        .floating-window h3 {
            font-size: 0.95rem;
        }

        .button-group {
            flex-direction: column;
        }

        .btn-half {
            width: 100%;
        }

        .top-nav {
            padding: 0.5rem 0.75rem;
        }

        .top-nav p {
            font-size: 0.75rem;
        }
    }

    .quick-add-marker-container {
        display: flex;
        justify-content: center;
        width: 100%;
    }

    .quick-add-marker {
        position: absolute;
        bottom: 60px;
        width: 75%;
        height: 60px;
        z-index: 2;
        box-shadow: var(--shadow-lg);
        border-radius: var(--radius-lg);
        box-sizing: border-box;
        padding: 0 1rem;
        background-color: var(--primary-color);
        color: white;
        font-weight: 600;
        font-size: 0.9rem;
    }

    .quick-add-marker:hover {
        background-color: var(--primary-hover);
        transform: translateY(-2px);
        box-shadow: var(--shadow-xl);
    }

    .floating-window h1 {
        margin-bottom: 1rem;
        padding-right: 2rem;
    }

    .floating-window h3 {
        margin-top: 1rem;
        margin-bottom: 0.5rem;
        color: var(--text-primary);
        font-size: 1rem;
    }

    .floating-window h5 {
        margin-bottom: 0.5rem;
    }

    .floating-window button {
        margin-top: 0.5rem;
    }

    .floating-window input[type="text"],
    .floating-window textarea,
    .floating-window select {
        width: 100%;
        margin-top: 0.25rem;
        margin-bottom: 0.5rem;
    }

    .floating-window textarea {
        min-height: 100px;
        resize: vertical;
    }

    .sidebar h3 {
        margin-top: 1.5rem;
        margin-bottom: 0.75rem;
        font-size: 1.1rem;
    }

    .sidebar p {
        color: var(--text-secondary);
        font-size: 0.875rem;
        line-height: 1.5;
    }

    .sidebar input[type="checkbox"] {
        margin-right: 0.5rem;
        vertical-align: middle;
    }

    .sidebar label {
        vertical-align: middle;
        line-height: 1.8;
    }

    .top-nav p {
        font-size: 0.85rem;
        color: var(--text-secondary);
        margin: 0;
    }

    /* Button variants */
    .btn-full {
        width: 100%;
        display: block;
    }

    .btn-half {
        width: calc(50% - 0.25rem);
        display: inline-block;
    }

    .button-group {
        display: flex;
        gap: 0.5rem;
        margin-top: 0.5rem;
    }

    .btn-danger {
        background-color: var(--danger-color);
    }

    .btn-danger:hover {
        background-color: var(--danger-hover);
    }

    .btn-success {
        background-color: var(--success-color);
    }

    .btn-success:hover {
        background-color: #15803d;
    }

    .btn-warning {
        background-color: var(--warning-color);
    }

    .btn-warning:hover {
        background-color: #c2410c;
    }

    .btn-secondary {
        background-color: var(--secondary-color);
    }

    .btn-secondary:hover {
        background-color: #475569;
    }

    .hazard-title {
        color: var(--danger-color);
    }

    /* Warning box */
    .warning-box {
        background-color: #fef2f2;
        border-left: 4px solid var(--danger-color);
        padding: 1rem;
        margin: 1rem 0;
        border-radius: var(--radius-sm);
    }

    .warning-box p {
        color: var(--danger-color);
        margin-bottom: 0.5rem;
        font-size: 0.875rem;
        line-height: 1.5;
    }

    .warning-box p:last-child {
        margin-bottom: 0;
    }

    /* Comment section */
    .comments-section {
        margin: 1rem 0;
        max-height: 300px;
        overflow-y: auto;
    }

    .comment {
        background-color: var(--surface);
        padding: 0.75rem;
        margin-bottom: 0.75rem;
        border-radius: var(--radius-md);
        border: 1px solid var(--border-color);
    }

    .comment-meta {
        color: var(--primary-color);
        font-weight: 600;
        font-size: 0.8rem;
        margin-bottom: 0.25rem;
    }

    .comment-text {
        color: var(--text-primary);
        font-size: 0.875rem;
        margin: 0;
        line-height: 1.5;
    }

    .no-comments {
        color: var(--text-secondary);
        font-style: italic;
        font-size: 0.875rem;
    }

</style>