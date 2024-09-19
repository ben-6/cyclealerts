<svelte:head>

</svelte:head>

<script>
    import { onMount } from 'svelte';
    import { initializeApp, getApps } from 'firebase/app' 
    import { getDatabase, ref, onValue, query, orderByChild, update, push, runTransaction, serverTimestamp, remove, set } from 'firebase/database'
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
        if (user) {
            onValue(ref(database, `users/${user.uid}/contributions`), (snapshot) => {
                contributions = snapshot.val() || 0;
            });
        }
    });


    let map;
    let mapContainer;
    let currentInfoWindow = null;
    let mapMarkers = {};
    let contributions = 0;

    function initMap() {
        const {PinElement, AdvancedMarkerElement} = google.maps.importLibrary("marker");
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
                    mapMarkers[childSnapshot.key].map = null;
                }
                
                createMarker(childSnapshot.key, markerData.longitude, markerData.latitude, markerData.type, markerData.details, markerData.start_date, markerData.end_date, markerData.status, markerData.confirmed, markerData.resolved, markerData.report_date, markerData.corridor, markerData.username, markerData.display_name);
            });
        });

        const infoBox = document.getElementById("infoBox");

        map.controls[google.maps.ControlPosition.LEFT_TOP].push(infoBox);


        const hazardTypes = ["bad road design", "bike lane ends", "pothole/bad pavement", "closure", "railroad crossing", "bad visibility", "door zone hazard"];
        const corridorList = ["", "520 trail", "I-90 trail", "alki trail", "burke gilman trail", "cedar river trail", "cross kirkland corridor", "downtown bellevue (108th, 112th, ...)", "downtown seattle (2nd, 4th, ...)", "eastrail", "east lake sammamish trail", "sammamish river trail", "ship canal trail/westlake cycletrack", "lake washington loop (Kirkland)", "lake washington loop (south bellevue/renton)", "lake washington loop (south seattle/central district)"];

        // InfoWindow for creating new hazard
        var isLoggedInText = "";
        if (user) {
            isLoggedInText = `<h3>posting as: ${user.displayName}</h3>
                <input type="checkbox" id="post_anonymously">
                <label for="post_anonymously">post anonymously</label><br>`;
        }
        
        var infoWindowContentFront = `
            <h1 style="color: #d32f2f">add hazard</h1>
            <h3>give a name for the hazard (or use one of the dropdown options)</h3>
            <input type="text" list="hazardTypeDataList" id="hazardTypeInput" style="width: 100%; padding: 10px; margin-bottom: 10px; border: 1px solid #ccc;">
            <datalist id="hazardTypeDataList">
                ${hazardTypes.map(type => `<option value="${type}">`).join('')}
            </datalist>
            <h3>select start date</h3>
            <input type="date" id="addStartDate"></input>
            <h3>select end date (leave blank if unknown)</h3>
            <input type="date" id="addEndDate"></input>
            <h3>description (optional)</h3>
            <textarea id="addDetails" placeholder="describe the hazard" style="height: 80px;"></textarea>
            <h3>select bike corridor (if applicable)</h3>
            <select id="corridorSelect" style="width: 100%; padding: 10px; margin-bottom: 10px; border: 1px solid #ccc;">
                ${corridorList.map(type => `<option value="${type}">${type}</option>`).join('')}
            </select>`;
        
        var infoWindowContentBack = `<button id="addHazardBtn">add hazard</button>`;

        const newHazardInfoWindow = new google.maps.InfoWindow({
            content: infoWindowContentFront + isLoggedInText + infoWindowContentBack
        }); 
        onAuthStateChanged(auth, (currentUser) => {
            var isLoggedInText = "";
            if (currentUser) {
                isLoggedInText = `<h3>posting as: ${currentUser.displayName}</h3>
                    <input type="checkbox" id="post_anonymously">
                    <label for="post_anonymously">post anonymously</label><br><br>`;
            }
            newHazardInfoWindow.setContent(infoWindowContentFront + isLoggedInText + infoWindowContentBack);
        });
        

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
        });
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
                    var post_username = document.getElementById("post_anonymously").checked == true ? "anonymous user" : user.displayName;

                    const hazardType = document.getElementById("hazardTypeInput").value;
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
                        corridor: corridor,
                        username: user.displayName,
                        display_name: post_username
                    };

                    const hazardsRef = ref(database, 'hazards');
                    push(hazardsRef, hazardData)
                        .then((snapshot) => {
                            console.log("hazard data saved successfully!");
                            contributions ++;
                            set(ref(database, `users/${user.uid}/contributions`), contributions);
                            createMarker(snapshot.key, hazardData["longitude"], hazardData["latitude"], hazardData["type"], hazardData["details"], hazardData["start_date"], hazardData["end_date"], hazardData['status'], hazardData['confirmed'], hazardData['resolved'], hazardData['report_date'], hazardData['corridor'], hazardData['username'], hazardData['display_name']);
                        })
                        .catch((error) => {
                            console.error("error writing hazard data:", error);
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
    

    function createMarker(id, longitude, latitude, type, details, start_date, end_date, status, confirmed, resolved, report_date, corridor, username, display_name) {
        
        //console.log(latitude, longitude, type);
        const pinScaled = new google.maps.marker.PinElement({
            background: status ? '#ffffff' : '#ffff00',
        });

        mapMarkers[id] = new google.maps.marker.AdvancedMarkerElement({
            map: map,
            position: { lng: longitude, lat: latitude },
            title: type,
            content: pinScaled.element
        });
        
        var corridorHTML = corridor != "" ? '<h5>corridor: ' + corridor + '</h5>': "";
        
        
        const commentsRef = ref(database, 'comments/'+id);

        var infoWindow = new google.maps.InfoWindow({})

        onValue(query(commentsRef, orderByChild('timestamp')), (snapshot) => {
            var amendmentHTML = "<p>nothing here...</p>";
            snapshot.forEach((childSnapshot) => {
                if (amendmentHTML == "<p>nothing here...</p>") {
                    amendmentHTML = "";
                }
                amendmentHTML += `<p><b>${childSnapshot.val().username} - ${childSnapshot.val().date}</b>: ${childSnapshot.val().comment}</p>`
            });
            //console.log(amendmentHTML);
            var reportHTML = `
                <h1>hazard: ${type}</h1>
                <h5>reported by: ${display_name}</h5>
                <h5>expected start: ${start_date}</h5>
                <h5>expected end: ${end_date}</h5>
                `+corridorHTML+`
                <p style="line-height: 1.6;">${details}</p>
            `;
            if (!status) {
                reportHTML += `
                    <p style="font-size: smaller; color: #d32f2f;">* this hazard may be resolved, proceed with caution.</p>
                    <p style="font-size: smaller; color: #d32f2f;">this hazard was either flagged for review by a community member, or is past the expected end date. as a crowdsourced tool, we rely on your reports to verify the presence of hazards.</p>
                    <p style="font-size: smaller; color: #d32f2f;">last reported: ${report_date}</p>`;
            }
            reportHTML += `
                    <h5>comments</h5>`+amendmentHTML;


            infoWindow.setContent(reportHTML+createMarkerLoggedInText(user, username, status, confirmed, resolved));

            onAuthStateChanged(auth, (currentUser) => {
                infoWindow.setContent(reportHTML + createMarkerLoggedInText(currentUser, username, status, confirmed, resolved));
            });

            // Click event to open InfoWindow
            mapMarkers[id].addListener("click", (function(infoWindow) { // Pass infoWindow into an IIFE
                return () => {
                    if (currentInfoWindow) {
                        currentInfoWindow.close();
                    }
                    infoWindow.open(map, mapMarkers[id]); 
                    currentInfoWindow = infoWindow;
                };
            })(infoWindow));



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
                onAuthStateChanged(auth, (currentUser) => {
                    if (currentUser && username == currentUser.displayName) {
                        document.getElementById("deleteHazard").addEventListener("click", () => {
                            deleteHazard(id);
                        });
                    }
                });
                
            });

        });

        
    }

    function createMarkerLoggedInText(user, username, status, confirmed, resolved) {
        var loggedInText = "";
        var canDeleteText = "";
        
        if (user) {
            loggedInText = `<div style="display: block">`;
        } else {
            loggedInText = `<div style="display: none">`;
        }
        
        loggedInText += `
            <textarea id="amend_input" placeholder="add a comment" style="height: 80px;"></textarea>
            <input type="checkbox" id="amend_anonymously">
            <label for="amend_anonymously">comment anonymously</label><br>
            <input type="button" id="submitAmendment" value="submit comment" style="margin-top:10px; background-color: #4CAF50; color: white; width: 40%;"><br>`;
        if (!status) {
            loggedInText += `
                <h5>if you have been here, is this hazard...</h5>
                <input type="button" id="confirmedIncrement" value="still present (${confirmed})" style="background-color: #4CAF50; color: white; width: 20%;display: inline-block;"><input type="button" id="resolvedIncrement" value="not present (${resolved})" style="background-color: #f44336; color: white; width: 20%";display: inline-block;>`;
        } else {
            loggedInText += `
                <input type="button" id="reportHazardResolved" value="flag" style="background-color: #f44336; color: white; width: 40%;">`;
        }

        if (user && user.displayName == username) {
            canDeleteText = `<br><input type="button" id="deleteHazard" value="delete" style="width: 40%;">`;
        }
        
        return loggedInText+canDeleteText+'</div>';
    }


    function handleLocationError(browserHasGeolocation, pos) {
        const infoWindow = new google.maps.InfoWindow({ // Using standard InfoWindow
            content: browserHasGeolocation
                ? 'error: the geolocation service failed.'
                : 'error: your browser doesn\'t support geolocation.',
            position: pos,
        });
        infoWindow.open(map);
    }

    function reportHazardResolved(id) {
        const hazardRef = ref(database, 'hazards/' + id);

        // Update a single property
        update(hazardRef, { status: 0, confirmed: 0, resolved: 0, report_date: new Date().toISOString().slice(0, 10)})
            .then(() => {
                console.log("hazard data updated successfully!");
            })
            .catch((error) => {
                console.error("error updating hazard data:", error);
            });

        if (currentInfoWindow) {
            currentInfoWindow.close(); // Close the previous InfoWindow
        }
    }

    function submitAmendment(id) {
        var amend_input = document.getElementById("amend_input").value.trim()
        
        var amend_username = document.getElementById("amend_anonymously").checked == true ? "anonymous user" : user.displayName;

        const hazardRef = ref(database, 'comments/' + id);
        
        push(hazardRef, {"username": amend_username, "date": new Date().toISOString().slice(0, 10), "comment": amend_input, "timestamp": serverTimestamp()})
            .then(() => {
                console.log("key-value pair added successfully!");
            })
            .catch((error) => {
                console.error("error adding key-value pair:", error);
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
                console.log("value incremented successfully!");
            })
            .catch((error) => {
                console.error("error incrementing value:", error);
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
                console.log("value incremented successfully!");
            })
            .catch((error) => {
                console.error("error incrementing value:", error);
            });
        
        if (currentInfoWindow) {
            currentInfoWindow.close(); // Close the previous InfoWindow
        }
    }

    function deleteHazard(id) {
        console.log("deleting");
        const hazardRef = ref(database, 'hazards/' + id);
        remove(hazardRef);
        const commentsRef = ref(database, 'comments/' + id);
        remove(commentsRef);
        mapMarkers[id].map = null;
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
        <p>[{contributions}] contributions</p>
        <button class="user-auth-button" on:click={signOutUser}>sign out.</button>
    {:else}
        <button class="user-auth-button" on:click={signInWithGoogle}>sign in with google</button>
    {/if}

    <p>in a <a href="https://web.pdx.edu/~jdill/Types_of_Cyclists_PSUWorkingPaper.pdf">2012 portland state university study</a>, only ~10% of cyclists consider themselves as enthused & confident / strong & fearless.
    many don't cycle because of the <b>unpredictability of bike lanes and roads.</b></p>
    <p>cyclealerts is my attempt at reducing the local knowledge threshold for the ~50% of interested but concerned cyclists.</p>
    <p><b>white markers</b> denote ongoing alerts.</p>
    <p><b>yellow markers</b> denote alerts that are expired or flagged as resolved. after 7 days, they will be archived unless a majority suggests the hazard is still present.</p>
    
    
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

    .user-auth-button {
        background-color: #6db4ff;
        color: white;
        padding: 10px 20px;
        border: none;
        border-radius: 5px;
        cursor: pointer;
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
        width: 100%;
    }
    :global(input[value="flag"]) { 
        background-color: #f44336; 
        color: white; 
        width: 100%;
    }

</style>