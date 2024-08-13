<svelte:head>

</svelte:head>

<script>
    import { onMount } from 'svelte';
    import { initializeApp, getApps } from 'firebase/app' 
    import { getDatabase, ref, onValue, query, orderByChild, update, push, runTransaction, serverTimestamp } from 'firebase/database'
    import { GoogleAuthProvider, getAuth, signInWithPopup, onAuthStateChanged, signOut } from 'firebase/auth';

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
    });


    let map;
    let mapContainer;
    let currentInfoWindow = null;
    let mapMarkers = {};

    function initMap() {
        const { AdvancedMarkerElement, PinElement } = google.maps.importLibrary("marker"); // Import AdvancedMarker

        map = new google.maps.Map(mapContainer, {
            center: { lat: 47.608027, lng: -122.308954 },
            zoom: 12,
            mapId: '4415a3f2c5782645'
        });

        const bikeLayer = new google.maps.BicyclingLayer();

        bikeLayer.setMap(map);

        if (navigator.geolocation) {
            navigator.geolocation.getCurrentPosition(
            (position) => {
                const pos = {
                    lng: position.coords.longitude,
                    lat: position.coords.latitude,
                };
                map.setCenter(pos);
            },
            () => {
                handleLocationError(true, map.getCenter());
            }
            );
        } else {
            handleLocationError(false, map.getCenter());
        }
        const hazardsRef = ref(database, 'hazards');
        onValue(hazardsRef, (snapshot) => {
            snapshot.forEach((childSnapshot) => {
                const markerData = childSnapshot.val();

                if (mapMarkers[childSnapshot.key]) {
                    mapMarkers[childSnapshot.key].setMap(null);
                }
                
                createMarker(childSnapshot.key, markerData.longitude, markerData.latitude, markerData.type, markerData.details, markerData.start_date, markerData.end_date, markerData.status, markerData.confirmed, markerData.resolved, markerData.report_date, markerData.corridor);
            });
        });




        const hazardTypes = ["pothole/bad pavement", "closure", "debris (glass, wet leaves, etc.)", "bad road design", "other"];
        const corridorList = ["", "520 Trail", "I-90 Trail", "Alki Trail", "Burke Gilman Trail", "Cross Kirkland Corridor", "Eastrail", "East Lake Sammamish Trail", "Sammamish River Trail", "SODO Trail", "Lake Washington Loop (Kirkland)", "Lake Washington Loop (South Bellevue/Renton)", "Lake Washington Loop (South Seattle/Central District)"];

        // InfoWindow for creating new hazard
        const newHazardInfoWindow = new google.maps.InfoWindow({
            content: `
                <h1 style="color: #d32f2f">Add Hazard</h1>
                <h3>Select Hazard Type</h3>
                <select id="hazardTypeSelect" style="width: 100%; padding: 10px; margin-bottom: 10px; border: 1px solid #ccc;">
                    ${hazardTypes.map(type => `<option value="${type}">${type}</option>`).join('')}
                </select>
                <h3>Select Start Date</h3>
                <input type="date" id="addStartDate"></input>
                <h3>Select End Date (leave blank if unknown)</h3>
                <input type="date" id="addEndDate"></input>
                <h3>Description</h3>
                <textarea id="addDetails" placeholder="Describe the hazard" style="height: 80px;"></textarea>
                <h3>Select Bike Corridor (if applicable)</h3>
                <select id="corridorSelect" style="width: 100%; padding: 10px; margin-bottom: 10px; border: 1px solid #ccc;">
                    ${corridorList.map(type => `<option value="${type}">${type}</option>`).join('')}
                </select>
                <button id="addHazardBtn">Add Hazard</button>
            `
        });
        
        const infoBox = document.getElementById("infoBox");

        map.controls[google.maps.ControlPosition.LEFT_TOP].push(infoBox); 

        // Click event to open InfoWindow for new hazard
        map.addListener("click", (event) => {
            if (currentInfoWindow) {
                currentInfoWindow.close(); // Close the previous InfoWindow
            }
            newHazardInfoWindow.setPosition(event.latLng);
            newHazardInfoWindow.open(map);	
            currentInfoWindow = newHazardInfoWindow;	
        });

        google.maps.event.addListener(newHazardInfoWindow, "domready", function() {
            handleHazardCreation(newHazardInfoWindow);
        })
    }

    
    function handleHazardCreation(newHazardInfoWindow) {
        const addStartDateBtn = document.getElementById("addStartDate");

        //console.log("addStartDateBtn: " + addStartDateBtn);
        if (addStartDateBtn) {
            const today = new Date();
            const year = today.getFullYear();
            let month = today.getMonth() + 1; // months are zero-indexed
            let day = today.getDate();

            // Add leading zeros if needed
            month = month < 10 ? `0${month}` : month;
            day = day < 10 ? `0${day}` : day;

            const formattedDate = `${year}-${month}-${day}`;
            addStartDateBtn.value = formattedDate;
        }

        const addHazardBtn = document.getElementById("addHazardBtn");
        
        //console.log("addHazardBtn: " + addHazardBtn);
        if (addHazardBtn) {
            addHazardBtn.addEventListener("click", () => {
                if (user) {
                    const hazardType = document.getElementById("hazardTypeSelect").value;
                    const details = document.getElementById("addDetails").value;
                    const start_date = document.getElementById("addStartDate").value;
                    const end_date = document.getElementById("addEndDate").value;
                    const corridor = document.getElementById("corridorSelect").value;

                    const latLng = newHazardInfoWindow.getPosition();
                    
                    document.getElementById("addStartDate").value = '';
                    document.getElementById("addEndDate").value = '';
                    document.getElementById("addDetails").value = ''; 

                    const hazardData = {
                        longitude: latLng.lng(),
                        latitude: latLng.lat(),
                        type: hazardType,
                        details: details,
                        start_date: start_date,
                        end_date: end_date,
                        status: betweenDateRange(start_date, end_date, 1),
                        confirmed: 0,
                        resolved: 0,
                        report_date: status === 0 ? new Date().toISOString().slice(0, 10) : null,
                        corridor: corridor
                    };

                    const hazardsRef = ref(database, 'hazards');
                    push(hazardsRef, hazardData)
                        .then((snapshot) => {
                            console.log("Hazard data saved successfully!");

                            createMarker(snapshot.key, hazardData["longitude"], hazardData["latitude"], hazardData["type"], hazardData["details"], hazardData["start_date"], hazardData["end_date"], hazardData['status'], hazardData['confirmed'], hazardData['resolved'], hazardData['report_date'], hazardData['corridor']);
                        })
                        .catch((error) => {
                            console.error("Error writing hazard data:", error);
                        }); 
                    
                    
                    newHazardInfoWindow.close();
                } else {
                    alert("please sign in to use this feature.");
                }
                
                
            });
        }

    }

    function betweenDateRange(startDate, endDate, status) {
        const now = new Date(); // Get the current date and time
        const today = new Date(now.getFullYear(), now.getMonth(), now.getDate()); // Normalize to just the date (no time)
    
        // Parse start and end dates
        const parsedStartDate = startDate ? new Date(startDate) : null;
        const parsedEndDate = endDate ? new Date(endDate) : null;
    
        if (parsedStartDate && parsedEndDate) {
            // Check if today is between start_date and end_date
            if (parsedStartDate <= today && today <= parsedEndDate) {
                //console.log("status:", status);
                return status;
            }
        } else if (parsedStartDate && !parsedEndDate) {
            // Check if today is within 7 days of start_date
            const sevenDaysFromStart = new Date(parsedStartDate);
            sevenDaysFromStart.setDate(sevenDaysFromStart.getDate() + 7);
        
            if (today <= sevenDaysFromStart) {
                //console.log("status:", status);
                //console.log(sevenDaysFromStart, today);
                return status;
            }
        }
        return 0; // Not active if none of the conditions are met
    }
    

    function createMarker(id, longitude, latitude, type, details, start_date, end_date, status, confirmed, resolved, report_date, corridor) {
        
        //console.log(latitude, longitude, type);
        const pinScaled = new google.maps.marker.PinElement({
            background: status ? '#ffffff' : '#ffff00',
        });

        const marker = new google.maps.marker.AdvancedMarkerElement({
            map: map,
            position: { lng: longitude, lat: latitude },
            title: type,
            content: pinScaled.element
        });
        
        var corridorHTML = corridor != "" ? '<h5 >Corridor: ' + corridor + '</h5>': "";
        
        
        const commentsRef = ref(database, 'comments/'+id);

        var infoWindow = new google.maps.InfoWindow({})

        onValue(query(commentsRef, orderByChild('timestamp')), (snapshot) => {
            var amendmentHTML = "";
            snapshot.forEach((childSnapshot) => {
                amendmentHTML += `<p><b>${childSnapshot.val().username} - ${childSnapshot.val().date}</b>: ${childSnapshot.val().comment}</p>`
            });
            //console.log(amendmentHTML);
            var reportHTML = "";
            if (!status) {
                reportHTML = `
                    <h1>Hazard: ${type}</h1>
                    <h5>Expected Start: ${start_date}</h5>
                    <h5>Expected End: ${end_date}</h5>
                    `+corridorHTML+`
                    <p style="line-height: 1.6;">${details}</p>
                    <p style="font-size: smaller; color: #d32f2f;">* This hazard may be resolved, proceed with caution.</p>
                    <p style="font-size: smaller; color: #d32f2f;">This hazard was either reported as resolved by a community member, or is past the expected end date. As a crowdsourced tool, we rely on your reports to verify the presence of hazards.</p>
                    <p style="font-size: smaller; color: #d32f2f;">Last reported: ${report_date}</p>
                    </div>
                    <h5>Comments</h5>
                    `+amendmentHTML+`
                    <textarea id="amend_input" placeholder="Add a comment" style="height: 80px;"></textarea>
                    <textarea id="amend_username" placeholder="Name (optional)"></textarea>
                    <input type="button" id="submitAmendment" value="Submit comment"><br>
                    <input type="button" id="confirmedIncrement" value="This hazard is still present (${confirmed})">
                    <input type="button" id="resolvedIncrement" value="This hazard is no longer present (${resolved})">`;
            } else {
                reportHTML = `
                    <h1>Hazard: ${type}</h1>
                    <h5>Expected Start: ${start_date}</h5>
                    <h5>Expected End: ${end_date}</h5>
                    `+corridorHTML+`
                    <p style="line-height: 1.6;">${details}</p>
                    <p style="font-size: smaller; color: #d32f2f;">As a crowdsourced tool, we rely on your reports to verify the presence of hazards.</p>
                    <h5>comments:</h5>
                    `+amendmentHTML+`
                    <textarea id="amend_input" placeholder="Add a comment" style="height: 80px;"></textarea>
                    <textarea id="amend_username" placeholder="Name (optional)"></textarea>
                    <input type="button" id="submitAmendment" value="submit comment">
                    <input type="button" id="reportHazardResolved" value="flag for review">`;
            }
            infoWindow.setContent(reportHTML);

            // Click event to open InfoWindow
            marker.addListener("click", (function(infoWindow) { // Pass infoWindow into an IIFE
                return () => {
                if (currentInfoWindow) {
                    currentInfoWindow.close();
                }
                infoWindow.open(map, marker); 
                currentInfoWindow = infoWindow;
                };
            })(infoWindow));

            mapMarkers[id] = marker;


            google.maps.event.addListenerOnce(infoWindow, 'domready', () => {
                document.getElementById("submitAmendment").addEventListener("click", () => {
                    if (user) {
                        submitAmendment(id);
                    } else {
                        alert("please sign in to use this feature.");
                    }
                });
                
                if (!status) {
                    document.getElementById("confirmedIncrement").addEventListener("click", () => {
                        if (user) {
                            confirmedIncrement(id);
                        } else {
                            alert("please sign in to use this feature.");
                        }
                    });
                    document.getElementById("resolvedIncrement").addEventListener("click", () => {
                        if (user) {
                            resolvedIncrement(id);
                        } else {
                            alert("please sign in to use this feature.");
                        }
                    });
                } else {
                    document.getElementById("reportHazardResolved").addEventListener("click", () => {
                        if (user) {
                            reportHazardResolved(id);
                        } else {
                            alert("please sign in to use this feature.");
                        }
                    });
                }
                
            });

        });

        
    }


    function handleLocationError(browserHasGeolocation, pos) {
        const infoWindow = new google.maps.InfoWindow({ // Using standard InfoWindow
            content: browserHasGeolocation
                ? 'Error: The Geolocation service failed.'
                : 'Error: Your browser doesn\'t support geolocation.',
            position: pos,
        });
        infoWindow.open(map);
    }

    function reportHazardResolved(id) {
        const hazardRef = ref(database, 'hazards/' + id);

        // Update a single property
        update(hazardRef, { status: 0, confirmed: 0, resolved: 0, report_date: new Date().toISOString().slice(0, 10)})
            .then(() => {
                console.log("Hazard data updated successfully!");
            })
            .catch((error) => {
                console.error("Error updating hazard data:", error);
            });

        if (currentInfoWindow) {
            currentInfoWindow.close(); // Close the previous InfoWindow
        }
    }

    function submitAmendment(id) {
        var amend_input = document.getElementById("amend_input").value.trim()
        var amend_username = document.getElementById("amend_username").value.trim()
        amend_username = amend_username == "" ? "user" : amend_username;

        const hazardRef = ref(database, 'comments/' + id);
        
        push(hazardRef, {"username": amend_username, "date": new Date().toISOString().slice(0, 10), "comment": amend_input, "timestamp": serverTimestamp()})
            .then(() => {
                console.log("Key-value pair added successfully!");
            })
            .catch((error) => {
                console.error("Error adding key-value pair:", error);
            });

        if (currentInfoWindow) {
            currentInfoWindow.close(); // Close the previous InfoWindow
        }

    }

    function confirmedIncrement(id) {
        const hazardRef = ref(database, 'hazards/' + id + '/confirmed');

        runTransaction(hazardRef, (currentCount) => {
            return (currentCount || 0) + 1; // Increment, or start at 1 if it doesn't exist
        })
            .then(() => {
                console.log("Value incremented successfully!");
            })
            .catch((error) => {
                console.error("Error incrementing value:", error);
            });

        if (currentInfoWindow) {
            currentInfoWindow.close(); // Close the previous InfoWindow
        }
    }

    function resolvedIncrement(id) {
        const hazardRef = ref(database, 'hazards/' + id + '/resolved');

        runTransaction(hazardRef, (currentCount) => {
            return (currentCount || 0) + 1; // Increment, or start at 1 if it doesn't exist
        })
            .then(() => {
                console.log("Value incremented successfully!");
            })
            .catch((error) => {
                console.error("Error incrementing value:", error);
            });
        
        if (currentInfoWindow) {
            currentInfoWindow.close(); // Close the previous InfoWindow
        }
    }

    

    onMount(async () => {
        if (window.google && window.google.maps) {
            initMap();
        } else {
            // If not loaded, create and append the script tag
            const script = document.createElement('script');
            script.src = `https://maps.googleapis.com/maps/api/js?key=${import.meta.env.VITE_GOOGLE_MAPS_API_KEY}`;   

            script.defer = true;
            script.onload = initMap; // Initialize the map once the script is loaded
            document.head.appendChild(script);
        }
        
    });
