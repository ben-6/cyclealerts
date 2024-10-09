<svelte:head>
    <link href='https://api.mapbox.com/mapbox-gl-js/v2.14.1/mapbox-gl.css' rel='stylesheet' />
</svelte:head>

<script>
    import { onMount } from 'svelte';
    import { initializeApp, getApps } from 'firebase/app' 
    import { getDatabase, ref, onValue, query, orderByChild, update, push, runTransaction, serverTimestamp, remove, set, off } from 'firebase/database'
    import { GoogleAuthProvider, getAuth, signInWithPopup, onAuthStateChanged, signOut } from 'firebase/auth';
    import mapboxgl from 'mapbox-gl';


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

    let showMenu = false;
    let isMobileSize = false;
    let innerWidth = 0
    let innerHeight = 0
    
    let loadingCurrentLocation = false;


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
    const corridorList = ["", "520 trail", "I-90 trail", "alki trail/east marginal way", "burke gilman trail", "cedar river trail", "downtown bellevue (108th, ...)", "downtown seattle (2nd, ...)", "eastrail/cross kirkland corridor", "east lake sammamish trail", "sammamish river trail", "ship canal trail/westlake cycletrack", "lake washington loop (Kirkland)", "lake washington loop (south bellevue/renton)", "lake washington loop (south seattle/central district)"];

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

    function reportHazardResolved() {
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
        console.log("deleting");
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
        
        const hazardsRef = ref(database, 'hazards');
        onValue(hazardsRef, (snapshot) => {
            snapshot.forEach((childSnapshot) => {
                const markerData = childSnapshot.val();

                if (mapMarkers[childSnapshot.key]) {
                    mapMarkers[childSnapshot.key].remove();
                    delete mapMarkers[childSnapshot.key];
                }
                
                const hazard = new Hazard(childSnapshot.key, markerData.longitude, markerData.latitude, markerData.name, markerData.description, markerData.status, markerData.confirmed, markerData.resolved, markerData.report_date, markerData.corridor, markerData.username)
                createMarker(hazard, markerData.status == 1 ? "black" : "red");
            });
        });

        map.on('click', (e) => {
            openHazardForm(e.lngLat);
        });

        displayCorridor(map, "bgt.geojson", 'layer-bgt', 'checkbox-bgt');

    });

    function openHazardForm(useLocation) {
        console.log(useLocation);
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

        console.log(newMarkerCoords);

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
    } else {
        isMobileSize = true;
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

{#if !isMobileSize || showMenu}
    <div class="sidebar">
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
        <p><b>black markers</b> denote ongoing alerts.</p>
        <p><b>red markers</b> denote alerts that are expired or flagged as resolved. after 7 days, they will be archived unless a majority suggests the hazard is still present.</p>
        
        <hr>
        <h3>regional trails and bicycle corridors:</h3>
        <input type="checkbox" id="checkbox-bgt"> 
        <label for="checkbox-bgt">burke gilman trail</label>
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
                <p style="font-size: smaller; color: #d32f2f;">* this hazard may be resolved, proceed with caution.</p>
                <p style="font-size: smaller; color: #d32f2f;">this hazard was either flagged for review by a community member, or is past the expected end date. as a crowdsourced tool, we rely on your reports to verify the presence of hazards.</p>
                <p style="font-size: smaller; color: #d32f2f;">last reported: {selectedHazard.report_date}</p>
            {/if}

            <h5>comments</h5>
            {#if comments.length == 0}
                <p>no comments yet</p>
            {:else}
                {#each comments as commentTuple}
                    <p><b>{commentTuple[0]}</b>: {commentTuple[1]}</p>
                {/each}
            {/if}

            {#if user}
                <textarea bind:value={amend_input} placeholder="add a comment" style="width: calc(100% - 20px); padding: 10px; border: 1px solid #ccc; height: 80px;"></textarea>
                <button on:click={submitAmendment} style="width: 100%;">submit comment</button><br>
                {#if !selectedHazard.status}
                    <h5>if you have been here, is this hazard...</h5>
                    <button on:click={confirmedIncrement} style="width: 50%;display: inline-block;">still present ({selectedHazard.confirmed})</button><button on:click={resolvedIncrement} style="width: 50%;display: inline-block;">not present ({selectedHazard.resolved})</button>
                {:else}
                    <button on:click={reportHazardResolved} style="width: 100%;">flag</button>
                {/if}
                <br>
                {#if user.displayName == selectedHazard.username}
                    <button on:click={deleteHazard} style="width: 100%;">delete</button>
                {/if}
            
            {/if}

        {:else if currentWindowOpen == "create hazard" && newMarkerCoords}
            <h1 style="color: #d32f2f">add hazard</h1>
            <h3>give a name for the hazard (or use one of the dropdown options) *</h3>
            <input type="text" list="hazardTypeDataList" bind:value={newMarkerName} style="width: calc(100% - 20px); padding: 10px; border: 1px solid #ccc;">
            <datalist id="hazardTypeDataList">
                {#each hazardTypes as type}
                    <option value={type}></option>
                {/each}
            </datalist>
            <h3>description</h3>
            <textarea placeholder="describe the hazard" bind:value={newMarkerDescription} style="width: calc(100% - 20px); padding: 10px; border: 1px solid #ccc; height: 80px;"></textarea>
            <h3>select bike corridor (if applicable)</h3>
            <select id="corridorSelect" bind:value={newMarkerCorridor} style="width: 100%; padding: 10px; border: 1px solid #ccc;">
                {#each corridorList as type}
                    <option value={type}>{type}</option>
                {/each}
            </select>
            {#if user}
                <h3>posting as: {user.displayName}</h3>
            {/if}
            <button on:click={handleHazardCreation}>add hazard</button>

        {/if}
    </div>
{/if}

{#if isMobileSize}
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
        background-color: #f0f0f0; 
        box-sizing: border-box;
        padding: 20px 20px;
        display: flex;
        justify-content: space-between;
        align-items: center;
        z-index: 5;
        left: 0;
        width: 100%;
        top: 0;
        height: 60px;
        margin-top:0%;
        position: fixed;
        overflow: hidden;
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
        padding: 10px;
        box-sizing: border-box;
        overflow-x: hidden;
        overflow-y: auto;
        box-shadow: 2px 0 5px rgba(0,0,0,0.1);
        background-color: white;
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
        background-color: white;
        padding: 20px;
        box-shadow: 5px 0 5px rgba(0, 0, 0, 0.1);
        border-radius: 5px;
        box-sizing: border-box;
        z-index: 3;
    }

    .close-button {
        position: absolute;
        top: 20px;
        right: 20px;
        cursor: pointer;
    }

    @media (max-width: 768px) { 
        #map-container {
            left: 0%;
            width: 100%;
            top: 0%;
            height: 100%;
        }
        .sidebar {
            left: 0%;
            width: 100%;
            top: 60px;
            height: calc(100%-60px);
            z-index: 3;
        }

        .floating-window {
            left: 0%;
            width: 100%;
            top: 80%;
            height: 40%;
        }

    }

    .quick-add-marker-container {
        display: flex;
        justify-content: center; 
        width: 100%;
    }

    .quick-add-marker {
        position: absolute;
        bottom: 70px;
        width: 80%;
        height: 50px;
        z-index: 2;
        box-shadow: 5px 0 5px rgba(0, 0, 0, 0.1);
        border-radius: 5px;
        box-sizing: border-box;
        padding: 0 10px; 
    }

</style>