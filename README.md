# California Wildfire Map
*SF Examiner Live Wildfire Tracking*

---

## Overview

Interactive Google Maps visualization showing active California wildfires in real-time. The map displays fire perimeters, acreage, containment status, and cause information sourced from the National Interagency Fire Center (NIFC).

**Live Map:** [View on SF Examiner](https://www.sfexaminer.com/wildfire-map) *(update with actual URL)*

---

## Features

- **Real-time data** from NIFC via Google Cloud Function
- **Interactive fire perimeters** with click for details
- **Hover tooltips** showing fire names
- **Mobile responsive** with adjusted zoom and info windows
- **California-focused** with boundary restrictions
- **Dark theme** for better fire visibility
- **Loading indicators** and error handling

---

## 🔒 Security Setup (CRITICAL)

### Google Maps API Key Restriction

The API key in `map.html` is currently **unrestricted** and visible in the source code. This poses security and billing risks.

**To secure it:**

1. Go to [Google Cloud Console → Credentials](https://console.cloud.google.com/apis/credentials)
2. Find API key: `AIzaSyBx-VcDFwhH3b7obq3aAnj1m8874HQ3vgM`
3. Click **Edit**
4. Under **Application restrictions**:
   - Select **HTTP referrers (web sites)**
   - Add these referrers:
     ```
     *.sfexaminer.com/*
     www.sfexaminer.com/*
     localhost:* (for testing)
     ```
5. Under **API restrictions**:
   - Select **Restrict key**
   - Enable **only**: Maps JavaScript API
6. Click **Save**

**Result:** The key will only work on sfexaminer.com domains, preventing unauthorized usage.

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

**Current:** All fires shown in same orange color  
**Improvement:** Color-code by severity/size

```javascript
map.data.setStyle(feature => {
  const acres = parseFloat(feature.getProperty('poly_GISAcres'));
  let color = 'rgba(255,165,0,0.6)'; // Default orange
  
  if (acres > 100000) color = 'rgba(139,0,0,0.7)';      // Dark red: mega fires
  else if (acres > 10000) color = 'rgba(255,69,0,0.7)'; // Red-orange: major
  else if (acres > 1000) color = 'rgba(255,140,0,0.6)'; // Orange: significant
  
  return {
    fillColor: color,
    strokeColor: color.replace('0.6', '0.9'),
    strokeWeight: 1
  };
});
```

### 2. **Filter Controls**

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
