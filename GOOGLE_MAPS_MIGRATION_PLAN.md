# Lulini Maps Audit and Google Maps Platform Migration Plan

**Scope:** `Cxvarisorka/lulini.ge` at commit `c7cb1bc` (master, 2026-09-07). Server (`server/`), passenger app (`mobile/`), driver app (`mobile-driver/`), manager app (`mobile-manager/`), admin SPA (`client/`), public web (`web/`).
**Method:** every claim below was verified by reading the code on that commit. File references use `path:line`. Nothing was inferred from file names alone. No code was changed.
**Note on repositories:** the session was opened against `Cxvarisorka/lulini`, which contains only a two-file HTML exercise. The taxi platform lives in `Cxvarisorka/lulini.ge`; that repo was attached read-only and is what this document describes. This document is committed to `lulini` because that is the branch this session was authorised to push to.

---

## 0. Executive summary

**Where the system stands today.** The platform is not "Google with some extras". It is a five-provider system with Google in a minority role:

| Concern | Primary today | Fallback(s) | Google role |
|---|---|---|---|
| Map rendering (all 3 mobile apps) | **Mapbox GL** (`@rnmapbox/maps 10.3.0`) | OSM raster tiles in Expo Go (manager) | none |
| Map rendering (admin SPA, public web) | **Google Maps JavaScript API** | – | primary |
| Address autocomplete | Mongo "known places" + **Nominatim** (public OSM) | **Google Places (legacy)** only when Nominatim is sparse or the query "looks like a POI" | secondary |
| Forward / reverse geocoding | **Nominatim** (public OSM) | Google Geocoding | fallback |
| Quote / fare routing and matrices | **OSRM** (public demo server) | Google Directions / Distance Matrix (legacy) | fallback |
| Turn-by-turn navigation routes | **Mapbox Directions** (driving-traffic) | Google Directions → OSRM | second fallback |
| Dispatch ETA (top 3 candidates) | **Google Distance Matrix (legacy)** | 30 km/h straight-line | primary |
| Pickup road snap | **Google Roads** (result never read) | – | primary, wasted |

The good news: the server already fronts every web-service provider behind a unified contract (`server/services/{routing,geocoding,autocomplete}.service.js`, `server/providers/*.provider.js`) with a Redis cache, and no mobile app embeds a Google key or calls a Google web service directly. Phases 2 to 5 of the migration are therefore server-side provider swaps behind stable interfaces.

The hard news: **the mobile map renderer is Mapbox, not Google**, and the codebase has already changed renderer twice this year (MapLibre → react-native-maps in March 2026, react-native-maps → Mapbox in April 2026, per git history). Moving to the Google Maps SDK is a third renderer swap across three apps with three divergent copies of the wrapper layer. It is also **mandatory, not optional**, for a Google-only end state: Google Maps Platform terms forbid displaying Places and Geocoding content on a non-Google map, so Places (New) and Geocoding cannot go live on Mapbox tiles. Phase 6 (rendering) must ship with or before Phases 2 and 3 reach production.

**The ten findings that matter most** (full detail in Section 9 onward):

1. **Fare can be set by the client** by sending pickup/dropoff coordinates as strings: the validator does not coerce, the server quote silently returns `null`, and the client's price is persisted (Critical, `server/middlewares/validators.js:28-33`, `server/services/pricing.service.js:120-123`, `server/controllers/ride.controller.js:220-227,324`).
2. **Sequential ETA dispatch is defeated** by a parallel queue publish that broadcasts every `ride:request` to `drivers:all` immediately (High, `server/controllers/ride.controller.js:717-726`, `server/workers/rideEventWorker.js:51-55`).
3. **Public OSRM and public Nominatim are the production defaults** for routing and geocoding, in breach of both projects' usage policies, and the passenger app also calls the public OSRM demo directly (Critical, `server/providers/osrm.provider.js:17`, `server/providers/nominatim.provider.js:22`, `mobile/src/services/googleMaps.js:229-257`).
4. **The passenger app reverse-geocodes every 5 seconds while idle** on the search screen and **polls a driver→pickup route every 15 seconds** while waiting. Today those calls mostly hit Nominatim and OSRM; after migration they become the two largest Google line items unless fixed first (High, `mobile/src/screens/TaxiScreen.js:997-1020,1092-1095`).
5. **Every driver GPS fix is uploaded twice** (foreground PATCH and background batch), the batch arrives late with a server-side timestamp, and the passenger's car marker jumps backwards (High, `mobile-driver/src/context/LocationContext.js:453-458`, `mobile-driver/src/services/backgroundLocation.js:201-269`).
6. **Server-side in-ride ETA does not exist.** The code that would compute it (`processPositionUpdate`) has no caller, its geofences are created without `active:true`, and passengers compute ETA client-side at 30 km/h straight-line (High, `server/services/navigation.service.js:220-297`).
7. **Stops are never priced and never drawn** in the shared ride route: both the booking quote and `activeRoute` call routing without waypoints (High, `server/services/pricing.service.js:128`, `server/controllers/ride.controller.js:1337-1341`).
8. **Final fare is whatever the driver app sends within ±15 %** of the quote; waiting fees are computed but never charged (High, `server/controllers/ride.controller.js:1412,1459-1472`).
9. **Google Place Details content is stored in Mongo indefinitely**, and Georgian text from any passenger's ride body is upserted as a globally ranked "known place" (High, ToS and abuse, `server/services/autocomplete.service.js:284-290`, `server/controllers/ride.controller.js:67-77,371-373`).
10. **Race conditions on the pickup pin**: adjust-mode reverse geocoding has no request ordering, and the confirm button commits the previous position's address with the new coordinates (High, `mobile/src/screens/TaxiScreen.js:2504-2508,2716-2724`). The correct pattern already exists in `mobile/src/screens/SavedPlaceEditorScreen.js:144-156`.

**Recommended shape of the migration.** Nine phases, ordered so that nothing user-facing regresses and Google spend is capped before it can grow:

1. Cleanup and hard bug fixes (2 to 3 weeks): pricing bypass, dispatch broadcast, double GPS upload, dead code, validators, shared map wrapper package.
2. Places API (New) behind the existing `/maps/autocomplete` and `/maps/place-details` proxy; Place ID plumbing through ride, favorite, and recent models.
3. Geocoding API behind `/maps/geocode`; client-side geocode throttling and request ordering.
4. Routes API `computeRoutes` behind `routing.getRoute` and `getNavRoute`; waypoints everywhere; server-authoritative quote and final fare.
5. `computeRouteMatrix` for dispatch and quote-screen nearest-driver ETA; server-pushed in-ride ETA on a movement/time trigger.
6. Google Maps SDK rendering in all three apps via one shared wrapper package; camera state machine; marker/polyline render discipline.
7. Real-time navigation: single GPS pipeline with device timestamps, single navigation engine, off-route detection with accuracy and heading, route-snapped smooth marker animation.
8. Cost and performance: Redis-backed limiters and throttles, negative caches, deduplication, per-SKU quotas, cost dashboard.
9. Remove Mapbox, OSRM, Nominatim, Roads, legacy Google endpoints, and their tokens.

Estimated Google spend in the target architecture is roughly USD 0.12 to 0.16 per ride at list price before free tier (Section 19.16). The current architecture, if simply pointed at Google without the Phase 1/3/5 fixes, would cost several times that, dominated by idle reverse geocoding and route polling.

---

## How to read this document

Sections 1 to 8 describe what exists. Sections 9 to 14 list findings in the required structure (Issue / Severity / Location / Current implementation / Problem / User impact / Performance impact / Google API cost impact / Recommended solution / Migration phase). Sections 15 to 18 draw the client/server line. Section 19 is the target architecture, Section 20 the phases, Sections 21 to 23 the file-level work list, complexity, and order.

Severity scale: **Critical** = money, legal, or total-outage risk now; **High** = wrong results or user-visible failures for many rides, or a hard migration blocker; **Medium** = degraded experience, waste, or maintainability risk; **Low** = hygiene.

Phase codes used in every issue: P1 cleanup, P2 Places, P3 Geocoding, P4 Routes, P5 ETA and dispatch, P6 rendering, P7 real-time navigation, P8 performance and cost, P9 remove legacy.

---

# Part A. The system as it exists today

## 1. Current architecture

### 1.1 Component map

```
 Passenger app (mobile/)          Driver app (mobile-driver/)        Manager app (mobile-manager/)
 ┌──────────────────────┐        ┌──────────────────────────┐       ┌──────────────────────────┐
 │ TaxiScreen (3,725 l) │        │ HomeScreen (2,260 l)     │       │ WialonScreen / Fleet /   │
 │ LocationSearchSheet  │        │ NavigationScreen         │       │ OpsMap (feature-flagged) │
 │ RideStatusSheet      │        │ RideDetailScreen         │       │                          │
 │ components/map/*  ───┼─ Mapbox GL ─┤ components/map/*     │       │ components/map/*  ── Mapbox
 │ services/googleMaps  │        │ services/directions      │       │ (+ OSM raster in Expo Go)│
 │ (thin proxy client)  │        │ navigation/* (engine)    │       │ polls /ops, /wialon      │
 └─────────┬────────────┘        │ context/LocationContext  │       └────────────┬─────────────┘
           │ HTTPS + Socket.IO   │ services/backgroundLoc.  │                    │ HTTPS
           │                     └───────────┬──────────────┘                    │
           ▼                                 ▼ HTTPS (no socket location)        ▼
 ┌───────────────────────────────────────────────────────────────────────────────────────────┐
 │ server/ (Express + Socket.IO, PM2 cluster, Redis, MongoDB, BullMQ)                        │
 │                                                                                           │
 │  routers/maps.router  ──► controllers/maps.controller ──► services/routing.service ──┐    │
 │  (protect + 30 rpm/user,  /directions /distance-matrix   services/geocoding.service │    │
 │   in-process Map)         /geocode /autocomplete         services/autocomplete.svc  │    │
 │                           /place-details /snap-to-road   services/places.service    │    │
 │  routers/locations.router (no rate limit) ──► same services                         │    │
 │  controllers/ride.controller ──► pricing.service ──► routing.service                │    │
 │                             ──► etaDispatch.service (own Google DM client)          │    │
 │                             ──► roadSnap.service (own Google Roads client)          │    │
 │  controllers/driver.controller (PATCH /drivers/location, /location/batch)           │    │
 │        └─► driverLocation.service (Redis GEO + hash, 120 s TTL, 30 s Mongo flush)   │    │
 │        └─► io.to(user:{id}).emit('driver:locationUpdate')                           │    │
 │  socket/handlers/navigation.js ──► navigation.service (nav:session_start/off_route) │    │
 │  services/cache.service (Redis, single key policy, TTL table)                       │    │
 │                                                                                     ▼    │
 │  providers/  osrm.provider ─► router.project-osrm.org (default)                          │
 │              nominatim.provider ─► nominatim.openstreetmap.org (default, 1 rps queue)    │
 │              mapbox.provider ─► api.mapbox.com/directions/v5 (driving-traffic)           │
 │              google.provider ─► maps.googleapis.com (legacy Directions, DM, Geocoding,   │
 │                                  Places Autocomplete/Details), roads.googleapis.com      │
 │              wialon.provider ─► Wialon geocoder (fleet panel only)                       │
 └───────────────────────────────────────────────────────────────────────────────────────────┘
           ▲                                          ▲
 client/ (Vite admin SPA)                     web/ (Next.js public site)
 Google Maps JS API + Places JS + Directions JS   Google Maps JS API + Directions JS
 + direct browser calls to public Nominatim       (shared-ride tracker)
```

### 1.2 Server entry points related to maps

| Endpoint / event | Handler | Downstream | Auth / limit |
|---|---|---|---|
| `GET /api(/v1)/maps/autocomplete` | `maps.controller.js:172-190` | `autocomplete.service.getPredictions` | `protect` + 30 rpm/user (`maps.router.js:10-53`), limiter is per PM2 worker |
| `GET /maps/place-details` | `maps.controller.js:194-200` | `autocomplete.service.resolvePrediction` | same |
| `GET /maps/geocode` (address or latlng) | `maps.controller.js:144-168` | `geocoding.service` | same |
| `GET /maps/directions` (`traffic`, `steps`, `waypoints`) | `maps.controller.js:44-95` | `routing.getNavRoute` or `getRoute` | same; waypoints unbounded |
| `GET /maps/distance-matrix` | `maps.controller.js:99-139` | `routing.getMatrix` | same; N×M unbounded |
| `GET /maps/snap-to-road` | `maps.controller.js:204-227` | `google.provider.snapToRoads` | same; no cache |
| `GET /locations/search`, `/reverse`, `/recent`, `/nearby-popular` | `locations.router.js:35-99` | geocoding, recentLocations, places | `protect` only, **no rate limit** |
| `GET /rides/quote` | `ride.controller.js:2971-3088` | Redis GEO + 2× `routing.getRoute` | protect |
| `POST /rides` | `ride.controller.js:148-755` | `pricing.computeServerQuote` → `routing.getRoute`; `etaDispatch.findDriversByETA`; `roadSnap` | protect + validators |
| `PATCH /rides/:id/start` | `ride.controller.js:1231-1406` | `routing.getNavRoute` → `Ride.activeRoute` | driver |
| `PATCH /rides/:id/complete` | `ride.controller.js:1411-1700` | fare band check on client `fare` | driver |
| `PATCH /drivers/location`, `POST /drivers/location/batch` | `driver.controller.js:550-713, 1195-1325` | `driverLocation.service` + socket emit | 30 rpm **per IP** |
| `POST /rides/:id/locations/batch` | `ride.controller.js:2321-2384` | `Ride.routePoints` | driver |
| socket `nav:session_start`, `nav:phase_change`, `nav:off_route` | `socket/handlers/navigation.js:37-115` | `navigation.service` → `routing.getNavRoute` | driver sockets only |
| `GET /drivers/nearby` | `driver.controller.js:1115-1190` | Redis GEO / Mongo `$near` | protect |

### 1.3 Data stores touched by map features

- **Redis:** route/matrix/geocode/autocomplete/place-details caches (`cache.service.js:21-33`), ETA cache `eta:*` (180 s), driver GEO sets and `driver:loc:{id}` hashes (120 s), `user:recent:{id}` lists (no TTL), nav session hashes, offer locks, surge grid.
- **MongoDB:** `rides` (pickup/dropoff/stops as `{lat,lng,address}`, `activeRoute` with decoded polyline + steps, `routePoints`), `drivers.location` (2dsphere, flushed from Redis every 30 s), `places` (warm cache of resolved places, no TTL), `favoritelocations`, `navigationsessions`, `geofences` (never cleaned), `locations` (dead model).
- **Device:** passenger SecureStore ride snapshot; driver AsyncStorage ride state and location buffer, SecureStore retry queue.

## 2. Current providers

| Provider | Where used | Why | What it provides | Keep? | Google replacement | Complexity | Risk of removal |
|---|---|---|---|---|---|---|---|
| **Mapbox GL SDK** (`@rnmapbox/maps 10.3.0`) | `mobile/src/components/map/*`, `mobile-driver/src/components/map/*`, `mobile-manager/src/components/map/*`; raw `Mapbox.MarkerView` in `mobile/src/screens/TaxiScreen.js:2997,3008,3074`, `RideDetailScreen.js:247-278` | Chosen April 2026 for marker stability and house-number styling after react-native-maps problems | Tiles, style layers, `ShapeSource`/`SymbolLayer` markers, `LineLayer` routes, house-number boost, dark style | **No** | Maps SDK for Android/iOS via `react-native-maps` `PROVIDER_GOOGLE`; Cloud-based map styling `mapId` for light/dark | **High** (three apps, ~20 wrapper files, 200+ line diffs between copies) | Lose house-number emphasis layer, `TrafficPolyline` per-segment colouring source, `SurgeHeatmap` `FillLayer` (becomes `Polygon`s). Marker animation must be rewritten. |
| **Mapbox Directions** (`driving-traffic`) | `server/providers/mapbox.provider.js`; primary in `routing.getNavRoute` (`routing.service.js:106`) | Only traffic-aware provider with per-segment congestion and posted speed limits | Nav routes with `congestion[]`, `maxspeed`, steps | No | Routes API `computeRoutes` with `routingPreference: TRAFFIC_AWARE_OPTIMAL` and `extraComputations: ["TRAFFIC_ON_POLYLINE"]` → `speedReadingIntervals` for colouring | Medium | **Posted speed limits are not available** from Routes; `SpeedLimitSign` (`mobile-driver/src/screens/NavigationScreen.js:392-396`) must be removed or gated. Congestion mapping changes from per-segment strings to interval ranges. |
| **OSRM** (public demo default) | `server/providers/osrm.provider.js:17`; primary in `getRoute`, `getMatrix`; last resort in `getNavRoute`; directly from passenger app `mobile/src/services/googleMaps.js:229-257`; dead `matching.service.js` | Free | Free-flow routes and tables | No | `computeRoutes` (Essentials, `TRAFFIC_UNAWARE`) for cached quotes; `computeRouteMatrix` for matrices | Medium | None functional. Public demo forbids production use; no traffic; steps only from first leg (`osrm.provider.js:72`). |
| **Nominatim** (public OSM default) | `server/providers/nominatim.provider.js:22`; primary in `autocomplete.service.js:53-77,202,223` and `geocoding.service.js:51,93`; directly from admin SPA `client/src/pages/admin/AdminCreateRide.jsx:46-48`; `nominatim/docker-compose.yml` (self-host, "not wired") | Free, returns coordinates inline with predictions | Address search and reverse geocoding for Georgian streets | No | Places Autocomplete (New) with `includedRegionCodes:["ge"]`, `locationBias`; Geocoding API | Low to Medium (provider deletion + response shape) | OSM policy explicitly forbids autocomplete against the public instance; 1 rps global queue. Georgian house-number coverage on Google must be validated in Kutaisi before cut-over (Section 3.4). |
| **Google Places (legacy)** | `google.provider.js:220-270` | POI/brand search | Autocomplete predictions, Place Details | Replace with **New** | `places:autocomplete`, `places/{id}` with field masks, session tokens (already plumbed) | Medium | Legacy endpoints are deprecated; response shapes differ. |
| **Google Geocoding** | `google.provider.js:190-216` | Fallback only | Forward/reverse | **Yes** (becomes primary) | Same API, add `result_type`, `location_type`, single `language` | Low | – |
| **Google Directions / Distance Matrix (legacy)** | `google.provider.js:80-169`; `etaDispatch.service.js:148-155` (own client) | Fallback routing; primary dispatch ETA | Routes, matrices | Replace | Routes API `computeRoutes`, `computeRouteMatrix` | Medium | Legacy SKUs; `departure_time=now` on every Directions call bills Advanced tier for cached free-flow quotes. |
| **Google Roads** (`snapToRoads`) | `google.provider.js:274-290`; `roadSnap.service.js:24-45`; proxy `/maps/snap-to-road`; driver `roadSnapping.js` (no importer) | Pickup precision | Snapped points | **No** | Nothing needed: `pickup.snappedRoadCoords` is never read; on-device route projection already exists (`mobile-driver/src/navigation/routeProjection.js`) | Trivial | None. |
| **Wialon geocoder** | `wialon.provider.js:482-524`, `wialon.service.js:126-162` | Fleet panel labels | Batch reverse geocode | Out of scope for ride-hailing; can move to Geocoding API later | Geocoding API | Low | Cost: batch becomes per-point (cache exists). |
| **Google Maps JavaScript API** | `client/` (7 duplicated loaders), `web/src/components/ride/SharedRideView.jsx` | Admin and share pages | Map, Directions JS, Places JS | Yes | Same, consolidated loader, `language=ka`, `region=GE`; move Places/Directions behind the server proxy | Low | – |
| **Apple/Yandex/Waze/Google Maps app deep links** | `mobile-driver/src/context/MapContext.js:15-65` | Driver preference for external navigation (default is `'google'` = leave the app) | Deep links | Product decision | – | – | See Section 12. |
| **Open-Meteo** | `mobile-driver/src/services/weather.js:36-38` | Weather badge | Keyless | Yes | – | – | – |

**What blocks a clean Google-only end state**

- Mapbox rendering in the three mobile apps (ToS blocker for Places/Geocoding display, see Section 9.13).
- `SpeedLimitSign` and the `maxspeed` step field (no Google equivalent at reasonable cost).
- `TrafficPolyline` colouring keyed off Mapbox `congestion[]` per segment.
- The house-number boost layer and Mapbox style state machine (`mobile-manager/src/components/map/MapViewWrapper.js:228,277-330,467-500`, similar in the other apps).
- The Nominatim-shaped reverse-geocode adapter on the passenger client (`mobile/src/services/googleMaps.js:162-198`) expects `components.road/house_number/suburb`.
- `osm:` and `manual:` canonical IDs in the `places` collection and in prediction payloads.

## 3. Current autocomplete flow

### 3.1 Passenger app

