# California Wildfire Map
*SF Examiner Live Wildfire Tracking*

---

## Overview

Interactive Mapbox GL experience showing active California wildfires in real-time. The page now renders a terrain/satellite basemap, overlays fire perimeters from the National Interagency Fire Center (NIFC), and layers Examiner-branded analytics (stats panel, top fires list, Datawrapper chart, filters, timeline slider, and air-quality tiles).

**Live Map:** [View on SF Examiner](https://www.sfexaminer.com/wildfire-map) *(update with actual URL)*

---

## Features

- **Live data feed** from the `serveNIFCGeojson` Cloud Function (NIFC perimeters, acreage, containment, metadata)
- **Mapbox GL renderer** with terrain/satellite toggle, CA max bounds, custom dark styling
- **Severity-driven symbology** (5 acreage buckets) plus interactive chip filters & containment dropdown
- **HUD analytics**: statewide totals, top 5 largest fires, Datawrapper year-to-date chart embed
- **Timeline slider** to scrub discovery dates and show how the season evolves
- **Air-quality overlay** (AQICN raster tiles) with quick toggle
- **Loading + error states** and “last updated” badge for data freshness

---

## Legacy Google Maps snapshot (Nov 2025)

The previous version used Google Maps JS with a static dark terrain style:

- **Entry point:** `map.html` called `initMap()` to mount Google Maps, apply custom styling, disable Street View, and clamp panning to California’s bounding box.
- **Data ingestion:** Fetched the same `serveNIFCGeojson` endpoint, added the features via `map.data.addGeoJson`, and showed a modal loading overlay plus a simple “last updated” label.
- **Symbology:** All perimeters were rendered with the same orange fill/stroke; no acreage/containment awareness.
- **Interactions:** Hover/click `InfoWindow`s displayed name, acreage, containment, cause, and description. Mobile tweaks were handled via `isMobile`.
- **UX guards:** Strict map bounds, but no analytics panes, color legend, filters, timeline, or AQ layer.

## Mapbox HUD (Dec 2025)

`map.html` now ships with a full Mapbox GL HUD:

1. **Stats & top fires panel (`#stats-panel`)**
   - Aggregates statewide totals and lists the top N incidents by acreage.
   - Hosts an optional Datawrapper embed for year-to-date acres burned (configurable chart ID).
2. **Severity-aware symbology**
   - Five acreage buckets defined in `SIZE_BUCKETS`.
   - Chip filters sync with Mapbox `setFilter` calls so the colors/legend stay in lockstep.
3. **Filters & overlays panel (`#legend-panel`)**
   - Contains the containment dropdown and the AQ toggle.
   - AQ overlay loads AQICN raster tiles (swap with your preferred provider/opacity).
4. **Timeline slider (`#timeline-panel`)**
   - Parses discovery timestamps (multiple candidate fields) and lets readers scrub to “fires discovered through” a given date.
5. **Basemap toggle**
   - Terrain vs. Satellite Mapbox styles; listeners rehydrate wildfire + AQ layers after every style swap.

All HUD blocks are mobile-friendly: on narrow screens the map becomes 60vh and panels stack beneath while retaining functionality.

---

## 🔐 Configuration & security

### Mapbox access token (required)

`map.html` reads `window.MAPBOX_ACCESS_TOKEN` (or `data-mapbox-token=""` on `<body>`) before Mapbox GL initializes. Create a **scoped token** in the [Mapbox dashboard](https://account.mapbox.com/access-tokens/) and limit it to:

- Allowed URLs: `https://www.sfexaminer.com/*`, `https://sfexaminer.com/*`, `http://localhost:*`
- Allowed scopes: **Styles: Read**, **Tilesets: Read**, **Datasets: Read** (no write scopes needed)

Embed the token either by:

```html
<script>window.MAPBOX_ACCESS_TOKEN = 'pk.xxx-your-token';</script>
<script src="/wildfire-map/map.html" defer></script>
```

or by server-side templating the `data-mapbox-token` attribute.

### Datawrapper embed (optional)

Set `data-datawrapper-chart="<chartID>"` (or `window.DATAWRAPPER_CHART_ID`) to load the YTD acres-burned line chart in the stats panel. Leave blank to hide the embed.

### Top fires limit & other knobs

- `data-top-fires="5"` controls how many fires appear in the “Largest active fires” list.
- Air-quality overlay currently uses the public AQICN **demo** token (`tiles.aqicn.org`). Swap in your paid token if you have one.
- All HUD controls and overlays load client-side only after the GeoJSON fetch succeeds; no server secrets are exposed.

---

## Recent Improvements (November 2025)

### ✅ Error Handling
- Added try/catch for GeoJSON loading failures
- User-friendly error message if data fetch fails
- Loading indicator while data downloads
- Console logging for debugging

### ✅ User Experience  
- "Last updated" timestamp displayed on map
- Loading spinner during data fetch
- Better mobile responsiveness

---

## Data Pipeline

### Current Setup

**Data Source:** NIFC (National Interagency Fire Center)  
**Delivery:** Google Cloud Function (`serveNIFCGeojson`)  
**Update Frequency:** Real-time (whenever NIFC publishes updates)  
**Format:** GeoJSON with fire perimeter polygons

### Data Fields Available:
- `poly_IncidentName` - Fire name
- `poly_GISAcres` - Acres burned
- `attr_PercentContained` - Containment percentage
- `attr_FireCause` - Cause of fire
- `attr_IncidentShortDescription` - Brief description
- Geometry coordinates for perimeter

---

## Potential Functional Improvements

### 1. **Fire Severity Indicators**

_Status: ✅ Completed via `SIZE_BUCKETS` severity palette + legend chips._

**Current:** All fires shown in same orange color  
**Improvement:** Color-code by severity/size

```javascript
const SIZE_BUCKETS = [
  { id: 'mega', min: 100_000, color: '#641220' },
  { id: 'major', min: 50_000, color: '#b21e35' },
  { id: 'large', min: 10_000, color: '#d94f30' },
  { id: 'medium', min: 1_000, color: '#f29e4c' },
  { id: 'small', min: 0, color: '#f6c177' }
];

function buildSeverityExpression() {
  const acresExpr = ['coalesce', ['to-number', ['get', '__acres']], 0];
  const expression = ['case'];
  SIZE_BUCKETS.forEach(bucket => {
    expression.push(['>=', acresExpr, bucket.min]);
    expression.push(bucket.color);
  });
  expression.push('#f6c177');
  return expression;
}
```

### 2. **Filter Controls**

_Status: ✅ Completed via chip filters + containment dropdown._

Add UI controls to filter fires by:
- **Size** (slider: 0-100k+ acres)
- **Containment** (show only <50% contained)
- **Cause** (human vs natural)
- **Date** (fires started in last 7/30/90 days)

### 3. **Historical Fire Archive**

**Integration with your data pipeline:**
- Daily snapshot of active fires
- Store in DataSF or similar
- Create Datawrapper charts showing:
  - Total acres burned per month/year
  - Number of active fires over time
  - Containment progress trends
  - Fire causes breakdown

### 4. **Fire Weather Integration**

Overlay weather data:
- Red flag warnings
- Wind speed/direction
- Humidity levels
- Temperature

### 5. **Evacuation Zones**

If CalFire provides evacuation data:
- Show evacuation zones in distinct color
- Add evacuation status in info windows
- Link to official evacuation resources

### 6. **Air Quality Integration**

_Status: ✅ Completed via AQICN raster overlay toggle._

Show AQI (Air Quality Index) overlays:
- Pull from PurpleAir or AirNow API
- Color-code regions by air quality
- Alert users to unhealthy air

### 7. **Search Functionality**

Let users search for:
- Specific fire names
- Their address/zip code
- County/city

### 8. **Share/Embed Capabilities**

- Social media share buttons
- Embeddable iframe version
- Direct link to specific fires

### 9. **Mobile App Features**

- Geolocation to show fires near user
- Push notifications for new fires nearby
- Offline mode with cached data

### 10. **Historical Comparison View**

Side-by-side or overlay comparison:
- This year vs last year same date
- Current season vs historical average
- Show fire perimeter growth over time

---

## Stretch roadmap: fire weather, search, history

### 8. Fire-weather overlays (winds, humidity, temperature)

- **Data sources:** NOAA HRRR grib files (wind speed/direction, humidity), NWS fire-weather zones (CAP/GeoJSON), or Open-Meteo’s tiled API.
- **Processing:** Batch convert the weather rasters into Mapbox-ready tiles (Tippecanoe for vector wind barbs, `rio-cogeo`/AWS for raster). Cache hourly slices so the client can switch timestamps quickly.
- **Frontend:** Add a new overlay toggle group next to AQ. Each toggle would add a dedicated Mapbox layer (e.g., wind arrows, humidity heatmap, red-flag polygons) and share the existing legend slot.
- **Automation:** Schedule a lightweight Cloud Function or GitHub Action to refresh weather tiles every 3 hours and publish to the CDN used by `map.html`.

### 10. Search & “fires near me”

- **Geocoder:** Use Mapbox’s Geocoding API + `@mapbox/mapbox-gl-geocoder` control to allow address/city searches. When a result is chosen, fly to it and display nearby fires.
- **Proximity filter:** After geocoding (or using `navigator.geolocation`), compute distance (Haversine) from the user’s point to each perimeter centroid. Highlight the nearest 3 fires and optionally auto-open the popup for the closest one.
- **Permissions:** Geolocation requires HTTPS + a user gesture. Provide a fallback (“Jump to my location” button) that gracefully degrades if the user declines.
- **Shareable links:** Encode the selected fire ID or lat/lon in the URL hash so editors can deep-link to a specific incident.

### 11. Historical archive & comparison mode

- **Data capture:** Extend the data pipeline to snapshot the GeoJSON nightly (see `wildfire_snapshot.py` example). Store both the full feature set and summary stats in `data_sources/wildfire/snapshots/YYYY/MM/DD/*.json`.
- **Charting:** Use the snapshots to power Datawrapper trend charts (acres by month, fire counts, containment averages) and expose the dataset via the existing GitHub Actions workflow.
- **UI toggle:** Add a “Season compare” control that swaps the live source with a selected historical snapshot. Combine with the existing timeline slider to replay a past season day-by-day.
- **Storage considerations:** A single season of daily snapshots is ~50–100 MB compressed. Include a retention policy (keep daily for current year, weekly for prior seasons) to minimize hosting costs.

---

## Priority Recommendations

### Immediate (This Week):
1. ✅ **Secure API key** - Restrict to sfexaminer.com
2. ✅ **Add error handling** - Already implemented
3. **Test on multiple devices** - Verify mobile experience

### Short-term (This Month):
4. **Add severity color coding** - Visual fire size/danger indicators
5. **Create historical archive** - Start capturing daily snapshots
6. **Build companion Datawrapper charts** - Acres burned trends, fire counts

### Long-term (Next Quarter):
7. **Add filter controls** - Let users customize view
8. **Integrate air quality data** - Show smoke impact
9. **Add evacuation zone overlays** - If data available
10. **Build automated alerts** - Email/push when fires appear

---

## Integration with SF Examiner Data Pipeline

### Potential Automation Opportunities:

**Daily Fire Snapshot Script:**
```python
#!/usr/bin/env python3
# wildfire_snapshot.py - Archive daily fire data

import requests
import json
from datetime import datetime

def capture_fire_snapshot():
    url = 'https://us-central1-sf-live-charts.cloudfunctions.net/serveNIFCGeojson'
    response = requests.get(url)
    data = response.json()
    
    # Extract summary stats
    fires = data['features']
    total_fires = len(fires)
    total_acres = sum(float(f['properties'].get('poly_GISAcres', 0)) for f in fires)
    
    # Save snapshot
    snapshot = {
        'date': datetime.now().isoformat(),
        'total_fires': total_fires,
        'total_acres': total_acres,
        'fires': fires
    }
    
    filename = f"snapshots/fires_{datetime.now().strftime('%Y%m%d')}.json"
    with open(filename, 'w') as f:
        json.dump(snapshot, f)
    
    return total_fires, total_acres

# Run daily, store results, push to Datawrapper charts
```

**Add to your GitHub Actions workflow:**
```yaml
- name: Archive Wildfire Data
  run: python wildfire_snapshot.py

- name: Update Wildfire Trend Charts
  env:
    DATAWRAPPER_API_KEY: ${{ secrets.DATAWRAPPER_API_KEY }}
  run: python wildfire_charts.py
```

---

## Technical Stack

- **Frontend:** Vanilla JavaScript, Google Maps API
- **Data Source:** NIFC via Google Cloud Function
- **Hosting:** Static HTML (GitHub Pages compatible)
- **Dependencies:** None (no build process needed)

---

## Files

- `map.html` - Main map visualization (400 lines)
- `README.md` - This documentation

---

## Next Steps

1. **Secure the API key** (instructions above)
2. **Test error handling** - Temporarily break the Cloud Function URL to verify error message
3. **Choose functional improvements** to implement
4. **Consider data pipeline integration** for historical tracking

---

*Last Updated: November 29, 2025*  
*Status: ✅ Functional with security and UX improvements*
