# Pine Cay Pocket Guide — Architecture

## Purpose
Offline-first mobile web app for navigating Pine Cay island (Turks & Caicos). GPS turn-by-turn on golf cart roads, live weather/marine conditions, wildlife calendar, and glow worm timing.

## Target User
Guests on Pine Cay — private island with 38 homeowners, limited cell service, no Google Maps coverage. Accessed on phone browser.

## Architecture Decision: Single-File PWA

**Decision:** Everything in one `index.html` — no build step, no framework, no server.

**Rationale:**
- Offline-first is critical (spotty island connectivity)
- No app store deployment friction (just a URL)
- 227KB total — loads once, works forever
- Zero infrastructure to maintain
- Can be saved as iOS/Android home screen app

**Tradeoffs:**
- No code splitting (acceptable at 227KB)
- GeoJSON + road graph inlined (184KB of data) — makes the file large but eliminates network dependencies
- No SSR/hydration — pure client-side (fine for this use case)

## Data Sources

| Data | Source | Update Frequency | Offline? |
|------|--------|------------------|----------|
| Island map (roads, buildings, coastline) | OpenStreetMap Overpass API | Baked in at build time | ✅ Yes |
| Road graph (routing) | Derived from OSM ways | Baked in at build time | ✅ Yes |
| POIs (named locations) | Manually curated + OSM | Baked in | ✅ Yes |
| Weather (temp, wind) | Open-Meteo API | Live (hourly) | ❌ Graceful fallback |
| Marine (waves, swell) | Open-Meteo Marine API | Live (hourly) | ❌ Graceful fallback |
| Tides | Calculated (lunar semi-diurnal model) | Client-side math | ✅ Yes |
| Glow worm dates | Calculated (full moon + 5 days) | Client-side math | ✅ Yes |
| Boogie board conditions | Open-Meteo Marine API | Live (hourly) | ❌ Graceful fallback |
| User pins | localStorage | Per-device | ✅ Yes |

## Components

```
┌─────────────────────────────────────────────┐
│                  index.html                  │
│                                              │
│  ┌─────────────┐  ┌──────────────────────┐  │
│  │  Leaflet Map │  │   Bottom Panel       │  │
│  │  - Satellite │  │   - 🧭 Navigate      │  │
│  │  - OSM tiles │  │   - 🌊 Tides         │  │
│  │  - GeoJSON   │  │   - 🏄 Surf/Boogie   │  │
│  │  - Route line│  │   - ✨ Glow Worm      │  │
│  │  - GPS dot   │  │   - 🐢 Wildlife      │  │
│  │  - POI icons │  │   - 💨 Wind           │  │
│  └─────────────┘  └──────────────────────┘  │
│                                              │
│  ┌─────────────┐  ┌──────────────────────┐  │
│  │  Nav Banner  │  │   Routing Engine     │  │
│  │  (turn-by-  │  │   - Road graph       │  │
│  │   turn)     │  │   - Dijkstra         │  │
│  └─────────────┘  │   - Turn detection   │  │
│                    └──────────────────────┘  │
│                                              │
│  ┌─────────────────────────────────────────┐ │
│  │  GPS Module                             │ │
│  │  - watchPosition (high accuracy)        │ │
│  │  - Blue dot + accuracy circle           │ │
│  │  - Auto-advance nav steps               │ │
│  └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

## Routing Engine

**Algorithm:** Dijkstra's shortest path
**Graph:** 107 roads, 1,270 nodes (bidirectional — all Pine Cay roads are two-way)
**Node key:** `"lat,lon"` string (6 decimal places)
**Turn detection:** Bearing delta between consecutive segments
  - < 30° = straight
  - 30-75° = bear right/left
  - 75-120° = turn right/left
  - 120-165° = sharp right/left
  - 165-195° = U-turn
**Step merging:** Consecutive "straight" segments < 30m get merged
**Speed assumption:** 15 km/h (golf cart)

## GPS Integration

- `navigator.geolocation.watchPosition` with `enableHighAccuracy: true`
- Position updates every 2s (maximumAge: 2000)
- Nav step advancement: advance when within 15m of step waypoint
- Arrival detection: within 20m of destination
- Map auto-centers on GPS during active navigation

## Tide Model

Semi-diurnal approximation using lunar cycle:
- Tidal period: 12.42 hours (lunarDay / 2)
- Reference high tide: 2026-01-01T06:00Z
- Phase calculation: `(hoursSinceRef % period) / period`
- TCI has near-semi-diurnal pattern — this is approximate but useful

**Note:** Not a replacement for official tide tables. Labeled as approximate.

## Surf/Boogie Board Conditions

**Source:** Open-Meteo Marine API — wave height, swell height, swell period, swell direction
**Rating system:**
- 🟢 Good: Wave height 0.5-1.5m, swell period > 8s
- 🟡 Fair: Wave height 0.3-0.5m or > 1.5m
- 🔴 Flat/Rough: < 0.3m (too flat) or > 2.5m (too rough)
**Display:** Hourly forecast with conditions rating + best window highlight

## Security Considerations

- No authentication (public guide)
- No PII collected
- localStorage only stores pin names + coordinates
- All API calls are to public free endpoints (Open-Meteo — no API key)
- No cookies, no tracking, no analytics

## Deployment

- GitHub Pages: `thorforasgard.github.io/pine-cay-guide`
- Single static file — no server, no build, no CI
- To update: edit index.html → push to main → auto-deploys

## Future Enhancements (Not in v1)

- Service Worker for full offline caching (including map tiles)
- Offline tile pack (download satellite tiles for island bbox)
- Share pins between devices (QR code or URL params)
- Trail difficulty ratings
- Photo attachment to pins
- Boat route to The Aquarium (water path, not road)

## Build Process

```bash
# 1. Fetch latest OSM data
curl -s "https://overpass-api.de/api/interpreter" \
  --data-urlencode 'data=[out:json][timeout:60];(way(21.855,-72.115,21.885,-72.075););out body;>;out skel qt;' \
  > osm-data.json

# 2. Generate GeoJSON + road graph
python3 scripts/process-osm.py  # (TODO: extract from inline)

# 3. Inject into HTML
python3 scripts/build.py  # (TODO: extract from inline)

# 4. Deploy
git add index.html && git commit -m "update" && git push
```

---
*Created: 2026-04-01 by Thor*
*This document should have been written BEFORE the code. Lesson noted.*
