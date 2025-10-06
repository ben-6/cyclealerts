# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

CycleAlerts is a crowdsourced bicycle hazard mapping application for the Seattle/Bellevue area. Users can report, flag, and track cycling hazards in real-time using an interactive map interface. The application is built with SvelteKit and Firebase, using Mapbox GL for map visualization.

## Tech Stack

- **Frontend**: SvelteKit 2.0 + Svelte 4
- **Map**: Mapbox GL JS
- **Backend**: Firebase (Realtime Database + Auth)
- **Hosting**: Firebase Functions support (currently minimal usage)
- **Build Tools**: Vite

## Development Commands

```bash
# Install dependencies
npm install

# Start development server
npm run dev

# Build for production
npm build

# Preview production build
npm run preview

# Format code
npm run format

# Lint code
npm run lint
```

## Firebase Functions

The `functions/` directory contains Firebase Cloud Functions setup, though currently minimal:

```bash
cd functions
npm run serve     # Start emulator
npm run deploy    # Deploy functions
npm run logs      # View logs
```

## Architecture

### Single-Page Application Structure

The entire application is contained in [`src/routes/+page.svelte`](src/routes/+page.svelte). This monolithic component handles:

- Firebase initialization and authentication (Google OAuth)
- Mapbox map initialization and management
- Hazard CRUD operations
- Real-time data synchronization
- Comment system for hazards
- Corridor/trail overlay visualization

### Key Classes and Data Models

**Hazard Class** (lines 85-103 in +page.svelte):
```javascript
class Hazard {
    constructor(id, longitude, latitude, name, description, status, confirmed, resolved, report_date, corridor, username)
}
```

**Hazard States**:
- `status: 1` = Active hazard (black marker)
- `status: 0` = Flagged/potentially resolved (red marker)
- `confirmed` = Vote count for "still present"
- `resolved` = Vote count for "not present"

### Firebase Database Structure

```
/hazards/{hazardId}
  - longitude, latitude
  - name, description
  - status, confirmed, resolved
  - report_date, corridor
  - username

/comments/{hazardId}/{commentId}
  - username, date, comment
  - timestamp

/users/{userId}
  - contributions (count)
```

### Environment Variables

Required in `.env` (Vite format with `VITE_` prefix):
- `VITE_FIREBASE_API_KEY`
- `VITE_FIREBASE_AUTH_DOMAIN`
- `VITE_FIREBASE_DATABASE_URL`
- `VITE_FIREBASE_PROJECT_ID`
- `VITE_FIREBASE_STORAGE_BUCKET`
- `VITE_FIREBASE_MESSAGING_SENDER_ID`
- `VITE_FIREBASE_APP_ID`
- `VITE_FIREBASE_MEASUREMENT_ID`
- `VITE_MAPBOX_API_KEY`

### Corridor GeoJSON Files

Located in [`src/lib/corridor_geojson/`](src/lib/corridor_geojson/), these define bike corridors/trails:
- 520 trail, I-90 trail, Burke-Gilman Trail
- Lake Washington Loop sections
- Downtown Seattle/Bellevue corridors
- Various regional trails

Each corridor is loaded and rendered as a toggleable line layer on the map.

### Map Interaction Flow

1. **Adding Hazards**: Click map → Creates blue temporary marker → Fill form → Submit
2. **Viewing Hazards**: Click marker → Opens floating window with details
3. **Flagging**: Active hazards can be flagged (turns marker red)
4. **Voting**: Flagged hazards allow "still present" / "not present" voting
5. **Comments**: Authenticated users can add comments to any hazard

### Responsive Design

- Desktop (>768px): Sidebar on left (25% width), map on right (75% width)
- Mobile (≤768px): Top navigation bar, fullscreen map, bottom floating panels

## Code Patterns

### Real-time Listeners

The app uses Firebase's `onValue()` for real-time updates:
- Hazards are synced automatically (lines 368-396)
- Individual hazards update when modified (lines 377-390)
- Comments sync per-hazard (lines 180-200)
- User contributions track via reactive listener (lines 52-55)

### State Management

All state is local to the Svelte component using reactive declarations:
- `$:` reactive statements for derived state (lines 196-200, 471-477)
- Window state controlled by `currentWindowOpen` variable
- Selected hazard state in `selectedHazard`

### Marker Management

Markers stored in `mapMarkers` object keyed by hazard ID:
- `createMarker()` function handles creation and click events (lines 159-178)
- Markers removed/recreated on updates (lines 381-387)
- Temporary marker (`tempMarker`) for new hazard placement

## Important Development Notes

### Modifying Hazard Types

Predefined hazard types are in `hazardTypes` array (line 105). Update this to add/remove hazard categories.

### Adding New Corridors

1. Add GeoJSON file to [`src/lib/corridor_geojson/`](src/lib/corridor_geojson/)
2. Add checkbox in sidebar HTML (around lines 520-547)
3. Call `displayCorridor()` in map's load event (lines 402-416)
4. Add to `corridorList` array (line 106)

### Authentication Requirements

User must be signed in to:
- Create hazards
- Add comments
- Flag hazards
- Vote on hazard status
- Delete own hazards

Check `user` variable for auth state throughout component.

### Mobile Behavior

Mobile view includes "add marker at current location" button (lines 625-631) that uses browser geolocation API.