```
keystroke ──► LocationSearchSheet.handleTextChange (:158-174)
   │  min 3 chars (:128), 500 ms debounce (:152), searchIdRef stale-drop (:133-148), no AbortController
   ▼
googleMaps.searchPlaces(query, userLocation, sessionToken)   (mobile/src/services/googleMaps.js:259-293)
   │  30-entry / 30-min LRU by NFC-lowercased query; exact hit → no network;
   │  prefix hit → local substring filter if ≥3 remain (:268-277)
   ▼
GET /maps/autocomplete?input&lat&lng&sessionToken          (no language, countryCode, radius sent)
   │
   ▼
autocomplete.service.getPredictions (server/services/autocomplete.service.js:171-245)
   │  Redis autocomplete:merged:{hash(query|bias@0.1°|lang)}  24 h / 5 min negative
   │  Mongo Place text search (4)  ──► kind:'known'
   │  MAPS_TIERED_AUTOCOMPLETE=true (.env.example:107):
   │     Nominatim /search (limit 8, countrycodes=ge, accept-language=ka, viewbox bias)
   │     Google legacy /place/autocomplete/json only if mongo+nominatim < 4 OR looksLikePOI(input)
   │  merge(): known → address → poi, dedupe on lower(mainText|secondaryText), slice 8
   ▼
selection ──► if no coords (Google POI): GET /maps/place-details?placeId&sessionToken
              resolvePrediction: Redis placeDetails 30 d → Mongo Place → Google legacy Details
              (fields place_id,formatted_address,geometry,address_components,name,types)
              → written to Redis 30 d and Mongo Place (no TTL)
   ──► session token rotated (LocationSearchSheet.js:226)
   ──► onPickupSelect / onDestinationSelect / onStopSelect → TaxiScreen state
```

Recent and saved places are loaded on every sheet mount (`LocationSearchSheet.js:324-354`) and shown when there are no suggestions; a recent entry without coordinates does nothing when tapped (`TaxiScreen.js:2065-2071`). The empty state also queries `/locations/nearby-popular` (Mongo `$near`, free). The list footer always reads "Powered by OpenStreetMap" even when Google predictions are present (`LocationSearchSheet.js:740-747`).

### 3.2 Admin SPA

`client/src/pages/admin/AdminCreateRide.jsx` calls the Places JavaScript `AutocompleteService.getPlacePredictions` **without a session token** on a 400 ms debounce (`:88-96`), `PlacesService.getDetails` **without a session token** (`:121-123`), and falls back to **public Nominatim from the browser** (`:46-48`). A session-aware hook exists (`client/src/hooks/usePlacesAutocomplete.js`) and is not used there.

### 3.3 Driver and manager apps

Neither calls autocomplete. The manager app defines `mapsAPI.autocomplete/placeDetails` (`mobile-manager/src/services/api.js:231-233`) with no callers; its dispatch screen is a placeholder.

### 3.4 Observed problems (detail in Section 9)

- Georgian address queries are usually one or two tokens with no digits, which `looksLikePOI` (`autocomplete.service.js:155-164`) classifies as POI, so Google is called on most misses anyway; the tiering saves less than intended and adds serial latency (Nominatim then Google).
- No location bias unless the client sends `lat/lng`; the manager app never does; Nominatim `viewbox` is `bounded=0` (bias only). Country-wide ranking favours Tbilisi for common street names.
- Cross-provider dedupe compares formatted text, so the same street appears as a Nominatim row, a Google row, and a `manual:` "known" row.
- Client prefix filtering hides results a longer query would surface (house numbers, lower-ranked POIs).
- The autocomplete effect depends on `userLocation`, which TaxiScreen passes as a fresh object every GPS tick (`TaxiScreen.js:2895`), restarting the debounce.
- The UI language is never forwarded; server defaults to `ka`.
- Public Nominatim as an autocomplete backend is a policy breach and a single 1 rps global queue with no cap (`nominatim.provider.js:24,39-54`).
- Session tokens are generated per sheet mount and rotated only after a selection resolved through place-details, which is correct for Google's billing model.

## 4. Current geocoding flow

Server: `geocoding.service.js`. Forward: Redis `geo:fwd:nominatim:*` 24 h → Nominatim `/search` → if empty or error → Redis `geo:fwd:google:*` → Google Geocoding (`components=country:GE`, `language=ka`). Reverse: Redis `geo:rev:nominatim:{lat4},{lng4}:{lang}` 24 h → Nominatim `/reverse` → Google `/geocode/json?latlng` (no `result_type`, takes `results[0]`). Empty and null results are never cached. `/locations/search|reverse` pass `language='ka,en'`, which Google does not accept and which fragments the cache key.

Passenger client call sites for reverse geocoding (all `GET /maps/geocode?latlng=` at full float precision, `mobile/src/services/googleMaps.js:358-371`):

| Trigger | Location | Rate |
|---|---|---|
| GPS watch in LOCATION_SEARCH, moved >20 m **or** >5 s | `TaxiScreen.js:997-1020` | up to every 5 s while stationary with GPS jitter |
| `refreshLocation` (mount, foreground, back to search, centre button) | `TaxiScreen.js:1841` | 1 to 2 per call |
| permission grant cached fix | `TaxiScreen.js:1914` | 1 (so up to 3 on cold start) |
| pickup / dropoff drag end | `TaxiScreen.js:2579,2594` | 1, unguarded |
| pan-to-adjust, 400 ms debounce | `TaxiScreen.js:2716-2724` | 1 per settle, unguarded |
| tap-to-select on map | `TaxiScreen.js:2768` | 1 per tap |
| saved place editor pan, 500 ms debounce | `SavedPlaceEditorScreen.js:144-167` | guarded by `reverseIdRef` (correct) |

Driver app: one OS reverse geocode after permission grant whose result nobody reads (`mobile-driver/src/context/LocationContext.js:332-365`). Forward geocoding on the client: none. Ride history and driver labels use stored address strings; no geocoding.

`onRegionChangeComplete` in the Mapbox wrapper fires for every camera frame where no gesture is active (`mobile/src/components/map/MapViewWrapper.js:253-276`), including programmatic animations and fling inertia. `SavedPlaceEditorScreen` works around this with a time window; `TaxiScreen` does not, so entering adjust mode geocodes the unchanged coordinate and a fling produces per-frame haptics and state updates.

## 5. Current routing flow

| Workflow | Where | Provider chain | Waypoints | Cached |
|---|---|---|---|---|
| Quote screen: nearest driver → pickup, pickup → destination | `ride.controller.js:3039-3052` (`GET /rides/quote`), called per vehicle class by `useRideQuote` | OSRM → Google Directions | no (stops not accepted) | `route:` 300 s |
| Booking quote | `pricing.service.js:128` via `createRide` | same | **no** (stops dropped) | same |
| Client route line + price in RIDE_OPTIONS | `TaxiScreen.js:1955-2034` → `GET /maps/directions?waypoints` → fallback **direct public OSRM** → straight line | OSRM → Google; then client OSRM | yes | client LRU 1 h + server |
| Per-stop leg ETAs | `TaxiScreen.js:1756-1782` | one `/maps/directions` per stop | – | – |
| Driver → pickup while waiting | `TaxiScreen.js:1073-1095`, every 15 s | OSRM → Google | – | keys at 5 decimals never hit |
| Ride start `activeRoute` | `ride.controller.js:1337-1341` | Mapbox → Google → OSRM (`getNavRoute`) | **no** | `route:nav` 90 s |
| Driver nav session (to_pickup / to_dropoff) | `navigation.service.js:132-136` via socket | Mapbox → Google → OSRM | yes (stops) | 90 s, keyed on moving origin |
| Driver app HTTP directions (same leg) | `mobile-driver/src/services/directions.js:67-142` → `/maps/directions?steps=true&traffic=true` | same | no | client LRU 5 min + server |
| Driver reroute | `navigation.service.js:442` **and** client fetch (`useNavigationEngine.js:242-277`) | same, twice | – | – |
| Driver RideDetail header map | `mobile-driver/src/services/googleMaps.js:20-50` | OSRM → Google (free-flow, no timeout) | no | – |
| Admin / share pages | `client/src/components/RouteMap.jsx:57-93`, `AdminRides.jsx:82-96`, `SharedRide.jsx:127-135`, `web/.../SharedRideView.jsx:156-170` | Google Directions JS in the browser | some | none |
| Destination change | **does not exist** (no endpoint) | – | – | – |
| Scheduled rides | quote frozen at booking, broadcast at T-10 min (`ride.controller.js:2872-2963`) | – | – | – |

All providers return a **decoded** coordinate array; nothing transports encoded polylines. `activeRoute` persists the decoded line, steps, and congestion per ride (`ride.model.js:290-298`) and is included in list endpoints.

## 6. Current ETA calculation

| ETA | Source today | Traffic-aware | Refresh |
|---|---|---|---|
| Dispatch ranking (driver → pickup), top 3 candidates | Google Distance Matrix `duration_in_traffic` (`etaDispatch.service.js:148-165`) | yes | per request, 180 s cache on a 111 m grid |
| Dispatch ranking, candidates 4+ and Mongo fallback | `straightLineKm / 30 × 3600` (`etaDispatch.service.js:293`) or `null` | no | – |
| Quote screen driver ETA | OSRM free-flow route from the single nearest driver of the exact class (`ride.controller.js:3041`) | no | per quote call |
| Passenger, after accept | **client**: haversine ÷ 30 km/h (`TaxiScreen.js:1273`), overridden by the 15 s driver-route poll (`:1082`) | no | every location event / 15 s |
| Passenger, in ride | `activeRoute.durationSeconds` once at `ride:started` (`TaxiScreen.js:1448-1451`), then frozen (`:1054`) | yes, once | never |
| Passenger "driver approaching" | server haversine ÷ 30 km/h at ≤500 m (`driver.controller.js:39-73`) | no | once per leg |
| iOS Live Activity | server remaining km ÷ 24 km/h per tick, 12 s throttle (`driver.controller.js:681-696`); progress divides km by metres | no | per tick |
| Driver nav HUD | local per-step decrement from the last route (`useNavigationEngine.js:338-355`) | only at fetch | on reroute |
| Driver HomeScreen banner | initial `routeInfo` (`HomeScreen.js:358-363`), never updated | – | never |
| Server in-ride ETA (`driver:movement`) | `navigation.service.processPositionUpdate` (`:220-297`) — **no caller** | – | dead |
| Scheduled rides | none | – | – |

There are at least five different speed assumptions (24, 30 km/h) and four uncoordinated ETA owners on the passenger side alone (`driverETA`, `rideQuote.driverEtaSeconds`, `estimatedDuration`, `RideStatusSheet` countdown).

## 7. Current driver location flow

```
GPS chip
 ├─ foreground watchPositionAsync (LocationContext.js:376-387)
 │     idle: Balanced / 10 s / 10 m        ride: High / 3 s / 5 m
 │     → speed gate 200 km/h; not in_progress → PATCH /drivers/location every ≥5 s & ≥10 m
 │       in_progress → RideTrackingService throttle 3 s / 10 m → PATCH; 30 s heartbeat
 └─ background task startLocationUpdatesAsync (backgroundLocation.js:351-370)   ← runs in foreground too
       idle: Balanced / 10 s / 15 m, pauses automatically on iOS   ride: High / 3 s / 5 m
       → spoof filter → batch of 5 or 30 s → POST /drivers/location/batch (timestamps)
       → retry queue in SecureStore (replayed after fresh batch)
                     │
                     ▼
server driver.controller.updateDriverLocation (:550-713)
   reads only {latitude, longitude, heading, heartbeat}; ts = Date.now(); speed derived server-side
   prev position from Redis hash or 60 s auth-cached driver doc
   → driverLocation.service: GEOADD (if cached status === 'online'), HSET driver:loc:{id} TTL 120 s
   → Mongo write only if Redis failed; else 30 s flush
   → io.to(user:{passengerId}).emit('driver:locationUpdate', {rideId, location:{latitude, longitude, ts, heading}})
     (every accepted tick, no server throttle; admin room throttled 5 s in-process)
   → haversine ≤500 m → 'ride:driverApproaching' once per leg
                     │
                     ▼
passenger TaxiScreen (:1224-1297)
   rideId guard → server-ts monotonic guard → 5 m jitter filter → haversine ETA → 2 s throttle
   → AnimatedCarMarker: JS requestAnimationFrame tween via ShapeSource.setNativeProps,
     duration clamp(elapsed×0.8, 500, 3000) ms, bearing from GPS delta ≥8 m else server heading,
     no route snapping, new target cancels in-flight tween (car decelerates between packets)
```

Socket.IO carries no driver location **into** the server; ingestion is HTTP only. Device `timestamp`, `speed`, and `accuracy` are discarded on the primary path. The batch path trusts client `heading`/`speed` and does not compare against the stored `ts`.

## 8. Current map rendering flow

**Passenger `TaxiScreen`:** one `MapViewWrapper` (Mapbox `streets-v12`/`dark-v11`), children built in four memoised groups: pins (`:2919-3112`), driver car (`:3158-3161`), route (`:3165-3209`), ambient driver cluster (`:3213-3217`). Every ETA change and every unthrottled compass reading re-runs the pin group because `driverETA` and `userHeading` are in its dependency list (`:3116-3118`) and pin `onPress` handlers are inline arrows (`:3066,3099,3110`), which defeats `memo` on `DraggablePickupMarker`, `DestinationMarker`, `StopMarker`. Camera moves are listed in Section 13 of the findings; there is no camera mode, only eight boolean refs. Theme switches unmount every overlay for up to 3 s (`MapViewWrapper.js:153-187`).

**Driver `NavigationScreen`:** `TrafficPolyline` (congestion runs) + `AnimatedMarker` puck + destination marker. The puck receives both a `coordinate` prop change (starts a 500 ms tween) and an imperative `animateMarkerToCoordinate` call (starts a second tween whose "from" is the previous "to"), so it snaps rather than glides (`NavigationScreen.js:195-214`, `AnimatedMarkerWrapper.js:112-118,205-212`). The polyline's first vertex is the moving snapped point, so the memo comparator fails and the whole GeoJSON is re-serialised every tick (`TrafficPolyline.js:85-88,131-143`). Camera: one `setCamera(linearTo)` per fix with speed-adaptive zoom and pitch 50 (`cameraController.js:74-95`), follow mode cancelled on pan.

**Driver `HomeScreen`:** a second, independent navigation implementation (`:317-455`): own route fetch, own projection, own step advance, `animateCamera` 800 ms per fix with raw heading coerced to 0 at stops, static `Marker` on Android re-rendered per tick, no reroute, no voice. `LocationContext` publishes a new `location` object per fix inside a memoised context value (`LocationContext.js:736-749`), so `HomeScreen` (2,260 lines, three modals) re-renders on every fix.

**Manager app:** only `MarkerWrapper` is used (markers snap on each 30 to 45 s poll); `AnimatedMarkerWrapper` and `PolylineWrapper` have no importers. `OpsMap` reads `locationUpdatedAt` and `heading` that the ops snapshot never sends (`OpsMap.js:184,194` vs `ops.controller.js:88-100`), so stale dimming never fires and every car points north. Expo Go renders raw OSM tiles through a hand-written Mercator projector (`MapViewWrapper.js:59-73,147-177`).

**Admin SPA / web:** Google Maps JS via seven duplicated `<script>` loaders without `loading=async`, `language`, or `region`; legacy `Marker`; `AdminLiveMap` is the only surface that patches positions from `driver:locationUpdate`.

**Shared wrapper divergence:** `diff` across the three apps shows only `polylineSimplify.js` identical everywhere. `MapViewWrapper.js` differs by 203 lines between passenger and manager; the NaN guard in `AnimatedMarkerWrapper.pushShape` exists only in the passenger copy; the `memo` comparator and `opacity` prop exist only in the manager copy; polyline layer ordering differs between passenger (`belowLayerID`) and driver/manager (`aboveLayerID`).

---

# Part B. Findings

Every finding uses the required structure. IDs are stable so the phase plan (Section 20) and file list (Section 21) can reference them.

## 9. All discovered bugs (correctness)

