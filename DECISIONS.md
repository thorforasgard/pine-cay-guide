# Decisions — Pine Cay Guide

## 2026-04-01 — Single-File Architecture
**Context:** Needed offline-capable island guide for a private island with limited connectivity
**Decision:** Single HTML file with all data inlined (no framework, no build, no server)
**Rationale:** Offline-first is a hard requirement. 227KB single file loads once and works without network. No app store friction.
**Alternatives considered:**
- React/Next.js — Overkill for a single page. Build complexity for no benefit.
- Native app (Capacitor) — App store friction, 2-week review cycle. URL is instant.
- Multiple files + Service Worker — Adds complexity, benefits minimal for 227KB.
**Output:** index.html (self-contained)

## 2026-04-01 — OpenStreetMap as Primary Data Source
**Context:** Needed detailed island map with roads, buildings, trails
**Decision:** OSM Overpass API for all geographic data, baked in at build time
**Rationale:** Pine Cay is mapped in extraordinary detail on OSM (107 roads, 100 buildings, 46 named houses). Free, open, no API key.
**Alternatives considered:**
- Google Maps embed — Requires API key, no offline, cost at scale
- Mapbox — Better tiles but API key + cost. Overkill.
- Hand-drawn map — Beautiful but not navigable
**Output:** osm-data.json → pine-cay.geojson + road-graph.json → inlined in HTML

## 2026-04-01 — Client-Side Routing (Dijkstra)
**Context:** Turn-by-turn navigation on island roads
**Decision:** In-browser Dijkstra on pre-built road graph
**Rationale:** 1,270 nodes is trivial for Dijkstra (< 10ms). Works offline. No routing server needed.
**Alternatives considered:**
- OSRM/Valhalla server — Way too heavy for 107 roads
- A* — Marginal speedup not needed at this graph size
- Simple nearest-road-follow — Wouldn't find shortest path
**Output:** Routing engine + turn-by-turn generator in ~150 lines of JS

## 2026-04-01 — Lunar Tide Approximation
**Context:** No free high-quality tide API for TCI specifically
**Decision:** Semi-diurnal calculation using lunar cycle math
**Rationale:** TCI has predictable semi-diurnal tides. Approximate is better than nothing for "should I go snorkeling now?" Clearly labeled as approximate.
**Alternatives considered:**
- NOAA CO-OPS — No TCI stations
- WorldTides API — Paid ($5/month)
- No tides at all — Misses a key feature (reef ball drift timing)
**Output:** Client-side phase calculation from lunar day reference

## 2026-04-01 — Process Mistake: Code Before Architecture
**Context:** Ken asked for an app idea, I jumped straight to building
**Decision (retroactive):** Should have written ARCHITECTURE.md + gotten approval before coding
**Lesson:** Even for quick-build vacation projects, the arch plan takes 10 minutes and prevents rework. I skipped it because I was excited. That's not a good reason.
**Fix:** This document exists now. Future features (boogie board, etc.) get documented here before implementation.

---
*Updated: 2026-04-01 by Thor*
