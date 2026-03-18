# Ice Risk Monitor (with optional NWS)

![ice risk](https://github.com/user-attachments/assets/ee6195c8-88a6-498f-b6f3-992c25c7cedf)


An automated property weather monitoring system that detects ice risk conditions across multiple locations and sends instant email alerts. Optionally enriches alerts with live National Weather Service (NWS) data for US-based properties.

---

## What It Does

Runs twice daily (6am and 10pm by default via cron) or on manual trigger. For each property in your tracking sheet, it checks current and forecast weather data, runs an ice risk evaluation algorithm, and fires an email alert if conditions meet the ice risk threshold.

**Ice is detected when all three conditions are true:**
- Precipitation occurred in the last 6 hours
- Current temperature or overnight low is at or below 32°F
- Snowfall in the window is below 0.5cm (rain/sleet freezing, not heavy snow)

For US properties, NWS active alerts are pulled and appended to the notification for additional context.

---

## Architecture

```
Schedule Trigger (6am / 10pm cron) OR Manual Trigger
        │
        ▼
Get property details from Google Sheets
        │
        ▼
Does Lat/Long exist in sheet?
   │                    │
  YES                   NO
   │                    ▼
   │         Get Lat/Long from OpenStreetMap (Nominatim)
   │                    │
   │         Normalize coordinates
   │                    │
   │         Update sheet with resolved coordinates
   │                    │
   └────────────────────┘
        │
        ▼
Get weather forecast (Open-Meteo API)
        │
        ▼
Preserve property data alongside weather
        │
        ▼
Ice risk evaluation (custom JS logic)
        │
        ▼
Was ice risk detected?
   │
  YES
   │
   ▼
Check if property is in US (lat/lon bounds check)
   │                    │
  YES (US)              NO (non-US)
   │                    │
   ▼                    ▼
Get NWS Alerts      Send alert email directly
   │
   ▼
Merge NWS alert data
   │
   ▼
Send alert email with NWS context
```

---

## Ice Risk Evaluation Logic

The core detection runs in a JavaScript Code node with the following thresholds (configurable):

```javascript
const PRECIP_WINDOW_HOURS = 6;   // how far back to check for precipitation
const FREEZE_F = 32;              // freezing point in Fahrenheit
const MAX_SNOW_CM = 0.5;          // max snowfall to still consider ice (not snow) risk
```

A property triggers an alert when:
1. `precipInWindow` — precipitation detected in the past 6 hours
2. `freezeCondition` — current temp ≤ 32°F OR overnight low ≤ 32°F
3. `iceOnlyCondition` — snowfall in window ≤ 0.5cm (liquid freezing, not heavy snow)

---

## Tech Stack

| Layer | Tool |
|---|---|
| Workflow automation | n8n |
| Property data | Google Sheets |
| Geocoding | OpenStreetMap Nominatim API (free) |
| Weather data | Open-Meteo API (free, no key required) |
| Weather alerts | National Weather Service API (US only, free) |
| Alert delivery | Gmail |

---

## Google Sheets Structure

Your property tracking sheet needs these columns:

| Column | Description |
|---|---|
| `Property_ID` | Unique identifier |
| `Property_Name` | Display name |
| `Street_Address` | Full street address |
| `City` | City |
| `State` | State/region |
| `Country` | Country |
| `ZIP_Code` | Postal code |
| `Latitude` | Auto-populated if blank |
| `Longitude` | Auto-populated if blank |
| `Notes` | Optional notes |

If Latitude/Longitude are empty, the workflow resolves them automatically via OpenStreetMap and writes them back to the sheet.

---

## Setup

### Prerequisites
- n8n instance
- Google Sheets with property data
- Gmail account for alerts

### Credentials to configure in n8n
```
- Google Sheets OAuth2
- Gmail OAuth2
```

### Configuration
1. Update the Google Sheets node to point to your property sheet
2. Update the `Send alert` Gmail node's `sendTo` field with your alert email
3. Set the Schedule Trigger cron expression to your preferred run times
4. Optionally disable the Schedule Trigger and use manual trigger only for testing

### APIs Used (no keys required)
- **Open-Meteo**: `https://api.open-meteo.com/v1/forecast` — free, no API key
- **OpenStreetMap Nominatim**: `https://nominatim.openstreetmap.org/search` — free, add a User-Agent header with your email as required by their terms
- **NWS Alerts**: `https://api.weather.gov/alerts/active` — free, US properties only

---

## Alert Email Format

When ice risk is detected, the alert email includes:

- Property name and ID
- Recent precipitation status (last 6h)
- Current temperature (°F)
- Overnight low (°F)
- Max snowfall in window (cm)
- Recommended action: salt-only application
- NWS active alerts (headline + description) if property is in the US

---

## Key Design Decisions

**Automatic geocoding fallback** — if coordinates are missing from the sheet, the workflow resolves them via Nominatim and writes them back. Properties don't need to be pre-geocoded.

**NWS US bounds check** — NWS only covers the contiguous United States (lat 24.4–49.4, lon -124.8 to -66.9). Properties outside this range skip NWS and receive Open-Meteo data only.

**Ice vs snow distinction** — the `MAX_SNOW_CM` threshold ensures the system detects freezing rain and light sleet (the dangerous invisible kind) rather than triggering on heavy snowstorms where the hazard is obvious.

---