### B-01
**Issue:** Client can bypass the server-authoritative quote by sending coordinates as strings.
**Severity:** Critical
**Location:** `server/middlewares/validators.js:28-33`; `server/services/pricing.service.js:120-123`; `server/controllers/ride.controller.js:220-227,324`
**Current implementation:** `body('pickup.lat').isFloat(...)` validates without `.toFloat()`, so `"42.27"` passes. `computeServerQuote` returns `null` when `typeof lat !== 'number'`. The controller then keeps `effectiveQuote = quote` (the client's) and persists it. The only remaining guard is the sanity band at `:203-211` (min fare to `max(100, …)`).
**Problem:** Any client can book at a self-chosen price up to 100 GEL with no surge; commission and driver earnings are computed from it.
**User impact:** Under-payment, disputes, driver earnings computed on fabricated fares.
**Performance impact:** None.
**Google API cost impact:** None.
**Recommended solution:** Add `.toFloat()` in the validator and coerce in `computeServerQuote`; when the server quote cannot be computed, reject with 400 or price from the server fallback path, never from the client; store `quote.source` (`routing|fallback|admin`).
**Migration phase:** P1 (validator, immediately), P4 (Routes-based quote)

### B-02
**Issue:** A parallel queue publish broadcasts every ride request to all drivers, defeating sequential ETA dispatch.
**Severity:** High
**Location:** `server/controllers/ride.controller.js:717-726`; `server/workers/rideEventWorker.js:51-55,59,75`
**Current implementation:** `createRide` starts the ETA offer loop (`:549-629`) and, in parallel, `publishRideEvent('ride:request', …, {broadcastDrivers:true})`; the worker emits to `drivers:all` and `admin` immediately and sends no push because `pushToDriver` is unset.
**Problem:** Every online driver sees the request while one driver is being offered it; offline drivers get no push except the one being offered.
**User impact:** "Fastest finger" behaviour returns; drivers see rides they cannot accept; DM spend on ranking is wasted.
**Performance impact:** N socket emits per ride to the whole fleet.
**Google API cost impact:** Indirect: dispatch Route Matrix calls buy nothing when the broadcast wins.
**Recommended solution:** Remove `broadcastDrivers` from this publish (keep the admin emit); set `pushToDriver` for the offered driver only; broadcast only in the explicit no-accept fallback at `:633-644`.
**Migration phase:** P1

### B-03
**Issue:** Stops are ignored by the booking quote, `activeRoute`, and the quote endpoint.
**Severity:** High
**Location:** `server/services/pricing.service.js:128`; `server/controllers/ride.controller.js:1337-1341,2971-2988`; stops validated only for truthiness at `:280-282`
**Current implementation:** `getRoute(pickup, dropoff)` and `getNavRoute(pickup, dropoff)` with no waypoints; `/rides/quote` accepts no stops.
**Problem:** Multi-stop rides are priced and drawn as direct trips; only the driver nav session honours stops.
**User impact:** Under-charging; passenger route and ETA wrong; the passenger's line disagrees with the driver's.
**Performance impact:** None.
**Google API cost impact:** None; the fix is the same single `computeRoutes` call with `intermediates`.
**Recommended solution:** Pass `stops` as waypoints in all three places; with Routes API use `intermediates` and sum `legs[]`; validate stops with the same rules as pickup/dropoff.
**Migration phase:** P4

### B-04
**Issue:** Final fare is driver-submitted within ±15 % and never reconciled; waiting fee is computed but not charged.
**Severity:** High
**Location:** `server/controllers/ride.controller.js:1412,1459-1472,1284-1296,1647`; driver clients `mobile-driver/src/services/api.js:95`, `RideDetailScreen.js:324-325`, `NavigationScreen.js:277-278`
**Current implementation:** `finalFare = fare ?? quote.totalPrice`, accepted if within 15 % of the quote; `waitingFee` stored but excluded; receipt derives `distanceCharge = fare − base − waitingFee`.
**Problem:** A modified driver client adds 15 % on every ride undetectably; receipts mis-attribute components; `validateFinalFare` in `pricing.service.js:215-226` is an unused duplicate.
**User impact:** Overcharging risk; wrong receipts.
**Performance impact:** None.
**Google API cost impact:** None.
**Recommended solution:** Compute the final fare server-side = quoted fare + waiting fee (+ a re-route surcharge only when the driven track deviates materially, using a Routes recompute); drop `fare` from the driver payload or restrict it to admin.
**Migration phase:** P4/P5

### B-05
**Issue:** A busy driver re-enters the dispatch GEO index for up to 60 s after accepting.
**Severity:** High
**Location:** `server/middlewares/auth.middleware.js:77-101`; `server/controllers/driver.controller.js:565,621-624`; `server/controllers/ride.controller.js:1023`; `invalidateDriver` only called from `auth.controller.js:1273,1371`
**Current implementation:** `acceptRide` sets `busy` and `ZREM`s the driver, but the next location tick (≤3 s later) uses the 60 s-cached `req.driver.status === 'online'` and `GEOADD`s again with hash `status:'online'`, which passes the defence filter in `etaDispatch.service.js:260-263`.
**Problem:** Busy drivers are candidates for a minute; each burns a 15 s offer slot and Route Matrix elements.
**User impact:** Slower matching; drivers see offers they cannot accept.
**Performance impact:** Wasted offer rounds.
**Google API cost impact:** Matrix elements spent on non-dispatchable drivers.
**Recommended solution:** Call `invalidateDriver(userId)` in accept/complete/cancel/status change, or decide `indexInGeo` from the Redis hash status rather than the auth cache.
**Migration phase:** P1

### B-06
**Issue:** Server in-ride ETA and geofencing code is unreachable, and its geofences could never fire.
**Severity:** High
**Location:** `server/services/navigation.service.js:220-297` (no caller outside tests); geofences built at `:171-189` without `active`; `checkGeofences` skips `!gf.active` at `:263`
**Current implementation:** `processPositionUpdate` is exported and never invoked; no `nav:position` socket event exists; the driver app never emits snapped positions.
**Problem:** `driver:movement` with traffic ETA, `driver:arriving`, and server arrival detection do not exist in production.
**User impact:** No server ETA after accept; passengers get 30 km/h straight-line guesses.
**Performance impact:** Nav session hashes written for nothing.
**Google API cost impact:** None today.
**Recommended solution:** Delete in P1; rebuild in P5/P7 as a server-owned ETA refresher driven by the existing location ingestion (Section 19.6).
**Migration phase:** P1 (remove), P5/P7 (rebuild)

### B-07
**Issue:** Mongo `$near` fallback when the Redis GEO set is merely empty offers rides to drivers whose app stopped reporting.
**Severity:** Medium
**Location:** `server/services/etaDispatch.service.js:234-248`; `server/services/driverDispatch.service.js:70-97`
**Current implementation:** `nearby === null || nearby.length === 0` → Mongo query on `status:'online'` with no freshness filter; `etaSeconds` is `null`.
**Problem:** Drivers whose 120 s hash expired but never went offline are offered rides; each costs a 15 s timeout.
**User impact:** Up to 75 s of dead offers before broadcast.
**Performance impact:** Minor.
**Google API cost impact:** None on that path, but no ETA ranking either.
**Recommended solution:** Fall back to Mongo only when Redis is down (`null`); on empty, widen the radius or broadcast; add a `location.updatedAt` freshness filter.
**Migration phase:** P5

### B-08
**Issue:** The quote-screen price formula differs from the booked price formula.
**Severity:** Medium
**Location:** `server/controllers/ride.controller.js:3061-3073` vs `server/services/pricing.service.js:48-84`; client `mobile/src/screens/TaxiScreen.js:1945-1949,1975,2074-2093`, `RideOptionsSheet.js:26-30`
**Current implementation:** Quote endpoint: `round((base + km×kmPrice)×surge, 0.1)`, no `minFare`. Booking: per-component rounding to 0.01 plus `minFare`. Client: its own `base + km×kmPrice` with no surge, displayed instead of the server quote it already fetched.
**Problem:** Three prices for one trip.
**User impact:** "Price changed" surprises; under surge the displayed price is wrong until booking.
**Performance impact:** None.
**Google API cost impact:** The `/rides/quote` call is wasted for price.
**Recommended solution:** `getRideQuote` calls `pricingService.quote()`; the client renders `rideQuote.totalPrice` per class and deletes `calculatePrice`.
**Migration phase:** P4/P5

### B-09
**Issue:** Live Activity progress divides kilometres by metres.
**Severity:** Low
**Location:** `server/controllers/driver.controller.js:681-686` (`quote.distance` is metres per `ride.controller.js:240`, `pricing.service.js:168`)
**Current implementation:** `remainingKm / quote.distance`.
**Problem:** Progress ≈ 1 from the first tick.
**User impact:** Lock-screen bar shows "almost there" immediately.
**Performance impact:** None. **Google API cost impact:** None.
**Recommended solution:** Divide by `quote.distance / 1000`; later use remaining route distance from the server ETA refresher.
**Migration phase:** P1

### B-10
**Issue:** Ride watchdog covers only `in_progress`, infers "moving" from ride age, and reads stale Mongo positions.
**Severity:** Medium
**Location:** `server/services/rideWatchdog.service.js:104-106,159-160,192-197`
**Current implementation:** Ride age < 5 min ⇒ moving; `driver.location` from Mongo although `metaMap` already holds the Redis position.
**Problem:** A driver going dark during `accepted` is never detected.
**User impact:** Frozen car with no warning before pickup.
**Performance impact:** None. **Google API cost impact:** None.
**Recommended solution:** Include `accepted`/`driver_arrived`; use `meta.speed`; emit `metaMap` coordinates.
**Migration phase:** P7

### B-11
**Issue:** Nearby-driver and quote endpoints use the exact vehicle type, return no ids or headings, and the Mongo fallback is unbounded.
**Severity:** Medium
**Location:** `server/controllers/driver.controller.js:1128-1145,1176`; `server/controllers/ride.controller.js:2999,3019` vs `driverDispatch.service.js:52-59`
**Current implementation:** `[vehicleType]` exact; payload `{lat,lng}` only; no `.limit()` on the Mongo path.
**Problem:** Quote says "no driver" for economy when a comfort driver would be dispatched; passenger map cannot rotate ambient cars.
**User impact:** False "no drivers available"; static markers.
**Performance impact:** Unbounded Mongo read on fallback.
**Google API cost impact:** None.
**Recommended solution:** Use `getEligibleDriverTypes`; return `{id, lat, lng, heading, ts}`; `.limit(50)`.
**Migration phase:** P5/P6

### B-12
**Issue:** Arrive/complete proximity gates read Mongo positions that lag Redis by up to 30 s and are spoofable.
**Severity:** Medium
**Location:** `server/controllers/ride.controller.js:1139-1155,1425-1441`; `driver.controller.js:636-641`; two different client thresholds `mobile-driver/src/screens/NavigationScreen.js:41,166` (50 m) and `RideDetailScreen.js:35` (500 m)
**Current implementation:** With Redis-backed locations, Mongo is written only by the 30 s flush; the gate compares against that copy.
**Problem:** "You are 620 m from pickup" right after pulling up; conversely any client-sent coordinate passes.
**User impact:** Failed "Arrived"/"Complete" taps.
**Performance impact:** None. **Google API cost impact:** None.
**Recommended solution:** Read `getDriverMeta` first; require N recent Redis points within radius; one server-owned constant.
**Migration phase:** P5

### B-13
**Issue:** Navigation socket payload coordinates are validated only as `type:'object'`.
**Severity:** Medium
**Location:** `server/socket/handlers/navigation.js:41-48,78,100` (`COORD_SCHEMA` at `:10-13` unused); `server/services/navigation.service.js:146-165`
**Current implementation:** No range check on nested coords.
**Problem:** Malformed payloads write `NaN` into Redis sessions and trigger provider calls with garbage.
**User impact:** Nav session silently broken.
**Performance impact:** Wasted provider calls.
**Google API cost impact:** Billed error requests after migration.
**Recommended solution:** Validate with `COORD_SCHEMA` and a Georgia bounding box.
**Migration phase:** P1

### B-14
**Issue:** Scheduled rides freeze surge and route at booking (up to 7 days early) and are broadcast rather than dispatched.
**Severity:** Medium
**Location:** `server/controllers/ride.controller.js:224-226,284-300,2872-2963`
**Current implementation:** Quote computed once; `broadcastScheduledRides` sends the stored quote to type rooms at T-10 min.
**Problem:** Wrong surge; no traffic ETA at execution; broadcast storm.
**User impact:** Mis-priced scheduled trips; slow matching at start time.
**Performance impact:** None. **Google API cost impact:** None today.
**Recommended solution:** Re-quote at T-10 min with Routes `departureTime = scheduledFor`, then run the ETA dispatch loop.
**Migration phase:** P4/P5

### B-15
**Issue:** `language='ka,en'` is passed to Google and fragments the geocode cache key.
**Severity:** Medium
**Location:** `server/routers/locations.router.js:49,68`; `server/providers/google.provider.js:190,207`; `cache.service.js:72-73`
**Current implementation:** `/locations/*` default `'ka,en'` flows unchanged into `google.forwardGeocode/reverseGeocode` and the key.
**Problem:** Google accepts one code and silently falls back; the same point is cached under `:ka` and `:ka,en`.
**User impact:** Latin-script labels for Georgian users on the manager app.
**Performance impact:** Double cache footprint.
**Google API cost impact:** Duplicate reverse calls for the same point.
**Recommended solution:** Normalise to one code (`ka` default, allowlist `ka|en|ru`) before keying.
**Migration phase:** P3

### B-16
**Issue:** Reverse geocoding takes `results[0]` with no `result_type`/`location_type` filter.
**Severity:** Low
**Location:** `server/providers/google.provider.js:207-215`
**Problem:** First result may be a plus code or route segment; pin labels look odd.
**User impact:** Confusing pickup labels. **Performance impact:** None. **Google API cost impact:** None.
**Recommended solution:** Request `result_type=street_address|route|premise|subpremise` and prefer `ROOFTOP`/`RANGE_INTERPOLATED`; return a provider-neutral `{mainText, secondaryText, formattedAddress, placeId}`.
**Migration phase:** P3

### B-17
**Issue:** Steps are lost for multi-leg OSRM routes; Google step names are stripped HTML instructions.
**Severity:** Low
**Location:** `server/providers/osrm.provider.js:72`; `server/providers/google.provider.js:123`
**Problem:** Turn-by-turn banner content depends on which provider answered.
**User impact:** Inconsistent driver navigation with stops. **Performance impact:** None. **Google API cost impact:** None.
**Recommended solution:** Routes API: iterate all legs; use `navigationInstruction.instructions` with `languageCode: 'ka'`.
**Migration phase:** P4/P7

### B-18
**Issue:** Temporal-dead-zone reference in a `useEffect` dependency array.
**Severity:** High
**Location:** `mobile/src/screens/TaxiScreen.js:496` uses `handleDestinationSelectWithCoords`, declared as `const` at `:2037`
**Current implementation:** The dependency array is evaluated during render before the `const` initialises; Hermes returns `undefined` instead of throwing.
**Problem:** Instant `ReferenceError` on any strict-TDZ engine or in any test that renders the screen.
**User impact:** None today; crash if the JS engine changes.
**Performance impact:** One extra effect run per mount. **Google API cost impact:** None.
**Recommended solution:** Move the Book-Again effect below `:2050` or route through a ref.
**Migration phase:** P1

### B-19
**Issue:** App-foreground refresh silently resets a manually chosen pickup without refetching route or price.
**Severity:** High
**Location:** `mobile/src/screens/TaxiScreen.js:594` → `refreshLocation` `:1839-1840`
**Current implementation:** Every foreground calls `refreshLocation()`, whose `applyCoords` does `setLocation` + `setCustomPickup(null)` + reverse geocode when the fix moved ≥20 m.
**Problem:** In RIDE_OPTIONS with a custom pickup, returning from background drops the pickup to GPS while `routePolyline` and `estimatedPrice` stay stale.
**User impact:** Ride requested from the wrong point with a price for another.
**Performance impact:** One extra geocode per foreground.
**Google API cost impact:** +1 Geocoding per foreground; the Routes call that is needed is missing.
**Recommended solution:** Refresh only in LOCATION_SEARCH and only when `customPickup == null`.
**Migration phase:** P1

### B-20
**Issue:** Users without a GPS fix cannot book even with a valid typed pickup.
**Severity:** High
**Location:** `mobile/src/screens/TaxiScreen.js:2147-2150` (vs `:2195`, which already supports `customPickup`)
**Current implementation:** `if (!location) Alert(...)`.
**Problem:** Permission-denied and indoor users are hard-blocked.
**User impact:** Lost bookings.
**Performance impact:** None. **Google API cost impact:** None.
**Recommended solution:** Gate on `customPickup || location`.
**Migration phase:** P1

### B-21
**Issue:** SOS from the ride status sheet sends `location: null`.
**Severity:** Medium
**Location:** `mobile/src/screens/TaxiScreen.js:2844-2859` (no `userLocation` prop) vs `mobile/src/components/taxi/RideStatusSheet.js:98-101`
**Problem:** The passenger's coordinates never reach the SOS payload.
**User impact:** Safety feature degraded. **Performance impact:** None. **Google API cost impact:** None.
**Recommended solution:** Pass `locationRef.current`.
**Migration phase:** P1

### B-22
**Issue:** Passenger ETA and route are frozen for the whole in-progress phase.
**Severity:** Medium
**Location:** `mobile/src/screens/TaxiScreen.js:1054,1448-1451,1151`
**Current implementation:** Route effect returns early for IN_PROGRESS; `driverETA` set once from `activeRoute.durationSeconds`; `osrmEtaPriorityRef` suppresses updates.
**Problem:** Countdown, Live Activity, and polyline never update during the trip.
**User impact:** Stale "arriving in N min". **Performance impact:** None. **Google API cost impact:** None (the fix must not add a per-tick Routes call).
**Recommended solution:** Server pushes remaining ETA/distance with location updates (Section 19.6); client trims the polyline locally.
**Migration phase:** P5/P7

### B-23
**Issue:** Recent places never deduplicate, never expire, are not deleted with the account, and carry no place id.
**Severity:** Medium
**Location:** `server/services/recentLocations.service.js:11,53-54`; `server/controllers/ride.controller.js:366`; `server/jobs/hardDelete.js:72-81`; client `mobile/src/screens/TaxiScreen.js:2065-2071`
**Current implementation:** `LPUSH` + `LTRIM 0 19`, no `EXPIRE`, `canonicalId` always `null`; a recent entry without coords does nothing when tapped.
**Problem:** Ten rides home show ten identical rows; location history survives account deletion.
**User impact:** Cluttered list; GDPR exposure.
**Performance impact:** Negligible. **Google API cost impact:** None.
**Recommended solution:** Dedupe by place id or rounded coords, 180-day TTL, delete in hard-delete, store `placeId` + coords.
**Migration phase:** P1 (hygiene), P2 (place id)

### B-24
**Issue:** Minor passenger marker bugs.
**Severity:** Low
**Location:** `mobile/src/components/map/AnimatedCarMarker.js:101-104,174` (initial shape memoised once with `[]`, so an invalid first coordinate means the car never renders); `TaxiScreen.js:644,1339,1344` (driver location stored without `_ts`, disabling the stale-timestamp guard at `:1232`); `TaxiScreen.js:2735` vs `mapboxGeo.js:69` (zoom formula off by one level for `DriverCluster`)
**Problem:** Edge-case rendering errors.
**User impact:** Rare missing car; cluster threshold off by one zoom level. **Performance impact:** None. **Google API cost impact:** None.
**Recommended solution:** Derive initial shape from the first valid coordinate; always stamp `_ts`; one zoom helper.
**Migration phase:** P1/P6

### B-25
**Issue:** Every driver GPS fix is uploaded twice and the late copy moves the car backwards.
**Severity:** High
**Location:** `mobile-driver/src/context/LocationContext.js:453-458,597-632`; `mobile-driver/src/services/backgroundLocation.js:201-269`; server `driver.controller.js:670,1258-1270,1290-1296`
**Current implementation:** The foreground-service task keeps delivering while the app is in the foreground, so the foreground PATCH path and the 30 s background batch both send; the server stamps `ts = Date.now()` and the batch endpoint does not compare against the stored `ts`.
**Problem:** Batch arrives up to 30 s late with a newer server timestamp; the passenger's monotonic guard accepts it.
**User impact:** The car teleports backwards every ≤30 s.
**Performance impact:** ~2× location requests, Redis and Mongo writes, socket emits.
**Google API cost impact:** Doubles any future per-update server work (ETA refresh triggers).
**Recommended solution:** One pipeline (Section 19.7): keep the TaskManager task as the single source, fan out to UI via an emitter; send device `timestamp` and `accuracy`; server rejects points older than the stored `ts`.
**Migration phase:** P1 (server ordering guard), P7 (single pipeline)

### B-26
**Issue:** The driver location watchdog treats stationary silence as watcher death and restarts GPS every 10 s at a red light.
**Severity:** High
**Location:** `mobile-driver/src/context/LocationContext.js:66-70,188-216,385-386`; `backgroundLocation.js:333,343-349`
**Current implementation:** `FOREGROUND_STALE_RIDE 10000` with `distanceInterval 5`; background restart after 180 s.
**Problem:** The OS legitimately emits nothing while stationary; the app restarts subscriptions and the foreground service.
**User impact:** GPS re-acquisition gaps exactly when the passenger is watching; Android notification flicker; battery drain.
**Performance impact:** Repeated `watchPositionAsync` churn.
**Google API cost impact:** None.
**Recommended solution:** Gate staleness on last known speed (as `useNavigationEngine.js:388-405` does) or use `distanceInterval: 0` during rides and rely on `timeInterval`.
**Migration phase:** P1

### B-27
**Issue:** Accepting a ride from HomeScreen never escalates the GPS profile; any status change fully restarts tracking.
**Severity:** High
**Location:** `mobile-driver/src/screens/HomeScreen.js:633-664`; `LocationContext.js:636-673`; callers only `RideDetailScreen.js:140`, `NavigationScreen.js:129,139,252,282`; `DriverContext.loadActiveRides` never calls it
**Current implementation:** `setActiveRide` is screen-driven; when called it does `stopTracking()` then `startTracking()`.
**Problem:** Built-in-map drivers on Home stay at Balanced/10 s/10 m for the to-pickup leg; after an app restart the profile stays idle until RideDetail opens; each phase transition creates a multi-second location gap.
**User impact:** Coarse, laggy driver position for the passenger; late arrival detection.
**Performance impact:** GPS cold starts. **Google API cost impact:** None.
**Recommended solution:** Drive the profile from `DriverContext.activeRides[0]` in one effect; re-issue subscriptions with new options without stopping.
**Migration phase:** P1

### B-28
**Issue:** The navigation puck's glide is cancelled by a second interpolation every tick.
**Severity:** High
**Location:** `mobile-driver/src/screens/NavigationScreen.js:195-214`; `mobile-driver/src/components/map/AnimatedMarkerWrapper.js:112-118,205-212`
**Current implementation:** The `coordinate` prop effect starts a 500 ms tween; the parent's effect then calls `animateMarkerToCoordinate` with the same target, which sets `from = to` and cancels the first RAF.
**Problem:** The second tween is target→target; the marker snaps.
**User impact:** Stuttering puck, the exact symptom the navigation rebuild aimed to remove.
**Performance impact:** Two RAF chains per tick. **Google API cost impact:** None.
**Recommended solution:** One driver of motion per marker (Section 19.9).
**Migration phase:** P6

### B-29
**Issue:** HomeScreen runs a second, divergent navigation engine with raw heading coerced to 0 and camera animation per fix.
**Severity:** High
**Location:** `mobile-driver/src/screens/HomeScreen.js:317-455,739-750,798,807,906-952`
**Current implementation:** Own route fetch, projection, step advance, `animateCamera` 800 ms per fix, no reroute, no voice, static ETA (`:358-363`).
**Problem:** Contradicts the single-engine design in `NAVIGATION_SYSTEM_HANDOFF.md`; map spins north at every stop; instructions never advance on deviation.
**User impact:** Jerky camera, wrong ETA on the primary screen.
**Performance impact:** Second projection + simplification per tick.
**Google API cost impact:** One extra route fetch per phase when both screens are visited.
**Recommended solution:** Mount `useNavigationEngine` in a shared provider or make Home a static overview fed by the engine.
**Migration phase:** P7

### B-30
**Issue:** `useRideRecovery` and `usePermissionMonitor` are never mounted.
**Severity:** High
**Location:** `mobile-driver/src/hooks/useRideRecovery.js`, `usePermissionMonitor.js` (no importers); `RideTrackingService.js:241`
**Current implementation:** After a process kill mid-ride nothing restarts `RideTrackingService`, flushes the buffer, or clears stale `@ride:active`; the permission monitor would send `latitude: 0, longitude: 0` if mounted (`usePermissionMonitor.js:70-75`).
**Problem:** Lost breadcrumbs; stale ride state persists into the next ride; permission revocation undetected.
**User impact:** Broken recovery after crashes. **Performance impact:** None. **Google API cost impact:** None.
**Recommended solution:** Mount recovery in `App.js` after auth; rewrite the permission monitor to call a dedicated endpoint.
**Migration phase:** P1

### B-31
**Issue:** Server navigation routes are computed and discarded; reroutes are computed twice; reroutes storm while off-route.
**Severity:** Medium
**Location:** `mobile-driver/src/navigation/useNavigationEngine.js:242-277,320-327,409-438,464-487`; server `navigation.service.js:401,460`
**Current implementation:** Client always fetches its own route; `nav:route_computed` is adopted only if no local route exists; `nav:off_route` triggers a server reroute and a local fetch; while still off-route every tick calls `triggerReroute` under an 8 s cooldown; the origin carries no heading or accuracy.
**Problem:** Two routing calls per leg and per reroute; a driver on a parallel road 40 m away triggers a paid route every 8 s.
**User impact:** Polyline flicker; repeated "recalculating".
**Performance impact:** Minor.
**Google API cost impact:** Up to 2× `computeRoutes` per leg and unbounded reroutes per minute per driver.
**Recommended solution:** One route owner (Section 19.10); exponential back-off after the first reroute; require lateral > `max(40 m, 1.5 × accuracy)` and heading deviation; pass `vehicleHeading`.
**Migration phase:** P4/P7

### B-32
**Issue:** Retry queue in SecureStore replays old points after fresh ones and has no lock.
**Severity:** Medium
**Location:** `mobile-driver/src/services/backgroundLocation.js:21-23,97-130,175-188`
**Problem:** Old points are sent after the fresh batch; the server stamps them newer; keychain I/O in a background task; read-modify-write races.
**User impact:** Car jumps back after connectivity loss. **Performance impact:** Keychain I/O. **Google API cost impact:** None.
**Recommended solution:** AsyncStorage or SQLite, oldest-first flush, device timestamps, server drops points older than last accepted.
**Migration phase:** P1

### B-33
**Issue:** Speed, accuracy, and device timestamp are dropped on the primary location path.
**Severity:** Medium
**Location:** `mobile-driver/src/services/RideTrackingService.js:219-223`; `LocationContext.js:628`; server `driver.controller.js:551,1288-1292`
**Problem:** Passenger ordering uses server receive time; no accuracy for confidence or filtering; no speed for server ETA.
**User impact:** Cannot fix B-25/B-32 server-side without it. **Performance impact:** None. **Google API cost impact:** None.
**Recommended solution:** Send `{lat, lng, heading, speed, accuracy, ts}` on both paths; forward `ts` and `accuracy` in `driver:locationUpdate`.
**Migration phase:** P7

### B-34
**Issue:** iOS idle tracking can pause indefinitely while a parked driver waits for requests.
**Severity:** Medium
**Location:** `mobile-driver/src/services/backgroundLocation.js:368-369`; `RideTrackingService.js:114-116`
**Current implementation:** `pausesLocationUpdatesAutomatically: true` with `AutomotiveNavigation` while idle; significant-change recovery only during `in_progress`.
**Problem:** Dispatch sees a stale driver.
**User impact:** Missed offers. **Performance impact:** None. **Google API cost impact:** None.
**Recommended solution:** `pausesLocationUpdatesAutomatically: false` with Balanced accuracy while online, or register the significant-change task whenever online.
**Migration phase:** P1

### B-35
**Issue:** Ride status exists in five places on the driver; `DriverContext` ignores the `ride:updated` payload.
**Severity:** Medium
**Location:** `mobile-driver/src/context/DriverContext.js:161-163,169`; `NavigationScreen.js:58-59`; `RideDetailScreen.js`; `LocationContext.activeRideRef`; `rideStorage`
**Problem:** Home's `activeRide` can lag Navigation's; screens patch their own copies.
**User impact:** Inconsistent status and `activeRoute` between screens. **Performance impact:** None. **Google API cost impact:** None.
**Recommended solution:** Merge the payload into `activeRides` and make it the single ride source.
**Migration phase:** P1

### B-36
**Issue:** In-progress breadcrumbs are appended twice with mixed clocks.
**Severity:** Medium
**Location:** `mobile-driver/src/services/RideTrackingService.js:168-180,226-228,239-258`; `backgroundLocation.js:221-223`
**Problem:** Duplicate, out-of-order `routePoints`; wrong actual distance if ever used for pricing.
**User impact:** Unusable dispute evidence. **Performance impact:** 2× storage writes. **Google API cost impact:** None.
**Recommended solution:** Single ingestion point, dedupe by device timestamp.
**Migration phase:** P7

### B-37
**Issue:** Manager ops map reads fields the snapshot never sends.
**Severity:** High (for the ops surface; feature-flagged off today)
**Location:** `mobile-manager/src/components/ops/OpsMap.js:184,194` vs `server/controllers/ops.controller.js:88-100`
**Problem:** `isFresh(undefined)` returns true, so stale dimming never fires; heading is always 0.
**User impact:** Operators see hour-old positions as current, all cars pointing north. **Performance impact:** None. **Google API cost impact:** None.
**Recommended solution:** Add `locationUpdatedAt` and `heading` to the snapshot payload with a shape test.
**Migration phase:** P6

### B-38
**Issue:** Admin ride creation queries public Nominatim from the browser and derives pricing distance client-side.
**Severity:** High
**Location:** `client/src/pages/admin/AdminCreateRide.jsx:46-48,182-184`; `client/src/components/RouteMap.jsx:57-93` → `routeInfo` posted; server `ride.controller.js:770,792-796`
**Problem:** Browsers drop the `User-Agent` header, so requests arrive anonymous (OSM policy breach); results carry no canonical id; the server trusts admin `routeInfo` distance/duration verbatim.
**User impact:** Intermittent 403/429 from OSM; dispatcher's price may differ from the server's.
**Performance impact:** None. **Google API cost impact:** Uncached browser Directions calls.
**Recommended solution:** Use the server proxy for search and route; take fare inputs from the server quote.
**Migration phase:** P2/P3/P4

### B-39
**Issue:** No default location bias when the client omits coordinates; country-wide ranking prefers Tbilisi.
**Severity:** High
**Location:** `server/providers/google.provider.js:228-231`; `server/controllers/maps.controller.js:182`; `autocomplete.service.js:54-56`; manager `services/api.js:232`
**Problem:** Kutaisi users typing common street names see Tbilisi first.
**User impact:** More keystrokes to disambiguate; wrong selections.
**Performance impact:** None. **Google API cost impact:** More autocomplete requests per selection.
**Recommended solution:** Default `locationBias` to the operating city centre (`server/utils/city.js:44`) with ~30 km radius; `locationRestriction` when the ride city is known.
**Migration phase:** P2

### B-40
**Issue:** Cross-provider dedupe and POI heuristic produce duplicates and defeat tiering for Georgian queries.
**Severity:** Medium
**Location:** `server/services/autocomplete.service.js:94-100,116-128,155-164`
**Problem:** Same street appears under Nominatim, Google, and `manual:` variants; one- or two-token Georgian street names classify as POI so Google is called anyway; `'სავარჯიშო'` means "exercise", not "gym".
**User impact:** Duplicates crowd out real options in 8 slots. **Performance impact:** Serial provider latency. **Google API cost impact:** Tiering saves far less than intended.
**Recommended solution:** Removed by Google-only autocomplete.
**Migration phase:** P2/P9

### B-41
**Issue:** The passenger reverse-geocode adapter is written against Nominatim component keys.
**Severity:** Medium (migration blocker)
**Location:** `mobile/src/services/googleMaps.js:162-198`
**Problem:** With Google-shaped `address_components`, `mainText` collapses to `address.split(',')[0]`.
**User impact:** Pin labels lose street + house number after migration. **Performance impact:** None. **Google API cost impact:** None.
**Recommended solution:** Server returns a provider-neutral `{placeId, mainText, secondaryText, formattedAddress, street, houseNumber, city}`; client stops parsing components.
**Migration phase:** P3

### B-42
**Issue:** Favorites cap race and multiple home/work entries; no place id.
**Severity:** Low
**Location:** `server/controllers/favorites.controller.js:45-57`; `server/models/favoriteLocation.model.js:33-37`
**Recommended solution:** Partial unique index `{user, type}` for home/work; atomic cap; optional `placeId`.
**Problem / User impact / Performance / Cost:** Confusing saved-place UI; none; none.
**Migration phase:** P2

### B-43
**Issue:** Fallback coordinates are Tbilisi in most places and are used as if real.
**Severity:** Low
**Location:** `mobile/src/screens/TaxiScreen.js:147-150`, `RideDetailScreen.js:214-218`; `mobile-driver/src/context/LocationContext.js:47-50` (used by SOS at `HomeScreen.js:502-505`), `HomeScreen.js:682-687`, `NavigationScreen.js:496-501`, `RideDetailScreen.js:415-418`; `client/src/pages/admin/AdminLiveMap.jsx:11`; `web/.../SharedRideView.jsx:47`
**Problem:** SOS and car marker report Tbilisi when GPS fails; empty maps open 200 km away.
**Recommended solution:** One `KUTAISI_CENTER` constant for camera defaults; keep `location = null` on failure and gate SOS/markers on real fixes.
**User impact / Performance / Cost:** Wrong SOS location; none; none.
**Migration phase:** P1

### B-44
**Issue:** Minor driver-app defects.
**Severity:** Low
**Location:** `mobile-driver/src/context/LocationContext.js:423-431` (speed gate also suppresses the UI update, contrary to its comment); `SocketContext.js:276-280` (any server `'error'` permanently disables reconnection); `services/directions.js:14-32` (route token cache never invalidated on logout → 401 after re-login within 5 min); `HomeScreen.js:298-299` with `mapSafety.js:119-122` (`hasFitted` set even when the debounced fit was rejected); `LocationContext.js:286-300` (Android background permission requested at cold start, English-only alert)
**Recommended solution:** Fix individually during P1 cleanup.
**Problem / User impact / Performance / Cost:** Edge-case failures; small; none; none.
**Migration phase:** P1

## 10. Performance problems

### P-01
**Issue:** Unthrottled compass heading re-renders the whole passenger screen and every pin.
**Severity:** High
**Location:** `mobile/src/screens/TaxiScreen.js:462-466` (`watchHeadingAsync`), `:3116` (`userHeading` in `mapChildren` deps), `:3066,3099,3110` (inline `onPress` arrows)
**Current implementation:** `setUserHeading` on every magnetometer callback; the pin group memo includes `userHeading`; pin handlers are new closures each render.
**Problem:** Tens of renders per second on noisy sensors; `memo` on `DraggablePickupMarker`/`DestinationMarker`/`StopMarker` is defeated.
**User impact:** Jank during pan/zoom; battery.
**Performance impact:** Full 3,725-line screen render plus marker element rebuilds per tick.
**Google API cost impact:** None.
**Recommended solution:** Throttle to ≥100 ms and ≥3°; feed heading to the user marker via a ref or use the SDK's own location layer; memoise handlers.
**Migration phase:** P6

### P-02
**Issue:** `driverETA` is a dependency of the marker tree; every ETA change rebuilds all pins and the status sheet.
**Severity:** Medium
**Location:** `mobile/src/screens/TaxiScreen.js:3083-3086,3118,2908`
**Problem:** ETA changes every 2 s during DRIVER_FOUND.
**User impact:** Frame drops while waiting. **Performance impact:** Marker and sheet re-render every 2 s. **Google API cost impact:** None.
**Recommended solution:** Isolated `PickupEtaPin`; with react-native-maps set `tracksViewChanges={false}` and toggle only when label text changes.
**Migration phase:** P6

### P-03
**Issue:** JS-thread `requestAnimationFrame` interpolation pushing `setNativeProps` at 60 Hz.
**Severity:** Medium
**Location:** `mobile/src/components/map/AnimatedCarMarker.js:62-73,160-168`; `mobile-driver/src/components/map/AnimatedMarkerWrapper.js:194-224`
**Problem:** 60 bridge messages per second; stutters whenever JS is busy (socket bursts, sheet renders).
**User impact:** Car stutters. **Performance impact:** Bridge saturation. **Google API cost impact:** None.
**Recommended solution:** Native-driven animation (Section 19.9).
**Migration phase:** P6/P7

### P-04
**Issue:** Route polyline re-serialised across the bridge every GPS tick on the driver.
**Severity:** Medium
**Location:** `mobile-driver/src/components/map/TrafficPolyline.js:85-88,131-143`; `PolylineWrapper.js:45-55,99-109`; `useNavigationEngine.js:313-315`; `HomeScreen.js:429-431`
**Problem:** The first vertex is the moving snapped point, so the memo fails and the full GeoJSON is rebuilt; HomeScreen re-runs Douglas-Peucker.
**User impact:** Frame drops on low-end Android. **Performance impact:** O(n) per tick for 1 to 2 k points. **Google API cost impact:** None.
**Recommended solution:** Re-trim only when `segmentIndex` changes; draw the puck-to-next-vertex stub as a separate 2-point line.
**Migration phase:** P6

### P-05
**Issue:** Whole driver screens re-render per GPS fix via `LocationContext`.
**Severity:** Medium
**Location:** `mobile-driver/src/context/LocationContext.js:442,736-749`; consumers `HomeScreen.js:128`, `RideDetailScreen.js:53`, `NavigationScreen.js:50`, `DriverContext.js:26`
**Problem:** `location` is in the memoised context value.
**User impact:** Jank during rides. **Performance impact:** Full reconciliation of a 2,260-line screen per fix. **Google API cost impact:** None.
**Recommended solution:** Split into a stable controls context and a subscription store for fixes (the unused `zustand` dependency fits), consumed with selectors.
**Migration phase:** P6

### P-06
**Issue:** Two High-accuracy native location clients during rides, with three copies of the profile table.
**Severity:** Medium
**Location:** `mobile-driver/src/context/LocationContext.js:376-387`; `backgroundLocation.js:351-370`; `RideTrackingService.js:324-338`
**User impact:** Battery. **Performance impact:** Two location pipelines in JS per fix. **Google API cost impact:** None.
**Recommended solution:** Single pipeline and one profile module (Section 19.11).
**Migration phase:** P7

### P-07
**Issue:** `activeRoute` stores a decoded polyline plus full step geometry per ride and is returned by list endpoints.
**Severity:** Medium
**Location:** `server/models/ride.model.js:290-298`; `server/controllers/ride.controller.js:1348-1361`; list projections only exclude `routePoints` at `:1965,2018,2125,2184`
**Problem:** Tens of KB per ride document, re-sent in history lists and in `ride:started`.
**User impact:** Slow history screens. **Performance impact:** Document and socket payload bloat. **Google API cost impact:** None.
**Recommended solution:** Store the encoded polyline string plus `distanceMeters/durationSeconds`; exclude from list projections; decode on the client.
**Migration phase:** P4/P6

### P-08
**Issue:** Theme switch unmounts every map overlay for up to 3 s; Mapbox style surgery in the wrapper.
**Severity:** Low
**Location:** `mobile/src/components/map/MapViewWrapper.js:153-187`; `mobile-manager/src/components/map/MapViewWrapper.js:277-330,467-500`
**Recommended solution:** Cloud-based map style `mapId` per theme; delete the state machine.
**User impact / Performance / Cost:** Blank overlays on theme change; none; none.
**Migration phase:** P6

### P-09
**Issue:** `handleDestinationSelectWithCoords` blocks on the network before advancing the sheet; worst case ≈41 s.
**Severity:** Medium
**Location:** `mobile/src/screens/TaxiScreen.js:2041-2049`; timeouts `api.js:14,81-90`, `googleMaps.js:239-241`
**Problem:** Axios 10 s + two retries + OSRM 8 s all awaited before RIDE_OPTIONS shows.
**User impact:** Long "calculating route" with no way forward. **Performance impact:** None. **Google API cost impact:** None.
**Recommended solution:** Advance immediately with a skeleton; fill route and price asynchronously; single 6 s budget.
**Migration phase:** P4

### P-10
**Issue:** In-process throttles and limiters are per PM2 worker.
**Severity:** Low
**Location:** `server/routers/maps.router.js:10-49`; `server/socket/opsThrottle.js:6-16`; `server/services/rideActivity.js:51-59`; `etaDispatch.service.js:57` (in-flight map)
**Problem:** With `PM2_INSTANCES > 1` the maps limit is N×, admin map gets N× emits, Live Activity gets N× frames, matrix coalescing misses across workers.
**User impact:** Live Activity spam. **Performance impact:** Redundant emits. **Google API cost impact:** Coalescing misses.
**Recommended solution:** Redis `SET NX PX` throttles and the Redis rate-limit store already used in `middlewares/rateLimiter.js`.
**Migration phase:** P8

### P-11
**Issue:** Mongo text index on Georgian text plus an unindexed regex fallback.
**Severity:** Medium
**Location:** `server/models/place.model.js:42`; `server/services/places.service.js:162-177`
**Problem:** `$text` uses the English stemmer and whole tokens; the prefix fallback is a collection scan on every short query.
**User impact:** Warm layer rarely hits. **Performance impact:** COLLSCAN per keystroke as `places` grows. **Google API cost impact:** Misses fall through to provider calls.
**Recommended solution:** Index `normalizedAddress`, or drop Mongo-side search once session-priced autocomplete makes it cheap.
**Migration phase:** P8

### P-12
**Issue:** Serial `await` per candidate for ETA cache reads.
**Severity:** Low
**Location:** `server/services/etaDispatch.service.js:117-121`
**Recommended solution:** `MGET`.
**Problem / User impact / Performance / Cost:** N round trips before the matrix call; milliseconds; minor; none.
**Migration phase:** P8

## 11. Race conditions

### R-01
**Issue:** Pan-to-adjust reverse geocoding has no request ordering; confirm commits the previous address with new coordinates.
**Severity:** High
**Location:** `mobile/src/screens/TaxiScreen.js:2504-2508,2716-2724`
**Current implementation:** 400 ms debounced `reverseGeocode` with no request id; `handleAdjustConfirm` reads `adjustAddress` while `adjustGeocoding` may be true.
**Problem:** A → B → C where A's response lands last overwrites C's label; confirm mid-lookup stores a mismatched address.
**User impact:** Wrong pickup address sent to the driver.
**Performance impact:** None. **Google API cost impact:** None.
**Recommended solution:** `reverseIdRef` + `AbortController` as in `SavedPlaceEditorScreen.js:144-156`; disable confirm while resolving, or resolve on confirm.
**Migration phase:** P3

### R-02
**Issue:** Drag-end and tap-to-select reverse geocodes are unguarded.
**Severity:** Medium
**Location:** `mobile/src/screens/TaxiScreen.js:2577-2589,2594,2755-2768`
**Recommended solution:** Same ordering guard; one `useReverseGeocode` hook.
**Problem / User impact / Performance / Cost:** Stale label after quick successive drags; wrong address; none; none.
**Migration phase:** P3

### R-03
**Issue:** `fetchDirectionsAndUpdate` has no cancellation or ordering across ten call sites.
**Severity:** High
**Location:** `mobile/src/screens/TaxiScreen.js:1955-2034`; callers `:2041,2528,2532,2536,2587,2601,2616,2627,2789,2803`
**Problem:** Rapid destination/stop/drag changes let an older response overwrite polyline, `totalDistance`, `estimatedPrice`, and camera fit.
**User impact:** Route and price for the wrong destination. **Performance impact:** Redundant fits. **Google API cost impact:** Discarded Routes calls.
**Recommended solution:** Request id + `AbortController` (pattern in `useRideQuote.js:72-110`); collapse route and quote into one server call.
**Migration phase:** P4

### R-04
**Issue:** Autocomplete has an ordering guard but no cancellation, and the effect restarts on every GPS tick.
**Severity:** Medium
**Location:** `mobile/src/components/taxi/LocationSearchSheet.js:123-155` (`searchIdRef`, deps include `userLocation`); `TaxiScreen.js:2895`; `api.js:81-90` (retries)
**Problem:** Late responses are dropped but still complete (and may be retried); the debounce restarts as `userLocation` is a fresh object per tick.
**User impact:** `isSearching` flicker; extra latency. **Performance impact:** Minor. **Google API cost impact:** Duplicate autocomplete requests.
**Recommended solution:** `AbortController` per query; pass a rounded, memoised bias point.
**Migration phase:** P2

### R-05
**Issue:** Out-of-order driver positions from the batch path and retry queue.
**Severity:** High
**Location:** see B-25, B-32; server `driver.controller.js:1258-1270`
**Problem:** Server timestamps on receipt, no compare-and-set against the stored `ts`.
**User impact:** Car moves backwards. **Performance impact:** None. **Google API cost impact:** None.
**Recommended solution:** Device timestamps end-to-end; reject older writes; passenger guard on device `ts`.
**Migration phase:** P1/P7

### R-06
**Issue:** Nav route version guard exists but the server and client both compute; both can land.
**Severity:** Medium
**Location:** `mobile-driver/src/navigation/useNavigationEngine.js:263-277,464-487`
**Recommended solution:** Single route owner (Section 19.10).
**Problem / User impact / Performance / Cost:** Double route swap; polyline flicker; minor; 2× calls.
**Migration phase:** P4/P7

### R-07
**Issue:** Passenger stale-timestamp guard is disabled after restore/arrival because `_ts` is not stamped.
**Severity:** Low
**Location:** `mobile/src/screens/TaxiScreen.js:644,1339,1344` vs `:1232`
**Recommended solution:** Always stamp `_ts`.
**Problem / User impact / Performance / Cost:** Old socket events accepted after restore; brief jump; none; none.
**Migration phase:** P7

### R-08
**Issue:** `onRegionChangeComplete` fires per frame during programmatic moves and inertia.
**Severity:** Medium
**Location:** `mobile/src/components/map/MapViewWrapper.js:253-276`; consumers `TaxiScreen.js:2689-2739`, `SavedPlaceEditorScreen.js:70-80`
**Problem:** Entering adjust mode geocodes the unchanged coordinate; each fling frame ≥5 m fires haptics and state updates.
**User impact:** Haptic spam; address flicker. **Performance impact:** Per-frame renders. **Google API cost impact:** One wasted geocode per adjust entry.
**Recommended solution:** react-native-maps' native `onRegionChangeComplete(region, {isGesture})` fires once per settle; drop the time-window hacks.
**Migration phase:** P6

### R-09
**Issue:** Favorites cap is count-then-create.
**Severity:** Low
**Location:** `server/controllers/favorites.controller.js:45-57`
**Recommended solution:** Atomic cap (see B-42).
**Problem / User impact / Performance / Cost:** Concurrent POSTs exceed 10; minor; none; none.
**Migration phase:** P2

## 12. API-cost problems

Cost impacts are stated for the post-migration Google-only architecture unless marked "today".

### C-01
**Issue:** Reverse geocoding up to every 5 s while stationary on the search screen.
**Severity:** High
**Location:** `mobile/src/screens/TaxiScreen.js:997-1020` (`movedEnough || timeEnough`), `:979` (5 m dedup lets jitter through), `googleMaps.js:361` (full-precision coordinates defeat the 11 m server cache)
**Problem:** GPS jitter of 5 to 20 m passes the dedup and the 5 s clock alone triggers a call.
**User impact:** Address label flicker.
**Performance impact:** Network churn.
**Google API cost impact:** Up to 720 Geocoding calls per hour per idle user. At 10,000 rides/day with three minutes on the search screen that is ≈360,000 geocodes/day ≈ USD 1,800/day at list price. Financially dangerous.
**Recommended solution:** Geocode only on ≥25 m movement **and** ≥10 s; round to 5 decimals; client cache by rounded key; stop while the sheet is expanded or the user is typing; cap at one call per 10 s.
**Migration phase:** P3 (must land before Google Geocoding goes primary)

### C-02
**Issue:** Driver→pickup route polled every 15 s while waiting.
**Severity:** High
**Location:** `mobile/src/screens/TaxiScreen.js:1092-1095`; key precision `googleMaps.js:24`; pre-warm `:1067`; `RideDetailScreen.js:143`
**Problem:** Fixed cadence regardless of movement; 5-decimal cache keys never hit.
**User impact:** None.
**Performance impact:** None.
**Google API cost impact:** 240 `computeRoutes` per hour per waiting passenger; at 10,000 rides/day with a five-minute wait ≈200,000 calls/day ≈ USD 1,000/day (Essentials) or 2,000/day (traffic-aware). Financially dangerous.
**Recommended solution:** Server computes the driver→pickup ETA on a movement/time trigger from the location it already receives and pushes it; client never polls (Section 19.6).
**Migration phase:** P5 (must land before Routes goes primary)

### C-03
**Issue:** One route call per stop for leg ETAs.
**Severity:** Medium
**Location:** `mobile/src/screens/TaxiScreen.js:1756-1782`
**Google API cost impact:** N extra calls per multi-stop quote.
**Recommended solution:** Use `legs[].duration` from the single `computeRoutes` response with `intermediates`.
**Problem / User impact / Performance:** Redundant calls; none; none.
**Migration phase:** P4

### C-04
**Issue:** Up to three reverse geocodes on cold start and one per foreground.
**Severity:** Medium
**Location:** `mobile/src/screens/TaxiScreen.js:1914,1841,594`
**Recommended solution:** One geocode after the best fix; skip if <20 m from the last geocoded point.
**Problem / User impact / Performance / Cost:** Duplicate calls; none; none; 2 to 3× per launch.
**Migration phase:** P3

### C-05
**Issue:** Navigation leg fetched twice (socket session + HTTP), reroutes twice, RideDetail fetches a third non-traffic route; nav cache keyed on the moving origin.
**Severity:** High
**Location:** `server/services/navigation.service.js:132-136`; `server/controllers/maps.controller.js:61-71`; `mobile-driver/src/services/directions.js:95`; `useNavigationEngine.js:419,242-277`; `mobile-driver/src/services/googleMaps.js:20-50`; `routing.service.js:97-101`
**Google API cost impact:** 2 to 3× `computeRoutes` (Pro tier with traffic) per leg; reroute storms unbounded (B-31).
**Recommended solution:** One route owner per leg; reroute back-off; RideDetail reuses `activeRoute`.
**Problem / User impact / Performance:** Duplicate spend; flicker; minor.
**Migration phase:** P4/P7

### C-06
**Issue:** `departure_time=now` on every Google Directions call, including cached free-flow quotes.
**Severity:** Medium
**Location:** `server/providers/google.provider.js:82-85`; matrix fallback at `:145-147` has no `departure_time` (inconsistent)
**Google API cost impact:** Advanced/Pro tier billing for quotes that are then cached five minutes and treated as free-flow.
**Recommended solution:** Routes `TRAFFIC_UNAWARE` for quotes and matrices used for pricing; `TRAFFIC_AWARE_OPTIMAL` only for nav and live ETA.
**Problem / User impact / Performance:** 2× price for the same answer; none; none.
**Migration phase:** P4

### C-07
**Issue:** `/maps/directions` and `/maps/distance-matrix` accept unbounded input from any authenticated user; limiter is per worker; `/locations/*` has no limiter.
**Severity:** High
**Location:** `server/controllers/maps.controller.js:51-56,105-113`; `server/routers/maps.router.js:10-49`; `server/routers/locations.router.js:13`
**Google API cost impact:** A 25×25 matrix is 625 billed elements per request, 30 requests/min/user/worker; unbounded waypoints.
**Recommended solution:** Cap waypoints ≤5 (passenger) and matrix ≤1×10; validate finite coords inside a Georgia bbox; Redis-backed limiter; restrict `/distance-matrix` to driver/admin roles.
**Problem / User impact / Performance:** Abuse vector; none; large OSRM tables time out.
**Migration phase:** P1

### C-08
**Issue:** No negative caching for geocoding or place details; no circuit breaker.
**Severity:** Medium
**Location:** `server/services/geocoding.service.js:52-55,69-72,94-97,108-111`; `autocomplete.service.js:284-290`; `cache.service.js:252-263` (health recorded, never read)
**Google API cost impact:** Every repeat of a zero-result query or bogus place id is billed again.
**Recommended solution:** Cache empty/null for 5 min as autocomplete already does (`autocomplete.service.js:242`).
**Problem / User impact / Performance:** Repeated billed misses; slower typo handling; 8 s timeouts stack.
**Migration phase:** P3/P8

### C-09
**Issue:** Admin ride creation calls Places Autocomplete and Details without session tokens; browser Directions on every render of admin/share maps.
**Severity:** High
**Location:** `client/src/pages/admin/AdminCreateRide.jsx:88-96,121-123`; `client/src/components/RouteMap.jsx:57-93`; `AdminRides.jsx:82-96`; `SharedRide.jsx:127-135`; `web/.../SharedRideView.jsx:156-170`
**Google API cost impact:** Per-request autocomplete billing plus a full Details call per selection; uncached Directions per map open, all outside the server's cache and cost dashboard.
**Recommended solution:** Route through `/maps/autocomplete`, `/maps/place-details`, `/maps/directions`; or at minimum reuse `client/src/hooks/usePlacesAutocomplete.js` with tokens.
**Problem / User impact / Performance:** Highest per-keystroke cost pattern in the codebase; none; none.
**Migration phase:** P2/P4

### C-10
**Issue:** Google Roads calls for a value nobody reads; snap endpoint uncached.
**Severity:** Medium
**Location:** `server/services/roadSnap.service.js:24-45` (`FF_PICKUP_ROAD_SNAP`); `maps.controller.js:204-227`; `cache.keys.snap` unused
**Google API cost impact:** One Roads call per ride, wasted; proxy endpoint billed per request with no dedupe.
**Recommended solution:** Delete both.
**Problem / User impact / Performance:** Pure waste; none; none.
**Migration phase:** P1

### C-11
**Issue:** Dispatch ETA per-pair cache keyed on the driver's live position at 111 m; sequential reads.
**Severity:** Low
**Location:** `server/services/etaDispatch.service.js:73-75,117-121`
**Google API cost impact:** Low hit rate for moving drivers.
**Recommended solution:** Coarser origin bucket (~250 m), pickup-centred keys, `MGET`.
**Problem / User impact / Performance:** Cache rarely saves elements; none; minor.
**Migration phase:** P8

### C-12
**Issue:** Quote screen issues 2 route calls per vehicle class tab.
**Severity:** Low
**Location:** `server/controllers/ride.controller.js:3039-3052`; client `useRideQuote`
**Google API cost impact:** N+1 `computeRoutes` per quote screen.
**Recommended solution:** One endpoint returning all classes; one trip route; one 1×K matrix for nearest drivers across classes.
**Problem / User impact / Performance:** Redundant calls; latency; minor.
**Migration phase:** P4/P8

### C-13
**Issue:** Client prefix-filter cache hides better results and substring-matches unrelated items.
**Severity:** Medium (quality), Low (cost)
**Location:** `mobile/src/services/googleMaps.js:100-107,268-277`
**Recommended solution:** Keep the exact-hit LRU; drop prefix filtering (session-priced autocomplete makes extra keystrokes cheap).
**Problem / User impact / Performance / Cost:** "rustaveli 1" never sees house-number predictions; wrong picks; none; slight increase in requests, offset by session pricing.
**Migration phase:** P2

### C-14
**Issue:** Untracked spend: Distance Matrix and Roads bypass the metrics layer; no Mapbox counter.
**Severity:** Medium
**Location:** `server/services/etaDispatch.service.js:41-42,148-155`; `roadSnap.service.js:15-32`; `services/metrics.service.js:61-67`; `client/src/pages/admin/AdminCostMetrics.jsx:9-37`
**Google API cost impact:** The cost dashboard under-counts the largest recurring Google line (dispatch matrix) and misses Roads entirely.
**Recommended solution:** All Google calls through `google.provider` with `metrics.apiCall.*` and `withHealth`; per-SKU counters matching Google's SKU names.
**Problem / User impact / Performance:** Blind spot; none; none.
**Migration phase:** P1/P8

## 13. Security and compliance problems

### S-01
**Issue:** Browser Google key must carry Maps JS, Places JS, and Directions JS, so it can only be referrer-restricted.
**Severity:** High
**Location:** `client/src/lib/googleMaps.js:13-21` and the seven loaders; `web/.env.example:14-17`
**Current implementation:** One `VITE_GOOGLE_MAPS_API_KEY` / `NEXT_PUBLIC_GOOGLE_MAPS_API_KEY` used for map loads, Places, and Directions from the browser.
**Problem:** Referrer restriction is trivially spoofed by non-browser callers; nothing caps spend per API.
**User impact:** None until abuse.
**Performance impact:** None.
**Google API cost impact:** Unbounded exposure on the most expensive SKUs.
**Recommended solution:** Now: referrer-restrict to the production origins and set per-API daily quotas. After P2/P4: move Places and Directions behind the server; API-restrict the browser key to Maps JavaScript API only.
**Migration phase:** P1 (quotas), P2/P4 (restriction)

### S-02
**Issue:** Server key is read once at load and used by three separate clients; API restriction cannot be verified from code.
**Severity:** Medium
**Location:** `server/providers/google.provider.js:26`; `etaDispatch.service.js:41`; `roadSnap.service.js:15`
**Recommended solution:** Lazy read (as `mapbox.provider.js:30`); one client; in Cloud Console restrict the server key by **IP address** to the API hosts and by **API** to Routes API, Places API (New), Geocoding API only; separate keys per environment.
**Problem / User impact / Performance / Cost:** Rotation needs restart; none; none; none.
**Migration phase:** P1

### S-03
**Issue:** Mobile apps will need Maps SDK keys; today none exist and the README references a variable nothing reads.
**Severity:** Medium (planning)
**Location:** `mobile/app.config.js`, `mobile-driver/app.config.js`, `mobile-manager/app.config.js` (no `android.config.googleMaps.apiKey` / `ios.config.googleMapsApiKey`); `mobile-driver/README.md:56`
**Recommended solution:** Three Android keys restricted by package name (`com.lulini.mobile`, `com.lulini.driver`, `com.lulini.manager`) + release and debug SHA-1, three iOS keys restricted by bundle id, all API-restricted to Maps SDK for Android/iOS only. Inject via EAS secrets, never `EXPO_PUBLIC_*`.
**Problem / User impact / Performance / Cost:** None yet; none; none; a leaked SDK-only key cannot call web services.
**Migration phase:** P6

### S-04
**Issue:** Mapbox public token embedded in all three bundles; cannot be bound to app identity.
**Severity:** Medium
**Location:** `mobile/app.config.js:106`, `mobile-driver/app.config.js:88`, `mobile-manager/app.config.js:83`, each `index.js`
**Recommended solution:** Keep scopes minimal until cut-over; revoke with P9.
**Problem / User impact / Performance / Cost:** Tile quota theft; none; none; Mapbox bill.
**Migration phase:** P9

### S-05
**Issue:** `/locations/search` and `/locations/reverse` are unlimited and unvalidated; `/maps/*` params unvalidated.
**Severity:** High
**Location:** `server/routers/locations.router.js:13,42-73`; `maps.controller.js:145-154,174-182` (`countryCode`, `radius`, `input` length, NaN coords)
**Recommended solution:** Redis limiter; allowlist `countryCode`; clamp radius; cap input length ~200; reject non-finite or out-of-bbox coordinates.
**Problem / User impact / Performance / Cost:** Any account can drive billable calls; queued searches; unbounded outbound; direct spend.
**Migration phase:** P1

### S-06
**Issue:** Shared "known places" are poisonable by any passenger.
**Severity:** High
**Location:** `server/controllers/ride.controller.js:67-77,280-282,371-373`; `autocomplete.service.js:108-114,131`; `places.service.js:133-150`
**Current implementation:** Every ride upserts `Place{canonicalId:'manual:<lat4>,<lng4>', address:<client string>}` ranked first for everyone; stops bypass the length validator; `$set` overwrites the address with the latest wording.
**Problem:** Spam or offensive suggestions for all users; duplicates of one POI under `goog:` and `manual:`.
**User impact:** Trust and quality.
**Performance impact:** Collection growth.
**Google API cost impact:** Neutral.
**Recommended solution:** Only upsert places with a verified Google place id; validate stops like pickup; mark `manual:` docs non-suggestable.
**Migration phase:** P1 (validation), P2 (place id)

### S-07
**Issue:** Google Place Details content stored indefinitely in Mongo and 30 days in Redis; recent list stores derived names with no expiry.
**Severity:** High (ToS)
**Location:** `server/services/autocomplete.service.js:284-290`; `places.service.js:68-95`; `place.model.js` (no TTL); `cache.service.js:29`; `recentLocations.service.js:11`
**Problem:** Google Maps Platform terms allow storing Place IDs indefinitely but limit caching of other Places/Geocoding content to 30 consecutive days.
**Recommended solution:** Keep `placeId`, `usageCount`, and coordinates chosen by the user; add `refreshedAt` and re-fetch or expire `address/name/components` after 30 days (TTL index); drop `components` from storage.
**User impact / Performance / Cost:** None; none; a small refresh cost.
**Migration phase:** P2

### S-08
**Issue:** Google content will be displayed on a non-Google map.
**Severity:** Critical (ToS, sequencing)
**Location:** All three mobile map wrappers (Mapbox); `LocationSearchSheet.js:740-747` attribution
**Problem:** Google's terms prohibit using Places, Geocoding, and Routes content with a non-Google map. Today Google is a fallback and the UI claims OpenStreetMap; once Google becomes primary this is a live breach unless the renderer is Google. Predictions shown in a list without a Google map require the "Powered by Google" attribution.
**Recommended solution:** Ship P6 (Google Maps SDK rendering) with or before P2/P3 production cut-over; attribution driven by provider until then.
**User impact / Performance / Cost:** None; none; none.
**Migration phase:** P6 gate for P2/P3

### S-09
**Issue:** Public OSRM demo and public Nominatim used in production; browser Nominatim without identifying UA; Expo Go scrapes OSM tiles.
**Severity:** Critical (policy) today
**Location:** `server/providers/osrm.provider.js:17`; `nominatim.provider.js:22`; `mobile/src/services/googleMaps.js:229-257`; `client/src/pages/admin/AdminCreateRide.jsx:46-48`; `mobile-manager/src/components/map/MapViewWrapper.js:170`
**Recommended solution:** Replaced by Google in P3/P4; delete the direct paths in P1 where trivial (Expo Go tiles, browser Nominatim).
**Problem / User impact / Performance / Cost:** IP bans would take down search and quotes; none; none; none.
**Migration phase:** P1/P3/P4/P9

### S-10
**Issue:** Driver auth token persisted in plaintext AsyncStorage by an unused helper.
**Severity:** Medium
**Location:** `mobile-driver/src/services/RideTrackingService.js:86-88`; `rideStorage.js:72-80` (`readRideConfig` has no callers)
**Recommended solution:** Delete `persistRideConfig`/`readRideConfig`; the background task already reads the token from SecureStore (`backgroundLocation.js:172`).
**Problem / User impact / Performance / Cost:** Token exposure on rooted devices and backups; none; none; none.
**Migration phase:** P1

### S-11
**Issue:** Driver location limiter is per IP; arrive/complete gates are spoofable.
**Severity:** Medium
**Location:** `server/middlewares/rateLimiter.js:72-79` (no `keyGenerator`); see B-12
**Recommended solution:** Key by driver id; server-owned proximity from Redis history.
**Problem / User impact / Performance / Cost:** Carrier-NAT drivers share a bucket; dropped updates; none; none.
**Migration phase:** P1

### S-12
**Issue:** Privacy pages and UI attribution are inaccurate.
**Severity:** Low (legal hygiene)
**Location:** `client/src/pages/Privacy.jsx:43`; `web/src/components/pages/PrivacyPage.jsx:68` ("We use Google Maps for routing and geocoding"); `LocationSearchSheet.js:740-747` ("Powered by OpenStreetMap")
**Recommended solution:** Align with the provider set at each phase; final text after P9.
**Problem / User impact / Performance / Cost:** Mis-attribution; none; none; none.
**Migration phase:** P1/P9

## 14. Duplicated and dead code

### D-01
**Issue:** Three divergent copies of the map wrapper layer.
**Severity:** High (migration multiplier)
**Location:** `mobile/src/components/map/*`, `mobile-driver/src/components/map/*`, `mobile-manager/src/components/map/*` (see Section 8 for the diff summary)
**Recommended solution:** One shared package (`packages/maps` or a workspace module) exposing the react-native-maps-shaped API; port once to Google.
**Problem / User impact / Performance / Cost:** Every fix lands in one app; driver copy lacks the NaN guard; none; none.
**Migration phase:** P1

### D-02
**Issue:** Dead server modules and flags.
**Severity:** Low
**Location:** `server/services/matching.service.js` (no callers); `server/models/location.model.js` (unreferenced); `server/services/navigation.service.js:220-297` (`processPositionUpdate`); `server/services/roadSnap.service.js` (writes an unread field); `server/utils/featureFlags.js:25,28` (`FF_GEOCODING_CACHE`, `FF_CANONICAL_LOCATIONS` never read); `cache.keys.snap`; `pricing.service.js:86-94,192-231` (duplicate haversine, unused validators); inline haversines `ride.controller.js:188-197,800-806`; `socket/middleware/rateLimiter.js:9` (`user:locationUpdate` limit with no handler); `NavigationSession.js:52`, `Geofence.js:32` (2dsphere indexes never queried); `worker.js:40-92` duplicating `app.js:396-457` expiry jobs; stale header comments in `google.provider.js:3-9`, `geocoding.service.js:9-10`; `cache.service.js:189` health list omits `mapbox`
**Recommended solution:** Delete; one process for expiry jobs.
**Problem / User impact / Performance / Cost:** Misleading maintainers; none; duplicate jobs; Roads spend.
**Migration phase:** P1/P9

### D-03
**Issue:** Dead passenger-app modules and legacy shims.
**Severity:** Low
**Location:** `mobile/src/components/map/MarkerWrapper.js`, `AnimatedMarkerWrapper.js`, `BoltPinSymbol.js` (no importers); `mapStyle.js` exports beyond `ROUTE_STYLE*` (keep `mapStyle`/`mapStyleDark` for P6); `markerImages.destination/pickup/dropoff/stop/stopSmall` registered and unused; `utils/mapSafety.js` vs `TaxiScreen.js:62-71`; `services/googleMaps.js:296-297,373,377-390,393-397` (aliases, unused `decodePolyline`, stubs); `TaxiScreen.js:137` (`EMPTY_MAP_STYLE`), `:248`; `MapViewWrapper.js:284-286` (always-true ternary); `RideOptionsSheet.js:58,271` (`onScheduleRide` never wired, so the schedule feature is unreachable from TaxiScreen); `__mocks__/expo-location.js` incomplete
**Migration phase:** P1/P9

### D-04
**Issue:** Dead driver-app modules and dependencies.
**Severity:** Low
**Location:** `mobile-driver/src/services/roadSnapping.js` (+ test); `services/googleMaps.js:52` (`getDirectionsOSRM` alias); `directions.js:29-32,212` (`invalidateRouteToken`, `clearRouteCache` never called); `components/map/mapStyle.js` (keep for P6); `RideTrackingService.js:267-280` (`subscribe`, no subscribers), `:194-203` (`qualityTier`, never consumed); `rideStorage.js:63-65,82-101`; `api.js:72`; `LocationContext.js:74,355-365` (`address`); `HomeScreen.js:792-812` (Platform split with an Apple-Maps rationale); `MapViewWrapper.js:76-92` (legacy props); `package.json:24-27,51` (`@turf/*`, `zustand` unused); `README.md` and `LOCATION_TROUBLESHOOTING.md` (pre-Lulini "GoTours", fictional `driver:location` socket event)
**Migration phase:** P1/P9

### D-05
**Issue:** Dead manager-app and admin modules.
**Severity:** Low
**Location:** `mobile-manager/src/components/map/mapStyle.js`, `AnimatedMarkerWrapper.js`, `PolylineWrapper.js` (no importers); `services/api.js:231-233` (`mapsAPI` unused); `MapViewWrapper.js:428-430`; `client/src/App.jsx:55` (`libraries` unused); seven duplicated Maps JS loaders (`client/src/lib/googleMaps.js:15-26`, `AdminRides.jsx:55-61`, `AdminLiveMap.jsx:24-35`, `AdminSOS.jsx:13-19`, `AdminDriverInfo.jsx:63-69`, `AdminCreateRide.jsx:139-150`, `SharedRide.jsx:33-39`, `BoltDriversPanel.jsx:26-32`); `scripts/generate-houselabel-bg.js` (Mapbox-only)
**Migration phase:** P1/P6/P9

### D-06
**Issue:** Documentation contradicts the code and the migration decision.
**Severity:** Low
**Location:** `NAVIGATION_SYSTEM_HANDOFF.md:56-58` (lists "switching the map renderer" as a non-goal), `:124` vs `:17,339` (Mapbox provider existence); `LAUNCH_TEST_CHECKLIST.md:128` ("server-proxied Mapbox" autocomplete never existed), `:145,580` (`tracksViewChanges` is dropped by the wrappers); phase comments referencing the gitignored `MAPS_API_OPTIMIZATION_PLAN.md` (`autocomplete.service.js:41,139,172,186,240`, `nominatim.provider.js:27`, `mobile/src/services/googleMaps.js:45`, `AnimatedMarkerWrapper.js:19` → missing `spike/DriverCarSpike`)
**Recommended solution:** Check the plan into the repo or remove phase references; update the handoff's non-goals; this document supersedes both for map matters.
**Migration phase:** P1

### D-07
**Issue:** Test mocks drifted from the code and coverage gaps around exactly the files the migration touches.
**Severity:** Medium
**Location:** `server/__tests__/integration/maps.controller.test.js:16-20` (mocks `snapToRoad`, real name `snapToRoads`; `resolvePrediction` shape `{lat,lng}` vs real `{coords:{lat,lng}}`); no tests for `autocomplete.service` merge/tiering, `geocoding.service` fallback, `places.service`, `cache.service` keys, `recentLocations`, favorites, `useNavigationEngine`, `LocationContext`, background task, wrappers; passenger tests cover only `googleMaps`, `useRideQuote`, `rideStorage`, `api`
**Recommended solution:** Write contract tests against the unified shapes before swapping providers (Section 20, P1 exit criteria).
**Migration phase:** P1

---

# Part C. What to keep, what to replace, where things belong

## 15. What is already implemented correctly

Keep these; they are the foundation the migration builds on.

**Server**
- Server-side proxy for every web-service provider; keys never leave the server; provider errors do not leak URLs (`google.provider.js:52-56`); tests assert key absence (`maps.controller.test.js:64-69`).
- Unified provider contract (`{distanceMeters, durationSeconds, polyline, provider}`, `UnifiedPlace`, prediction shape) so provider swaps are confined to `providers/*.js`.
- One cache policy module with NFC-normalised lowercase queries, 4-decimal coordinate rounding, 1-decimal bias rounding, negative caching for autocomplete, Redis-only with silent degradation (`cache.service.js`).
- Session tokens plumbed end to end (client UUID → autocomplete + details) with the cache key deliberately excluding the token.
- Country restriction and `language=ka` defaults; Details field mask limited to basic fields.
- Dispatch cost controls: Redis GEO shortlist, top-3 matrix cap, per-pair cache with in-flight coalescing, blocklist applied before the paid call, two-tier ranking (`etaDispatch.service.js`).
- Offer serialisation: per-driver `SET NX` lock with CAS release, `MGET` pre-filter, pub/sub wait with decline fast path, cluster-wide in-flight breaker (`driverDispatch.service.js`, `rideDispatchPubsub.js`).
- Atomic accept/complete/cancel transactions and a unique partial index on active rides (`ride.model.js:345-354`).
- Driver location: pipelined `MULTI` writes, eviction of non-dispatchable drivers, `RENAME`-based dirty-set flush with retry, stale GEO cleanup, implausible-speed rejection (`driverLocation.service.js`, `driver.controller.js:598-602`).
- Navigation: ride and session ownership asserted server-side, reroute rate limit, leg resolution with stops, JWT-authenticated sockets, nav handlers registered only for driver sockets.
- Pricing intent: server recompute-and-override with live surge; client price logged only (the bypass in B-01 is a validator bug, not a design flaw).
- `queueUpsertPlace` bulk buffer; `nearby-popular` free empty state; metrics and provider-health counters with an admin cost dashboard.
- Polyline precision handled per provider (polyline6 vs polyline5) and unit-tested.

**Passenger app**
- Thin proxy client (`services/googleMaps.js`) with no embedded Google key.
- Autocomplete ordering guard, debounce, frozen snapshot on submit; `useRideQuote` with debounce, `AbortController`, rounding, and 30 s cache is the template for every network hook.
- GPS watch disabled in RIDE_OPTIONS and IN_PROGRESS to prevent quote churn.
- Driver-update ingest: rideId guard, monotonic timestamp guard, jitter filter, 2 s throttle with trailing flush, freshness watchdog with dimmed marker; car marker isolated from the pin tree.
- Route on `ride:started` comes from the server's `activeRoute`; idempotency key on ride creation; offline handling and ride-state reconciliation on reconnect.
- `SavedPlaceEditorScreen` reverse-geocode ordering and programmatic-move suppression are the reference implementation for pickup selection.
- `DriverCluster` grid clustering and the marker/driver/route/cluster memo split are renderer-agnostic.

**Driver app**
- Single GPS watcher feeding the navigation engine; route fetch keyed on `(ride, phase, destination)` never per tick; `to_dropoff` reuses the server-prefetched `activeRoute`.
- Route projection math (windowed perpendicular projection, along-route step advance, congestion-aligned trimming, heading smoothing) is correct and has 20 unit tests.
- Off-route hysteresis, reroute cooldown, version-ordered route application, GPS-lost detection gated on last speed.
- Voice guidance with de-duped buckets and a Georgian capability probe.
- Background pipeline fundamentals: module-scope `defineTask`, disk-backed ride state, batching with `AbortController` timeouts and 401 handling, iOS significant-change re-arm.
- Spoof and speed validation on both client paths; speed-adaptive camera with slew limiting; pan cancels follow mode; pre-registered marker bitmaps; `mapSafety` guards.

**Manager app**
- Image markers via one registry, value-comparing `memo`, per-car memoised components, reconciled poll results, native circle halos, fit-once with explicit recentre, simplification above 80 points, coordinate hygiene at every boundary, PII-aware realtime design, theme plumbing that is one token per palette.

## 16. What should be replaced by Google

| Today | Replace with | Notes |
|---|---|---|
| Mapbox GL rendering (3 apps) | Maps SDK for Android/iOS via `react-native-maps` `PROVIDER_GOOGLE` + Cloud-based map styling (`mapId`) | Only way to display Google content compliantly; enables native `tracksViewChanges`, `Marker.Animated`, native `onRegionChangeComplete`. |
| Nominatim autocomplete + Google legacy Places | Places API (New) `places:autocomplete` with session tokens, `includedRegionCodes:["ge"]`, `locationBias`, `languageCode`; `places/{id}` with an Essentials field mask | Keep the Mongo warm layer only for place ids and usage counts. |
| Nominatim geocoding + Google legacy Geocoding | Geocoding API (same API, tightened parameters) | Add `result_type`, `location_type`, one `language`. |
| OSRM route/table, Mapbox Directions, Google legacy Directions/Distance Matrix | Routes API `computeRoutes` and `computeRouteMatrix` | `TRAFFIC_UNAWARE` for pricing, `TRAFFIC_AWARE_OPTIMAL` for nav and live ETA; `intermediates` for stops; `polylineEncoding: ENCODED_POLYLINE`; `extraComputations: TRAFFIC_ON_POLYLINE` for colouring. |
| Google Roads snap | Nothing | On-device projection already exists. |
| Client-side straight-line ETAs (30 and 24 km/h) | Server-pushed Routes-based ETA | Section 19.6. |
| Wialon geocoder (fleet panel) | Geocoding API (later) | Out of ride-hailing scope. |

## 17. What should remain client-side

- Rendering the Google map, markers, polylines, circles, and heat polygons.
- Animating the driver car between server updates (interpolation, bearing, route snapping against the last known route geometry).
- Autocomplete UI, debounce, request ordering, session-token generation, exact-hit result cache.
- Obtaining device GPS with mode-appropriate accuracy; client-side plausibility filtering (mocked, accuracy, speed).
- Driver navigation engine: route projection, step advancement, off-route detection, voice, local ETA decrement between server refreshes, camera control.
- Polyline decoding and trimming of the travelled portion.
- Camera state machine and user-gesture handling.
- Display-only formatting of price, distance, and duration received from the server.

## 18. What should move (or stay) server-side

| Operation | Placement | Reason |
|---|---|---|
| Every Google web-service call (Places, Geocoding, Routes, Route Matrix) | Server | Key protection, caching, metering, quotas. Already true for mobile; **admin SPA must be moved** (C-09, B-38). |
| Quote and final fare | Server only | B-01, B-04. Client sends place ids and coordinates, receives a priced quote. |
| Route for the passenger's map (pickup → stops → destination) | Server, returned with the quote | One call serves price, ETA, and polyline. |
| Driver → pickup ETA while waiting, in-ride remaining ETA | Server, pushed over the socket on triggers | C-02, B-22; the server already receives every driver fix. |
| Dispatch candidate ranking | Server (`computeRouteMatrix`) | Already server-side; keep. |
| Turn-by-turn route computation | Server proxy (`computeRoutes` with steps) | Keep the client engine; make the server the single route owner per leg (Section 19.10). |
| Reverse geocoding of the pickup pin | Server proxy, client-throttled | C-01. |
| Place resolution and Place ID persistence | Server | Section 19.2. |
| Driver location ingestion, ordering, freshness | Server | Device timestamps end to end (B-33). |
| Driver ranking, offers, locks | Server | Keep. |
| Recent and saved places | Server | Store place ids. |
| Map rendering, marker animation, camera | Client | No server role. |

---

# Part D. Target architecture

## 19. Recommended Google Maps Platform architecture

### 19.1 Overview

```
 Passenger app                          Driver app                          Manager / Admin
 ┌──────────────────────┐              ┌────────────────────────┐          ┌──────────────────────┐
 │ Google Maps SDK      │              │ Google Maps SDK        │          │ Google Maps SDK /    │
 │ (react-native-maps,  │              │ (same shared package)  │          │ Maps JS API          │
 │  PROVIDER_GOOGLE,    │              │ nav engine (client):   │          │                      │
 │  mapId light/dark)   │              │  projection, steps,    │          │ all searches/routes  │
 │ Places UI + session  │              │  off-route, voice,     │          │ via server proxy     │
 │ tokens; camera FSM;  │              │  camera FSM            │          └──────────┬───────────┘
 │ car interpolation    │              │ single GPS pipeline    │                     │
 └──────────┬───────────┘              └───────────┬────────────┘                     │
            │ HTTPS: /maps/*, /rides/*             │ HTTPS: /drivers/location (ts, accuracy, speed)
            │ Socket: ride:*, driver:locationUpdate│ Socket: nav:* (server is route owner per leg)
            ▼                                      ▼                                  ▼
 ┌─────────────────────────────────────────────────────────────────────────────────────────────┐
 │ Taxi backend                                                                                │
 │  maps.controller ──► places.service (New)   ──► Places API (New): autocomplete, details     │
 │                  ──► geocoding.service      ──► Geocoding API                               │
 │                  ──► routing.service        ──► Routes API: computeRoutes                   │
 │  ride.controller ──► quote.service (routes + pricing + surge, authoritative)                │
 │  dispatch        ──► Redis GEO shortlist ──► computeRouteMatrix (≤ top 3..5) ──► rank/offer │
 │  eta.service     ──► trigger-based refresh (moved ≥300 m | Δt ≥180 s | off-route | phase)  │
 │                      pushes driver:eta to passenger; driver keeps local decrement           │
 │  location ingest ──► ordering by device ts, accuracy filter, Redis GEO/hash, 2 s emit gate  │
 │  cache.service   ──► Redis (routes 5 min / traffic 90 s / matrix 2 min / geocode 24 h /     │
 │                      details 30 d / autocomplete 24 h / negatives 5 min)                    │
 │  Place store     ──► Mongo: placeId (forever) + user-chosen coords + content refreshed ≤30 d │
 │  metrics         ──► per-SKU counters, quotas, alerts                                       │
 └─────────────────────────────────────────────────────────────────────────────────────────────┘
```

Design rules that fall out of the audit:

1. **One route owner per leg.** The server computes the leg route once (`computeRoutes`), stores the encoded polyline and steps on the ride, and pushes it to both apps. Clients never fetch a leg route independently.
2. **No client polling for ETA or routes.** ETA changes are pushed on server-side triggers.
3. **Place IDs travel with every location** the user picks; coordinates stay the source of truth for pricing and dispatch.
4. **Mobile apps hold only Maps SDK keys**, restricted to app identity and to the SDK API.
5. **Every Google call goes through `google.provider.js`** with metrics, health, timeout, and cache.

### 19.2 Place ID data model

Today rides, favorites, and recents store `{lat, lng, address}` only (`ride.model.js:3-40`, `favoriteLocation.model.js:5-38`, `recentLocations.service.js:45-54`); the warm `places` collection stores Google content with no TTL.

Recommended embedded `LocationRef` schema, used by ride pickup/dropoff/stops, favorites (home, work, custom), recents, hotel/transfer presets, and ride history:

```js
{
  placeId:        String,   // Google place id, nullable for pin-drops; storable indefinitely
  lat: Number, lng: Number, // user-confirmed coordinates: pricing/dispatch source of truth
  source:         'places' | 'geocode' | 'pin' | 'gps' | 'favorite' | 'recent' | 'admin',
  displayName:    String,   // prediction mainText or POI name shown to the user at selection time
  formattedAddress: String, // Geocoding/Details formatted address at selection time
  street: String, houseNumber: String, district: String, city: String, country: 'GE',
  resolvedAt:     Date,     // when Google content was fetched (drives the 30-day refresh)
}
```

Rules:
- Persist `placeId` + coordinates + the text the user saw. Do **not** persist `addressComponents` arrays.
- The `places` warm collection keeps `placeId`, `usageCount`, `lastUsedAt`, user-confirmed coordinates, and content fields with a `refreshedAt`; a TTL or job refreshes or blanks content older than 30 days. `manual:` and `osm:` ids are dropped; only Google ids are suggestable (S-06, S-07).
- "Ride again", favorites, and recents send `placeId` + coordinates to the quote endpoint; the server never re-resolves a place it already has coordinates for. Re-resolution happens only when a favorite has no coordinates (legacy rows) or content is older than 30 days and the user opens the editor.
- Recents dedupe on `placeId` (else 4-decimal coordinates), 180-day TTL, deleted with the account.
- Favorites: partial unique index on `{user, type}` for `home`/`work`; cap enforced atomically.
- Hotel and transfer presets (admin-managed) are `LocationRef` documents with a Google `placeId` and admin-verified coordinates; airports and landmarks come from Places (New) with `includedPrimaryTypes` when searched.

### 19.3 Autocomplete (Places API New)

Client (passenger `LocationSearchSheet`, admin `AdminCreateRide`, future manager dispatch):
- One session token per search box focus, rotated after a Details resolution (already done) **and** when the sheet closes without a selection; the same token is reused across pickup/stop/destination inputs only within one focus.
- Min 3 characters; 350 ms debounce; `AbortController` per query; request id ordering (keep `searchIdRef`); exact-hit LRU only (drop prefix filtering, C-13).
- Bias point = map centre or selected pickup, rounded to 3 decimals and memoised (never raw GPS per tick, R-04).
- Send `language` from `i18n.language`.
- Show "Powered by Google" whenever predictions are displayed off a Google map; not needed once the list sits over the Google map.

Server (`autocomplete.service.js` → `google.provider.autocompleteNew`):
- `POST https://places.googleapis.com/v1/places:autocomplete` with `{ input, sessionToken, languageCode: 'ka', regionCode: 'GE', includedRegionCodes: ['ge'], locationBias: { circle: { center, radius: 30000 } }, origin: <bias point> }`. Default the bias to the operating city from `server/utils/city.js` when the client sends none (B-39). Use `locationRestriction` (rectangle around the service area) for pickup search; use bias only for destinations so inter-city trips still resolve.
- Do not pass `includedPrimaryTypes` by default; Places (New) already returns addresses, POIs, hotels, airports, and businesses together. Offer a typed search only for explicit chips (airport, hotel) via `includedPrimaryTypes: ['airport']` etc.
- Response mapping keeps the existing prediction shape: `placeId: 'goog:' + placePrediction.placeId` (or drop the prefix once other providers are gone), `mainText = structuredFormat.mainText.text`, `secondaryText`, `types`, `distanceMeters`.
- Cache key unchanged (query | bias@0.1° | language), TTL 24 h populated / 5 min empty (within the 30-day content allowance).
- Place Details: `GET https://places.googleapis.com/v1/places/{id}` with `X-Goog-FieldMask: id,location,formattedAddress,shortFormattedAddress,addressComponents,types,viewport` and `sessionToken`. Do not request `displayName` (Pro tier); use the prediction's `mainText` as the display name.
- Mongo warm layer: keep as a **place-id cache** (skip Details when coordinates are known and content is fresh), not as a text-search provider; drop `$text` search or index `normalizedAddress` (P-11).
- Georgian coverage check before cut-over: run the top 500 historical destination strings and 200 Kutaisi street + house-number queries through both stacks and compare hit rate and coordinate distance; this decides whether a Mongo-backed house-number layer must remain.

### 19.4 Geocoding

Server (`geocoding.service.js` → `google.provider`):
- Reverse: `latlng`, `language=ka`, `region=ge`, `result_type=street_address|premise|subpremise|route`, prefer `ROOFTOP`/`RANGE_INTERPOLATED`, fall back to the first result. Return a provider-neutral shape `{placeId, mainText, secondaryText, formattedAddress, street, houseNumber, district, city, lat, lng}` so clients stop parsing components (B-41).
- Forward: only for legacy favorites without coordinates and admin tools; `components=country:GE`.
- One language code; cache keys at 4 decimals (≈11 m) for reverse, 24 h; negative cache 5 min.
- Validate coordinates as finite and inside a Georgia bounding box before any call.

Client:
- Reverse geocode **only** on `onRegionChangeComplete` (native, once per settle) in pin-adjust and drag modes, and on the first good GPS fix. Never on the GPS watch timer (C-01).
- Movement gate ≥25 m **and** ≥10 s from the last geocoded point; round to 5 decimals; exact-key client cache (50 entries).
- One `useReverseGeocode` hook with request id + `AbortController` (from `SavedPlaceEditorScreen`) used by all four pickup mechanisms (R-01, R-02).
- Address label updates immediately with "Locating…" and the pin moves with the map; the label resolves when the request returns and only if it is the latest.

### 19.5 Routes (Routes API)

`routing.service.js` keeps its contract; `google.provider` gains `computeRoutes` and `computeRouteMatrix`; OSRM and Mapbox providers are deleted in P9.

| Workflow | Call | Preference | Field mask (routes.*) | Cache |
|---|---|---|---|---|
| Quote / booking (pickup → stops → destination) | `computeRoutes` with `intermediates` | `TRAFFIC_AWARE` (Pro) for the displayed ETA; distance is preference-independent | `distanceMeters, duration, staticDuration, polyline.encodedPolyline, legs.distanceMeters, legs.duration` | `route:` 5 min at 4 decimals |
| Nearest-driver ETA on the quote screen | `computeRouteMatrix` 1×K (K ≤ 3 nearest across eligible classes) | `TRAFFIC_AWARE` | `originIndex, destinationIndex, duration, distanceMeters, condition` | 2 min |
| Dispatch ranking | `computeRouteMatrix` N×1, N ≤ 3 (raise to 5 only with evidence) | `TRAFFIC_AWARE` | same | 3 min per pair (existing) |
| Driver nav leg (to pickup, to dropoff via stops) | `computeRoutes` with `intermediates`, `origin.heading`, `computeAlternativeRoutes: false` | `TRAFFIC_AWARE_OPTIMAL` (Pro) | + `legs.steps.navigationInstruction, legs.steps.polyline.encodedPolyline, legs.steps.distanceMeters, legs.steps.staticDuration, travelAdvisory.speedReadingIntervals` with `extraComputations: ['TRAFFIC_ON_POLYLINE']` | `route:nav` keyed on `(rideId, phase, rerouteCount)`, not on the moving origin |
| Reroute | same as nav leg | same | same | none |
| In-ride ETA refresh | `computeRoutes` origin = driver, destination = next stop/dropoff, no steps | `TRAFFIC_AWARE` | `duration, distanceMeters` | none (trigger-gated) |
| Scheduled ride re-quote at T-10 min | `computeRoutes` with `departureTime` | `TRAFFIC_AWARE` | quote mask | none |
| Admin / share pages | server `/maps/directions` | `TRAFFIC_UNAWARE` | quote mask | 5 min |

Common parameters: `travelMode: DRIVE`, `languageCode: 'ka'`, `regionCode: 'GE'`, `units: METRIC`, `polylineEncoding: ENCODED_POLYLINE`, `polylineQuality: OVERVIEW` for passenger lines and `HIGH_QUALITY` for nav. Transport **encoded** polylines end to end and decode on the client (the decoder already exists in `googleMaps.js:376-390`); store the encoded string on `Ride.activeRoute` (P-07).

Mapping to the existing step contract: `maneuver.type/modifier` from `navigationInstruction.maneuver` (e.g. `TURN_SLIGHT_LEFT` → `{type:'turn', modifier:'slight left'}`, `ROUNDABOUT_LEFT` → `{type:'roundabout', modifier:'left'}`), `name` from `navigationInstruction.instructions`; `congestion[]` derived by expanding `speedReadingIntervals` (`startPolylinePointIndex`, `endPolylinePointIndex`, `speed`) into per-segment `NORMAL|SLOW|TRAFFIC_JAM`. `maxspeed` is removed; `SpeedLimitSign` is deleted or gated.

Input hygiene at the proxy: waypoints ≤ 5, matrix ≤ 1×10 for passengers, coordinates inside the service bbox, all numeric.

### 19.6 ETA policy

One server-side `eta.service` owns every ETA a passenger sees; the driver keeps a local per-step decrement between refreshes (already implemented).

| ETA | Computed by | Refresh trigger |
|---|---|---|
| Quote screen: nearest driver → pickup, trip | `computeRouteMatrix` 1×K, `computeRoutes` | per quote request (debounced client, 30 s cache) |
| Dispatch ranking | `computeRouteMatrix` | per dispatch |
| Driver → pickup after accept | `computeRoutes` origin = driver position, no steps | on accept; then when the driver has moved ≥300 m since the last computation **or** ≥180 s elapsed **or** the nav engine reported off-route **or** phase changed. Never per GPS tick. |
| In ride: remaining to next stop / dropoff | same | same triggers; plus destination or stop change |
| Between refreshes | server linear decay using last duration and elapsed time, corrected by along-route progress from the driver's last position projected onto the stored polyline | on every accepted location tick (no API call) |
| Arrival | server geofence ≤50 m along-route + speed < 2 m/s for 10 s, or driver tap | – |

Pushed to the passenger as part of `driver:locationUpdate` (`{…, etaSeconds, remainingMeters, etaSource, computedAt}`), gated to one emit per 2 s per ride. The passenger deletes its haversine ETA, the 15 s route poll, and the `osrmEtaPriorityRef` logic. Expected cost: 3 to 6 `computeRoutes` per ride (Section 19.14).

### 19.7 Dispatch

Keep the existing two-stage pipeline (`etaDispatch.service.js`) and swap the matrix call:

```
pickup → Redis GEOSEARCH per eligible class (15 km, 15 nearest)
      → drop engaged / blocked / stale (hash ts > 30 s old) / not 'online' by Redis hash (fixes B-05)
      → sort by straight-line
      → computeRouteMatrix(top 3 origins × 1 destination, TRAFFIC_AWARE) with 3 min per-pair cache
      → two-tier rank (real ETA first), offer loop (unchanged)
      → on exhaustion: next 3 (one more matrix) before broadcasting; never broadcast in parallel (B-02)
```

Route Matrix limits: ≤100 elements per request with traffic; our requests are 3 to 5 elements. Coalesce concurrent requests per pickup cell (existing in-flight map, moved to Redis for multi-worker). Cost: one Pro matrix request (3 elements) per ride request, plus ≤1 for the second wave.

For the quote screen, replace "nearest driver of the exact class → one route" with one 1×K matrix across eligible classes so "no driver available" is computed with the same hierarchy dispatch uses (B-11).

### 19.8 Map rendering (Google Maps SDK)

- One shared package `packages/maps` (workspace or git subtree) exporting `MapView`, `Marker`, `AnimatedMarker`, `Polyline`, `TrafficPolyline`, `Circle`, `HeatPolygons`, `useCameraController`, `mapSafety`, `polylineSimplify`, `polylineDecode`, `KUTAISI_CENTER`. All three apps import from it (D-01).
- `react-native-maps` with `provider={PROVIDER_GOOGLE}` on both platforms; `googleMapId` per theme (Cloud-based styling) replaces `customMapStyle` JSON and the Mapbox style state machine; house-number emphasis is configured in the cloud style, not in code.
- Markers: bitmap `image=` markers with `anchor`, `flat` and `rotation` for cars; `tracksViewChanges={false}` by default, flipped to `true` for one frame only when a label changes (P-02); `zIndex` ordering instead of layer ids.
- Polylines: `Polyline` with `strokeColors` (multi-colour) for traffic runs; travelled portion trimmed by index change only (P-04); passenger sees the encoded route decoded once per `ride:started`.
- Heat: `Polygon`s per surge cell.
- Manager Expo Go fallback: static placeholder only (S-09).
- Web/admin: one `useJsApiLoader` with `language:'ka'`, `region:'GE'`, `mapIds`; `AdvancedMarkerElement`; Places and Directions via the server proxy.

Render discipline checklist: no inline arrow props on markers; heading throttled and fed via ref; ETA text outside the marker tree; location context split into a stable controls context and a per-fix store with selectors (P-05).

### 19.9 Driver marker animation

Pipeline: server emit (device `ts`, `heading`, `speed`, `accuracy`, `etaSeconds`) → passenger ingest (rideId guard, device-`ts` monotonic guard, drop `accuracy > 50 m`, 5 m jitter floor) → route snap (project onto the stored leg polyline if lateral < 40 m; use snapped point and segment bearing) → queue.

Animation: a target queue rather than "cancel and restart". Each target animates over `clamp(Δts × 1.1, 600, 4000)` ms from the marker's **current interpolated position** (not the previous target), with ease-in-out; bearing interpolated along the shortest arc, capped at 90°/s, using segment bearing below 1.5 m/s. When the queue empties, dead-reckon along the polyline at the last speed for at most 3 s, then hold. Implementation: `Marker.Animated` + `AnimatedRegion.timing` on Android and `animateMarkerToCoordinate` on iOS, driven by the native animation system, replacing the 60 Hz JS `setNativeProps` loops (P-03, B-28). Stale marker dimming after 45 s stays.

### 19.10 Route polyline and route state

Single route state per app (`routeState = {legId, encodedPolyline, decoded, steps, congestionRuns, version, source:'server'}`), owned by a reducer:
- Set on `nav:route_computed` / `ride:started` / `ride:accepted` (server pushes the leg route in the accept payload for the passenger).
- Replaced only when `version` increases (reroute, phase change, destination change).
- Trimmed view derived from `segmentIndex` (driver) or the last snapped driver point (passenger); recomputed only when the index changes.
- Cleared on ride end. No module-level caches (`_cachedRoutePolyline`, `_cachedDriverRoute`, `_cachedFullRoute`, `_polylineCache`).

The driver app stops fetching leg routes over HTTP; `directions.js` remains for the RideDetail preview only and reads `ride.activeRoute`.

### 19.11 Off-route detection (driver)

Keep the projection engine and tighten the decision:

```
offRoute if lateral > max(40 m, 1.5 × accuracy) AND heading deviates > 45° from segment bearing (when speed > 3 m/s)
        for ≥ 2 consecutive fixes spanning ≥ 5 s
reroute: first immediately; then back-off 8 s → 16 s → 32 s (cap) while still off-route
reset:   lateral < 25 m for 2 fixes
request: origin with heading and accuracy; server-owned computeRoutes; version increments
ignore:  fixes with accuracy > 80 m; fixes with speed < 1 m/s within 60 m of the route (parked)
```

### 19.12 Camera modes

Replace the eight boolean refs with one state machine per screen:

| Mode | Enter | Camera behaviour | Exit |
|---|---|---|---|
| `pickupSelect` | search screen | centre on user once; follow user until first gesture | destination chosen |
| `adjustPin` | adjust tap | animate to pin once; no further programmatic moves | confirm/cancel |
| `routePreview` | route received | `fitToCoordinates(route + pins)` once with padding; refit only on route version change | ride requested |
| `driverApproaching` | accept | fit driver + pickup once; then follow driver with fixed zoom | arrival |
| `activeRide` | start | follow driver, heading-up optional | completion |
| `userControlled` | any gesture (`onPanDrag` / `onRegionChange` with `isGesture`) | no programmatic moves | recentre button or mode change |
| Driver `navFollow` / `overview` | engine | speed-adaptive zoom, pitch 50, one `animateCamera` per accepted fix (existing controller) | gesture / recentre |

`fitToCoordinates` is never called from a data effect; it is called by the mode transition. The recentre button re-enters the previous automatic mode.

### 19.13 Pickup selection

```
open map → last known fix (≤60 s) shown instantly → fresh fix → pin at user
user moves map → pin follows map centre (native), label "Locating…", haptic once per gesture end
onRegionChangeComplete (native, settle) → if moved ≥ 25 m from last geocoded point:
   reverseGeocode(rounded 5 dp) with request id N; response applies only if N is latest
confirm → { placeId?, lat, lng, displayName, formattedAddress, source:'pin' } → quote
```

Stale protection: request id + `AbortController`; confirm while resolving either waits (≤1.5 s) or sends coordinates with `displayName:'Selected location'` and lets the server label it. GPS-denied users can confirm a pin without a fix (B-20).

### 19.14 GPS and current location modes

| Mode | Accuracy | Interval / distance | Background | Upload |
|---|---|---|---|---|
| Passenger, search screen | Balanced (High only if accuracy > 50 m after 5 s) | 3 s / 10 m | none | none |
| Passenger, waiting / in ride | Balanced | 10 s / 25 m | none | none |
| Passenger, background | off | – | none | none |
| Driver offline | off; one fix on "go online" | – | task stopped | none |
| Driver online idle | Balanced | 10 s / 15 m; iOS `pausesLocationUpdatesAutomatically:false`, significant-change task registered | foreground service | every ≥10 s or ≥25 m, heartbeat 30 s |
| Driver accepted / arrived | High | 3 s / 5 m | foreground service | every ≥3 s or ≥10 m, heartbeat 20 s |
| Driver in ride | High (BestForNavigation on iOS) | 2 s / 5 m | foreground service | every ≥2 s or ≥8 m; breadcrumbs batched 20 points |

One pipeline per app: the driver app keeps only the TaskManager subscription (it delivers in foreground too) and fans out to the UI through an emitter; the profile table lives in one module; profile changes re-issue the subscription without stop/start gaps; watchdogs consider speed (B-25, B-26, B-27, P-06). Payload: `{lat, lng, heading, speed, accuracy, ts, seq}` on both PATCH and batch; server rejects `ts` older than the stored value and accuracy > 100 m.

### 19.15 API key security

| Key | Holder | API restriction | Application restriction |
|---|---|---|---|
| Server key (one per environment) | `server/.env` `GOOGLE_MAPS_API_KEY` | Routes API, Places API (New), Geocoding API | IP addresses of the API hosts |
| Android SDK keys (passenger, driver, manager) | EAS secret → `android.config.googleMaps.apiKey` | Maps SDK for Android only | package name + release SHA-1 (+ debug SHA-1 on a separate dev key) |
| iOS SDK keys (three) | EAS secret → `ios.config.googleMapsApiKey` | Maps SDK for iOS only | bundle identifier |
| Browser key (admin + web) | `VITE_/NEXT_PUBLIC_` | Maps JavaScript API only, after P2/P4 | HTTP referrers (`lulini.ge`, `www.lulini.ge`, admin host) |

Per-SKU daily quotas in Cloud Console at 3× the expected daily volume; billing alerts at 50/80/100 % of budget. Mapbox tokens revoked in P9. No `EXPO_PUBLIC_*` Google variables.

### 19.16 Cost model

List prices are the USD per-1,000 prices published for the March 2025 Google Maps Platform pricing change; confirm current values in the Cloud Console before budgeting. Free tier: 10,000 events per Essentials SKU, 5,000 per Pro SKU, 1,000 per Enterprise SKU per month. Maps SDK for Android/iOS map loads are free; Maps JavaScript dynamic map loads ≈ $7.

| SKU | Tier | ≈ price / 1,000 | Used for |
|---|---|---|---|
| Autocomplete Requests | Essentials | 2.83 | abandoned sessions only |
| Place Details Essentials | Essentials | 5 | selection (session-priced) |
| Geocoding | Essentials | 5 | pin reverse geocode |
| Compute Routes Essentials | Essentials | 5 | admin/share previews |
| Compute Routes Pro | Pro | 10 | quotes with traffic, nav legs, reroutes, ETA refresh |
| Compute Route Matrix Pro | Pro | 10 per element | quote-screen ETA, dispatch |
| Dynamic Maps (JS) | Essentials | 7 | admin/web |

**Target architecture, per ride (after P1 to P8):**

| Operation | Calls per ride | Cost |
|---|---|---|
| Autocomplete sessions ending in Details (destination, sometimes pickup) | 1.3 Details | 0.0065 |
| Abandoned autocomplete sessions | 0.4 × 5 requests | 0.0057 |
| Reverse geocoding (initial pin + one adjust, 30 % cache hits) | ≈1.0 | 0.005 |
| Quote route (traffic-aware) | 1 | 0.010 |
| Quote-screen driver ETA matrix | 3 elements | 0.030 |
| Dispatch matrix | 3 elements | 0.030 |
| Nav legs (2) + 0.5 reroute | 2.5 | 0.025 |
| Trigger-based ETA refreshes | 3 to 5 | 0.030 to 0.050 |
| **Total** | | **≈ 0.14 to 0.16** |

Optional reductions: `TRAFFIC_UNAWARE` for the quote route (−0.005) and Essentials matrix for the quote screen (−0.015) bring it to ≈ 0.12.

**Monthly estimate (30 days, list price, before negotiated discounts, free tier applied once per SKU):**

| Scale | Rides / month | Target architecture | Current architecture pointed at Google without P1/P3/P5 fixes* |
|---|---|---|---|
| 100 rides/day | 3,000 | ≈ 450 list → **≈ 0 to 150** after free tier | ≈ 1,800 |
| 1,000 rides/day | 30,000 | **≈ 4,000 to 4,500** | ≈ 18,000 |
| 10,000 rides/day | 300,000 | **≈ 40,000 to 45,000** | ≈ 180,000 |
| 50,000 rides/day | 1,500,000 | **≈ 200,000 to 225,000** at list; volume pricing above 100k/SKU makes negotiation mandatory | ≈ 900,000 |

*The right-hand column assumes the audited behaviours stay: ≈36 reverse geocodes per ride from the 5 s search-screen loop (C-01), ≈20 route polls per five-minute wait (C-02), triple nav fetches (C-05), and per-class quote routes (C-12): ≈ USD 0.60 per ride. **These four items are the financially dangerous architecture** and must be fixed before Google becomes primary, independent of everything else in this plan.

Other cost levers: session tokens on the admin SPA (C-09), negative caches (C-08), input caps on the proxy (C-07), and Redis-backed limiters (P-10).

### 19.17 Caching strategy (within Google Maps Platform terms)

| Data | Store | TTL | Basis |
|---|---|---|---|
| Place IDs | Mongo (rides, favorites, recents, places) | indefinite | permitted explicitly |
| Place Details content (formatted address, components, viewport) | Redis + Mongo `places.content` | ≤30 days, then refresh on next use | 30-day content allowance |
| Autocomplete predictions (shared) | Redis | 24 h populated, 5 min empty | within allowance; attribution required |
| Geocoding results | Redis, 4-decimal key | 24 h; negatives 5 min | within allowance |
| Routes: free-flow quote | Redis | 5 min | transient, performance only |
| Routes: traffic-aware nav | Redis keyed on `(ride, phase, version)` | 90 s | transient |
| Route Matrix elements | Redis per pair | 2 to 3 min | transient |
| Driver ETAs | Redis | 3 min | transient |
| Ride's own route polyline and distance (business record of the transaction) | Mongo `Ride.activeRoute` (encoded) | ride lifetime + retention policy | part of the user's transaction record; do not aggregate into a derived dataset without legal review |
| User-chosen coordinates | Mongo | indefinite | user data, not Google content |
| Client: exact-hit autocomplete LRU, reverse-geocode LRU, decoded route | memory | session | display cache |

Do not: store `addressComponents` long-term, cache Google predictions client-side beyond the session, or pre-fetch routes for rides that do not exist.

### 19.18 Network failure behaviour

| Condition | Behaviour |
|---|---|
| Google timeout (>4 s routes, >2.5 s autocomplete/geocode) | Autocomplete: show cached/recent/saved entries with a "search unavailable" hint. Geocode: label "Selected location", allow confirm with coordinates. Quote: server prices from the last cached route for the same 4-decimal OD pair (≤30 min old) or, failing that, `haversine × 1.4` with `quote.source:'fallback'` and a visible "estimated" badge; never from client distance. Nav: driver keeps the current route; retry chip. |
| HTTP 429 from Google | Provider raises `RateLimited`; circuit breaker opens for 30 s per SKU; callers use the fallbacks above; alert. |
| HTTP 5xx | Retry once after 500 ms with jitter, then fallback. |
| No internet on device | Passenger: cached suggestions only, pin confirm allowed, booking queued until online (existing `NetInfo` gate). Driver: breadcrumbs buffered oldest-first, replayed with device `ts`; nav continues on the last route. |
| GPS unavailable | Passenger: map at last known position or Kutaisi centre; typed or pinned pickup allowed. Driver: GPS-lost banner after 15 s while moving (existing); server marks stale after 30 s and excludes from dispatch. |
| Socket reconnect | Client re-emits `nav:session_start` keyed on socket id; server replays current leg route and latest ETA; passenger reconciles ride state (existing). |
| App background / foreground | Passenger: pause GPS watch and driver-ETA rendering; on foreground re-sync ride state, no location reset (B-19). Driver: single background task continues; on foreground re-subscribe UI only. |

Nothing in the map stack throws to the UI; every provider call returns a typed failure the screen can render.

### 19.19 State model

Passenger (one reducer/store, replacing ~45 `useState` + ~35 refs + 8 module singletons in `TaxiScreen`):

```
gps:      { fix: {lat,lng,accuracy,heading,ts} | null, permission, watching: bool }
pickup:   LocationRef | null          // source 'gps' | 'pin' | 'places' | 'favorite'
dropoff:  LocationRef | null
stops:    LocationRef[]
quote:    { id, classes: [{type, price, etaSeconds}], route: {encoded, distanceMeters, durationSeconds}, surge, source, at }
ride:     { id, status, driver, activeRoute } | null       // server-authoritative snapshot
driver:   { pos: {lat,lng,heading,speed,ts}, snapped, etaSeconds, remainingMeters, stale: bool }
camera:   { mode, userControlled: bool }
ui:       { bookingStep, sheetState, searching, error }
```

Driver: `LocationStore` (fix stream), `RideStore` (single source for active ride, fed by `ride:updated` payloads), `NavStore` (route state, version, step index, off-route, eta), `CameraStore` (mode). Screens select slices; no per-screen copies of the ride.

---

# Part E. Migration plan

## 20. Migration phases

Each phase is independently shippable behind the existing feature-flag mechanism (`server/utils/featureFlags.js`, `MAPS_*` env switches) and ends with measurable exit criteria. Google becomes *primary* for a capability only when its phase passes; until P9 the old provider stays as the fallback in `tryChain`.

### Phase 1 — Audit, cleanup, and hard bug fixes (≈2 to 3 weeks)
Goal: stop the bleeding and create the surfaces the later phases plug into.
- Fix B-01 (validator `.toFloat()`, reject when the server quote is unavailable), B-02 (remove the parallel broadcast), B-05 (invalidate the driver cache on status changes), B-09, B-13, B-18, B-19, B-20, B-21, B-23 (TTL, dedupe, hard-delete), B-26, B-27, B-30, B-32, B-34, B-35, B-43, B-44, S-05, S-06 (validation half), S-10, S-11.
- Server ordering guard for driver locations (reject `ts` older than stored; accept device `ts`/`accuracy` fields if present) — first half of B-25/R-05.
- Proxy input caps and Redis-backed limiter (C-07, P-10), route all Google calls through `google.provider` with metrics (C-14, S-02).
- Delete: `roadSnap.service`, Roads proxy endpoint and driver `roadSnapping.js` (C-10); `matching.service`; `location.model`; `processPositionUpdate` and dead flags (D-02); dead client modules (D-03, D-04, D-05); browser Nominatim and Expo Go OSM tiles (S-09).
- Create `packages/maps` from the passenger copy of the wrappers (it has the NaN guards), add the manager's `memo` comparator and `opacity`, and point all three apps at it while still on Mapbox (D-01). This is the seam P6 swaps behind.
- Contract tests for `routing.service`, `geocoding.service`, `autocomplete.service`, `cache.service` keys, `pricing.computeServerQuote` (numbers and strings), dispatch ranking, and the location ordering guard (D-07).
- Docs: check in or delete the optimisation-plan references; update the navigation handoff non-goals (D-06).
**Exit criteria:** no client-set price can persist (test); one `ride:request` delivery path; driver marker never moves backwards in a 30-minute drive log; all three apps build from the shared map package; cost dashboard shows Distance Matrix and Roads counters.

### Phase 2 — Google Places (New) (≈2 weeks)
- `google.provider`: `autocompleteNew`, `placeDetailsNew` with field masks and session tokens.
- `autocomplete.service`: Google-only prediction path behind `MAPS_PLACES_NEW=1`; Mongo becomes a place-id/coordinate cache; default `locationBias` per city (B-39); dedupe removed (B-40).
- `LocationRef` schema and `placeId` plumbed through `validators.js`, `ride.model.js`, `favoriteLocation.model.js`, `recentLocations.service.js`, `favorites.controller.js`, and the passenger's selection handlers (Section 19.2); 30-day content refresh on `places` (S-07); only Google ids suggestable (S-06).
- Passenger: `AbortController` and memoised bias (R-04), drop prefix filtering (C-13), forward `language`, provider-driven attribution (S-08 interim).
- Admin SPA: `AdminCreateRide` through `/maps/autocomplete` + `/maps/place-details` (C-09).
- Georgian coverage benchmark (Section 19.3) run and signed off.
**Exit criteria:** ≥95 % of the benchmark queries return the expected place within 100 m; p95 autocomplete latency < 400 ms from Kutaisi; per-selection cost matches the session-priced model in the dashboard.

### Phase 3 — Geocoding (≈1 week)
- `google.provider.reverseGeocode` with `result_type`, single language, provider-neutral response (B-15, B-16, B-41); negative cache (C-08); bbox validation.
- Passenger `useReverseGeocode` hook with ordering and cancellation used by all four pickup paths (R-01, R-02); geocode only on settle and movement gates (C-01, C-04); rounded coordinates.
- `/locations/*` either removed or brought under the same limiter and validation (S-05).
**Exit criteria:** ≤2 reverse geocodes per ride in a 50-ride field test; zero mismatched pickup labels in the adjust-pin race test (A→B→C with delayed A).
**Gate:** Phases 2 and 3 may run Google as *fallback* in production immediately, but flipping Google to *primary* for user-visible content waits for Phase 6 (S-08).

### Phase 4 — Google Routes (≈2 to 3 weeks)
- `google.provider.computeRoutes` and step/congestion mapping (Section 19.5); `routing.getRoute`/`getNavRoute` use Routes behind `MAPS_ROUTES_API=1` with OSRM/Mapbox as fallback until P9.
- Waypoints everywhere (B-03); `pricing.quote` used by the quote endpoint (B-08); server-owned final fare (B-04); scheduled re-quote (B-14); encoded polylines on `Ride.activeRoute` (P-07); traffic preferences per workflow (C-06).
- Passenger: route + quote collapsed into one server call with request ordering (R-03, P-09); `legs[]` for stop ETAs (C-03); remove direct OSRM (S-09).
- Driver: server becomes the route owner per leg; client engine consumes `nav:route_computed`; `directions.js` reduced to RideDetail preview reading `activeRoute` (B-31 first half, C-05); `SpeedLimitSign` removed or gated.
- Admin/web: route previews via the proxy (B-38).
**Exit criteria:** every priced ride has `quote.source:'routing'` with stops included; one `computeRoutes` per leg in logs; passenger and driver draw the same geometry.

### Phase 5 — ETA and dispatch (≈2 weeks)
- `google.provider.computeRouteMatrix`; `etaDispatch` switched with Redis-backed coalescing (Section 19.7); hierarchy-aware quote-screen matrix (B-11); Mongo fallback only when Redis is down (B-07); second wave before broadcast.
- `eta.service` with trigger-based refresh and `driver:locationUpdate` enrichment (Section 19.6); passenger deletes haversine ETA and the 15 s poll (C-02, B-22); driver HomeScreen ETA from the engine (B-29 partial); Live Activity from server values (B-09 follow-up); proximity gates server-owned from Redis history (B-12).
**Exit criteria:** ≤6 ETA route calls per ride at p95; passenger ETA error < 2 minutes at p90 against actual arrival in a 100-ride sample; no client-side speed constants remain (grep).

### Phase 6 — Google Maps SDK rendering (≈3 to 4 weeks, the largest phase)
- `packages/maps` re-implemented on `react-native-maps` `PROVIDER_GOOGLE`; SDK keys per app (S-03); Cloud map styles for light/dark and house-number emphasis; `Polygon` surge heat; `strokeColors` traffic polyline; native `onRegionChangeComplete` (R-08); `tracksViewChanges` discipline (P-01, P-02); context split (P-05); camera state machine (Section 19.12); ops snapshot fields (B-37); admin loader consolidation (D-05).
- Remove raw `Mapbox.*` usages from screens (`TaxiScreen.js:2997-3088`, `RideDetailScreen.js:247-278`).
- Device matrix test: low-end Android (2 GB), mid iPhone; dark mode; theme switch; 2,000-point polyline; 50 ambient drivers.
**Exit criteria:** frame time p95 < 16 ms during a simulated ride on the low-end device; no marker flicker on label change; Google content displayed only on Google tiles → Phases 2/3 flip to primary.

### Phase 7 — Real-time navigation (≈2 to 3 weeks)
- Driver single GPS pipeline with device `ts`/`accuracy`/`speed`/`seq` (B-25, B-33, B-36, P-06); profile table (Section 19.14); single navigation engine shared by Home and Navigation (B-29); off-route rules with accuracy, heading, and back-off (B-31 second half, Section 19.11); reconnect re-emit (B-31); watchdog gaps (B-10).
- Passenger: route-snapped, queued, natively driven car animation (Section 19.9, B-28, P-03); device-`ts` stale guard (R-07); polyline trim by index (P-04).
**Exit criteria:** car marker never reverses or teleports in a 30-minute recorded drive; reroutes ≤ 2 per 10 km on a scripted detour test; battery drain ≤ 8 %/hour online idle on the reference Android device.

### Phase 8 — Performance and cost optimisation (≈1 to 2 weeks, overlaps P6/P7)
- Redis throttles and limiters everywhere (P-10); `MGET` and coarser ETA keys (C-11, P-12); `normalizedAddress` index or removal of Mongo text search (P-11); one quote endpoint for all classes (C-12); per-SKU quotas and alerts (S-01); cost dashboard aligned to Google SKUs (C-14).
**Exit criteria:** dashboard cost per ride within 20 % of the Section 19.16 model for two consecutive weeks.

### Phase 9 — Remove legacy providers (≈1 week, after two stable weeks of P6/P7)
- Delete `osrm.provider`, `mapbox.provider`, `nominatim.provider`, legacy Google endpoints in `google.provider`, `nominatim/` directory, `scripts/generate-houselabel-bg.js`, `@rnmapbox/maps` and its plugin/config, Mapbox tokens (revoke), `MAPBOX_TOKEN`, `OSRM_URL`, `NOMINATIM_*` env, `tryChain` fallbacks, `osm:`/`manual:` id handling, "Powered by OpenStreetMap" (S-12), `@turf/*` and `zustand` if still unused, privacy text update.
**Exit criteria:** grep for `mapbox|osrm|nominatim|openstreetmap|roads.googleapis|/maps/api/` returns only history/docs.

## 21. Exact files and components to modify

Grouped by phase; representative lines are cited in the findings.

**P1**
- `server/middlewares/validators.js` (toFloat, stops rules); `server/services/pricing.service.js` (coerce, reject path, remove duplicates); `server/controllers/ride.controller.js` (quote persistence, publish flags, stops validation, inline haversines, Live Activity progress); `server/workers/rideEventWorker.js`; `server/controllers/driver.controller.js` (accept `ts/accuracy/speed`, ordering guard, `invalidateDriver` calls); `server/middlewares/auth.middleware.js`; `server/middlewares/rateLimiter.js` (driver key generator); `server/routers/maps.router.js` and `locations.router.js` (Redis limiter, caps, validation); `server/controllers/maps.controller.js`; `server/socket/handlers/navigation.js` (coord schema); `server/services/etaDispatch.service.js` (use provider + metrics); `server/services/metrics.service.js`; `server/services/recentLocations.service.js`; `server/jobs/hardDelete.js`; delete `server/services/roadSnap.service.js`, `matching.service.js`, `models/location.model.js`, `navigation.service.processPositionUpdate`, `utils/featureFlags.js` dead flags; `server/__tests__/integration/maps.controller.test.js` (mock names) + new unit tests.
- `mobile/src/screens/TaxiScreen.js` (TDZ, foreground refresh, booking gate, SOS prop, `_ts`); `mobile/src/services/googleMaps.js` (remove OSRM direct); dead modules in `mobile/src/components/map/`.
- `mobile-driver/src/context/LocationContext.js` (watchdog, profile driven by ride store, iOS pause, permission prompt); `mobile-driver/src/context/DriverContext.js` (`ride:updated` merge); `mobile-driver/App.js` (mount recovery); `mobile-driver/src/hooks/usePermissionMonitor.js`; `mobile-driver/src/services/backgroundLocation.js` (queue store, ordering, device ts); `mobile-driver/src/services/RideTrackingService.js` (remove `persistRideConfig`, profile dup); `mobile-driver/src/services/rideStorage.js`; `mobile-driver/src/context/SocketContext.js`; `mobile-driver/src/services/directions.js` (token invalidation); delete `roadSnapping.js`.
- `mobile-manager/src/components/map/MapViewWrapper.js` (remove OSM tiles); `client/src/pages/admin/AdminCreateRide.jsx` (remove browser Nominatim).
- New: `packages/maps/*` (from `mobile/src/components/map/*` + manager additions); all three `app.config.js`/`babel`/`metro` to resolve it.

**P2**
- `server/providers/google.provider.js` (Places New); `server/services/autocomplete.service.js`; `server/services/places.service.js`; `server/models/place.model.js` (TTL/refresh); `server/models/ride.model.js`, `favoriteLocation.model.js` (LocationRef); `server/controllers/favorites.controller.js`; `server/controllers/ride.controller.js` (placeId intake, `upsertRidePlace` rules); `server/utils/city.js` (bias config).
- `mobile/src/components/taxi/LocationSearchSheet.js`; `mobile/src/services/googleMaps.js` (LRU, language, attribution flag); `mobile/src/screens/TaxiScreen.js` (selection handlers carry `placeId`); `mobile/src/screens/SavedPlaceEditorScreen.js`, `FavoriteLocationsScreen.js`.
- `client/src/pages/admin/AdminCreateRide.jsx`, `client/src/hooks/usePlacesAutocomplete.js`.

**P3**
- `server/providers/google.provider.js` (geocode params), `server/services/geocoding.service.js` (shape, negatives, bbox), `server/routers/locations.router.js`.
- New `mobile/src/hooks/useReverseGeocode.js`; `mobile/src/screens/TaxiScreen.js` (four pickup paths, watch-loop gating); `mobile/src/services/googleMaps.js` (`toReverseResult` removal, rounding, cache).

**P4**
- `server/providers/google.provider.js` (`computeRoutes`, step/congestion mapping); `server/services/routing.service.js`; `server/services/cache.service.js` (nav key by ride/phase/version, encoded payloads); `server/services/pricing.service.js`; `server/controllers/ride.controller.js` (quote endpoint, `createRide`, `startRide`, `completeRide`, scheduled re-quote); `server/models/ride.model.js` (`activeRoute` encoded); `server/services/navigation.service.js` (route owner, push on start/phase/reroute).
- `mobile/src/screens/TaxiScreen.js` (`fetchDirectionsAndUpdate` → quote call with ordering, remove per-stop calls, decode encoded polyline); `mobile/src/hooks/useRideQuote.js` (extend); `mobile/src/components/taxi/RideOptionsSheet.js` (server prices).
- `mobile-driver/src/navigation/useNavigationEngine.js` (consume server routes only); `mobile-driver/src/services/directions.js`; `mobile-driver/src/screens/NavigationScreen.js` (`SpeedLimitSign`); `mobile-driver/src/screens/RideDetailScreen.js` (use `activeRoute`).
- `client/src/components/RouteMap.jsx`, `client/src/pages/admin/AdminRides.jsx`, `client/src/pages/SharedRide.jsx`, `web/src/components/ride/SharedRideView.jsx`.

**P5**
- `server/services/etaDispatch.service.js` (`computeRouteMatrix`, Redis coalescing, second wave); `server/services/driverDispatch.service.js`; `server/controllers/ride.controller.js` (quote-screen matrix, proximity gates); new `server/services/eta.service.js`; `server/controllers/driver.controller.js` (enriched emit, gate); `server/services/rideActivity.js`.
- `mobile/src/screens/TaxiScreen.js` (delete haversine ETA, 15 s poll, `osrmEtaPriorityRef`); `mobile/src/components/taxi/RideStatusSheet.js`; `mobile-driver/src/screens/HomeScreen.js` (ETA from engine).

**P6**
- `packages/maps/*` (all wrappers on `react-native-maps`), `mobile/app.config.js`, `mobile-driver/app.config.js`, `mobile-manager/app.config.js` (SDK keys, plugin), `eas.json` secrets; `mobile/src/screens/TaxiScreen.js` (camera FSM, marker tree, heading throttle, raw Mapbox removal), `mobile/src/screens/RideDetailScreen.js`, `mobile/src/components/map/mapStyle.js` → cloud style ids; `mobile-driver/src/screens/NavigationScreen.js`, `HomeScreen.js`, `RideDetailScreen.js`; `mobile-driver/src/context/LocationContext.js` (context split); `mobile-manager/src/components/ops/OpsMap.js`, `screens/fleet/*`, `theme/palettes.js`; `server/controllers/ops.controller.js` (snapshot fields); `client/src/lib/googleMaps.js` and the seven loaders; `web/src/components/ride/SharedRideView.jsx`.

**P7**
- `mobile-driver/src/services/backgroundLocation.js`, `RideTrackingService.js`, `LocationBuffer.js`, `LocationThrottle.js`, `LocationHeartbeat.js`, `context/LocationContext.js` (single pipeline, profile module); `mobile-driver/src/navigation/useNavigationEngine.js` (off-route rules, reconnect), new `NavProvider`; `mobile-driver/src/screens/HomeScreen.js` (remove second engine); `packages/maps/AnimatedMarker` (queue, native driver, route snap); `mobile/src/screens/TaxiScreen.js` (ingest on device ts, snap, trim); `server/services/rideWatchdog.service.js`; `server/socket/handlers/navigation.js` (reconnect replay).

**P8**
- `server/routers/maps.router.js`, `server/socket/opsThrottle.js`, `server/services/rideActivity.js`, `server/services/etaDispatch.service.js`, `server/models/place.model.js`, `server/services/places.service.js`, `server/services/metrics.service.js`, `client/src/pages/admin/AdminCostMetrics.jsx`, Cloud Console quotas.

**P9**
- Deletions listed in Section 20; `server/utils/validateEnv.js`, `server/.env.example`, three `.env.example`s, `index.js` of each app, `package.json`s, `client/src/pages/Privacy.jsx`, `web/src/components/pages/PrivacyPage.jsx`, `mobile/src/components/taxi/LocationSearchSheet.js` attribution.

## 22. Estimated complexity per change

| Change | Complexity | Notes |
|---|---|---|
| Validator/pricing bypass (B-01) | Low | 10 lines + tests |
| Remove parallel broadcast (B-02) | Low | flags only |
| Driver cache invalidation (B-05) | Low | 4 call sites |
| Server location ordering guard, device ts (B-25/R-05 server half) | Low–Medium | controller + hash compare |
| Proxy caps, Redis limiter, validation (C-07, S-05) | Low | existing limiter store |
| Shared `packages/maps` on Mapbox (D-01) | Medium | mechanical consolidation, three apps' imports |
| Contract tests (D-07) | Medium | ~15 test files |
| Places (New) provider + service (P2) | Medium | shapes, field masks, session semantics |
| `LocationRef` + placeId plumbing (19.2) | Medium–High | 4 models, validators, 3 client selection paths, migration script for legacy rows |
| Geocoding params + neutral shape (P3 server) | Low | |
| `useReverseGeocode` and pickup unification (P3 client) | Medium | four call paths in a 3,725-line file |
| `computeRoutes` provider + step/congestion mapping (P4) | Medium–High | maneuver mapping, `speedReadingIntervals` expansion, encoded polyline everywhere |
| Waypoints + server final fare + scheduled re-quote (P4) | Medium | controller changes, receipt update |
| Route+quote single call with ordering (P4 client) | Medium | replaces `fetchDirectionsAndUpdate` |
| Server route owner for nav (P4 driver) | Medium | engine already version-guards |
| Route Matrix dispatch + quote-screen matrix (P5) | Low–Medium | service already structured |
| `eta.service` trigger refresh + enriched emit (P5) | Medium | new service, projection on server |
| Passenger ETA cleanup (P5) | Low | deletions |
| Google Maps SDK rendering package (P6) | **High** | third renderer swap; markers, traffic polyline, heat, camera; 3 apps; device QA |
| Camera state machine (P6) | Medium | replaces 8 refs |
| Context split / render discipline (P6) | Medium | |
| Single driver GPS pipeline (P7) | Medium–High | background task as sole source, emitter, profile table |
| Single navigation engine (P7) | Medium | provider + HomeScreen simplification |
| Off-route rules + back-off (P7) | Low | constants and one predicate |
| Queued native car animation with route snap (P7) | Medium | new `AnimatedMarker` |
| Cost dashboard, quotas, Redis throttles (P8) | Low–Medium | |
| Legacy removal (P9) | Low | deletions and token revocation |

## 23. Prioritised implementation order

1. **Week 1:** B-01, B-02, B-05, S-05/C-07 (caps + limiter), server location ordering guard, C-10 deletions, S-10. These stop money and dispatch leaks with tiny diffs.
2. **Weeks 2–3:** rest of P1: driver GPS watchdog/profile fixes (B-26, B-27, B-34), recovery hooks (B-30), passenger correctness (B-18, B-19, B-20, B-21), shared map package, contract tests, metrics routing.
3. **Weeks 4–5:** P2 Places (New) as fallback-first, Place ID model, Georgian benchmark, admin SPA proxying.
4. **Week 6:** P3 geocoding server + passenger hook; C-01 throttling.
5. **Weeks 7–9:** P4 Routes: provider, waypoints, authoritative quote and fare, server route owner, encoded polylines.
6. **Weeks 10–11:** P5 matrix dispatch and trigger-based ETA push; delete client polling (C-02).
7. **Weeks 12–15:** P6 Google Maps SDK rendering in all three apps behind the shared package; device QA; **flip Places/Geocoding/Routes to primary** at the end of this phase.
8. **Weeks 16–18:** P7 single GPS pipeline, single nav engine, off-route, smooth animation.
9. **Weeks 17–19 (overlapping):** P8 cost and performance hardening; two weeks of cost-dashboard observation.
10. **Week 20:** P9 remove Mapbox, OSRM, Nominatim, Roads, legacy endpoints; revoke tokens; update privacy text.

Dependencies: P6 gates the production flip of P2/P3/P4 content (S-08). C-01, C-02, C-05, and C-12 must be fixed before their respective Google APIs become primary. P7's single pipeline depends on P1's server ordering guard. P9 waits for two stable weeks after P6 and P7.

---

## Appendix A. Provider calls per ride, today vs target

| Step | Today | Target |
|---|---|---|
| Quote screen | 3 OSRM (per class) + 1 OSRM trip | 1 `computeRoutes` + 1 matrix (≤3 elements) |
| Booking | 0–1 OSRM | 0 (cache) |
| Pickup snap | 1 Google Roads (unused) | 0 |
| Dispatch | 1 Google DM (3 elements) | 1 `computeRouteMatrix` (3 elements) |
| Waiting | ≈20 OSRM polls (15 s) from the passenger | 0 client calls; 2–3 server ETA refreshes |
| Nav legs | 3–6 Mapbox (session + HTTP duplicate + phase + start + reroutes) | 2 `computeRoutes` + ≈0.5 reroute |
| In ride | 0 (ETA frozen) | 2–3 server ETA refreshes |
| Search screen idle | ≈36 Nominatim reverse geocodes (5 s loop) | ≤2 Geocoding |
| Autocomplete | Nominatim per keystroke + Google when sparse; legacy Details | session-priced Places (New) |

## Appendix B. Verification checklist for the whole programme

- Pricing: fuzz `POST /rides` with numeric, string, and missing coordinates; the stored `quote.totalPrice` must equal the server computation in every case.
- Dispatch: log every `ride:request` delivery; exactly one recipient at a time until the fallback broadcast.
- Location ordering: replay a recorded drive with a 30 s delayed batch; the Redis `ts` must be monotonic and the passenger marker must never reverse.
- Pickup race: automate map moves A→B→C with response delays 900/300/100 ms; the confirmed address must be C's.
- Autocomplete race and cost: type 12 characters at 80 ms intervals; at most 3 network requests and one Details call; cost dashboard increments match.
- Routes parity: for 200 historical rides compare OSRM/Mapbox vs Routes distance (±3 %) and polyline Hausdorff distance (<30 m) before switching pricing.
- ETA accuracy: 100-ride sample of predicted vs actual arrival, p90 < 2 min.
- Rendering: Perfetto/Instruments traces on the low-end device during a simulated 15-minute ride; p95 frame < 16 ms; no `tracksViewChanges` churn.
- Keys: attempt each web-service API with each SDK key and the browser key; all must be refused.
- Terms: no Google content on non-Google tiles after the P6 flip; "Powered by Google" visible in list-only contexts; `places` content older than 30 days is refreshed or blanked.