</script>



<div id="map" bind:this={mapContainer}></div>

<div id="infoBox">
    <h1>click on the map to mark a hazard.</h1>
    {#if user}
        <p>welcome, {user.displayName}!</p>
        <button on:click={signOutUser}>sign out.</button>

        
    {:else}
        <button on:click={signInWithGoogle}>Sign in with Google</button>
    {/if}

    <p>in a <a href="https://web.pdx.edu/~jdill/Types_of_Cyclists_PSUWorkingPaper.pdf">2012 portland state university study</a>, only ~10% of cyclists consider themselves as enthused & confident / strong & fearless.
    many don't cycle because of the <b>unpredictability of bike lanes and roads.</b></p>
    <p>cyclealerts is my attempt at reducing the local knowledge threshold for the ~50% of interested but concerned cyclists.</p>
    
</div>


<style>
    #map {
        height: 100vh; /* Adjust as needed */
        width: 100%;
    }

    #infoBox {
        position: absolute;
        background-color: white;
        z-index: 10;
        margin: 20px;
        padding: 10px;
        width: 20%;
        
    }

    :global(textarea) { 
        width: 100%; 
        padding: 10px; 
        margin-bottom: 10px; 
        border: 1px solid #ccc; 
    }
    :global(textarea:focus) {
        outline: none; 
        border-color: #4CAF50;
        box-shadow: 0 0 5px #4CAF50;
    }

    :global(input[type="button"]) { 
        padding: 10px 20px; 
        border: none; 
        cursor: pointer; 
    }
    :global(input[value="submit comment"]) { 
        background-color: #4CAF50; 
        color: white; 
    }
    :global(input[value="flag for review"]) { 
        background-color: #f44336; 
        color: white; 
        margin-left: 10px; 
    }

</style>