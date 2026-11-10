# One-shot prompt for Claude Fable

Paste everything below the line into Claude Fable as the only instruction. Do not attach secrets. The map key, Google key, Mapbox token, Yelp key, and xAI key are environment variables, never source.

---

You are implementing **Halt**, a working drive-stop planner. Ship the whole product in one pass. Do not leave a form-only mock, a fake map, invented fuel prices, or invented ratings.

## What it is

Halt plans a drive from a start to an end, finds fuel stops that match the driver's rules, and shows those stops on a map so the driver picks them. Changing preferences must change the candidate set and the arrival time. Do not freeze one route and only highlight a different exit.

The front door is a **Grok bot**. The driver talks. The bot calls the planner. The map is the result, not the form. A vehicle and filter sheet stays one tap away for people who want to set the car without typing it.

Build a web app. Keep the planner, stop rules, and provider calls in modules that do not import React, so a later Kotlin client can call the same rules. Do not build Android or a native shell in this pass.

## Visual

Minimal and military. Dark ink background, hairline borders, no cards with drop shadows, no rounded pastel bubbles, no emoji.

- Display: Barlow Condensed. Body: Barlow. Labels and readouts: IBM Plex Mono.
- Accent is warm amber. Route line is amber. The other toll route is a muted dashed line.
- Suggested stops, driver-chosen stops, and outliers use three distinct colors.
- Large tap targets (at least 44px). Dynamic Type-sized text. Every stop has an accessible name.
- Header wordmark: HALT. Title: GROK.

## Screens

1. **Talk.** A transcript, a text field, and Send. Do not call the model on page load. Opening copy is static: ask for a start, an end, and fuel percent. Chips may fill the box, not send. A **Start here** button stores a device fix. A **Vehicle** button opens the sheet.
2. **Map**, after a plan exists. Phone: Talk / Map toggle. Desktop: chat column about 22rem beside the map. The map draws the chosen route, the other toll route when it is actually different, and pins. Tapping a pin toggles it. The list shows the same stops.
3. **Vehicle sheet.** Year, make, model, trim from the EPA catalog. MPG and tank fill from that pick. Fuel percent is a 0–100 slider. Station type, amenities, stop-every, toll preference, max detour, min rating stay here too. Tell the driver to ask Grok to replan after a change. Do not auto-call the model when the sheet changes.

## Driver inputs

- **Start.** Typed place with city/address autocomplete, or the device location. If a device fix is set and the start means "here", "my location", "current", or "device", geocode is skipped.
- **End.** Same autocomplete. A picked suggestion is a pinned lat/lng, not a second geocode of the label.
- **Leave.** Default now. The sheet can also set a later date and time. Pass a real ISO time. Do not use a flat speed for duration.
- **Vehicle.** MPG, tank gallons, fuel percent. Fuel percent cannot exceed 100. Usable range is `mpg * tankGallons * (fuelPercent / 100)`. Full-tank range ignores the current percent. Stop frequency does not change usable range.
- **Station type**, multi-select: `any`, `truck_stop`, `travel_center` (Love's, Pilot, Flying J, TA), `convenience` (Sheetz, Wawa, QuikTrip, Buc-ee's), `diesel`, `budget`, `highly_rated`.
- **Amenities**, multi-select, every marked one is required: food, restrooms, showers, truck parking, EV charging, Wi-Fi, open late, dog area. Unknown is not a match. Station type is a separate match.
- **Stop every** 1, 2, 3, or 4 hours of driving. Default 2. Spacing in miles is `min(hours * corridorMph, usableRangeMiles)`. Never space stops by a fixed percent of range.
- **Tolls.** Plan both. `with` and `without` are separate geometries. Open the one the driver asked for. If distance differs by under 1.5 miles and time by under 180 seconds, mark `sameRoads` and do not draw a second line.
- **Max detour** default 5 miles, allowed 0.5–25. **Min rating** default 4.

## Saved vehicle

`localStorage` key `halt.vehicle.v1`. Save year, make, model, option id, trim, label, mpg, tank gallons, and the mpg note. Do **not** save fuel percent. Auto-save once year, make, and model are set. Next visit shows a card, not empty dropdowns, with Change and Forget. Hydrate after mount so server render does not disagree with the browser. Ignore JSON whose mpg is outside 1–200 or tank outside 1–150.

## Planner rules

Pure functions. No keys, no UI.

Project each place onto the route. Keep miles-off. A stop fails when:

- It misses the requested station type. `any` passes. `budget` and `highly_rated` are extra gates, not brands. Other selected types pass if the stop has any one of them.
- A required amenity is not explicit yes.
- It has a rating and that rating is under the minimum. **Unrated stops stay on the map and can be tapped. They are not auto-suggested.** Note them as unrated.
- `highly_rated` needs 30 combined reviews, or 8 if fewer than 3 other stations sit within 40 miles (rural).
- `budget` needs a real price in the cheapest 40 percent of priced stations within 50 miles. No price means it cannot match budget. Do not invent a price.
- Two or more reviews match the same rejection: closed / out of business, no or locked restroom, unsafe or robbed, out of fuel or pumps down. One review is not enough to overrule the star average.
- Miles off the route exceed the detour cap.

Flag Google vs Yelp when both exist and they differ by more than 0.8. Show both scores. Match a Yelp business to a Google place by name and distance under 150 meters.

Suggestions walk the route. From the current mile, look ahead one spacing interval, pick the best passing **rated** stop near the end of that window (rating, trust, less detour), then continue from there on a full tank. Cap around 16 suggestions. Do not suggest a stop that failed. Outliers: review rejection, dirty flood, a filter match whose only failure is the detour, and a price more than 15 percent off the local median. A trusted stop rated about 0.6 above nearby stops is an info outlier, not a failure.

Dwell time belongs in the arrival. Arrival uses router duration plus dwell, not a guessed speed.

## Data sources, in fallback order

Never embed keys in the client. If a key is missing, use the next source and say so in `provider.notes`. If every source fails, say the place or route could not be found. Do not guess a coordinate.

**Geocode**

1. Google Geocoding if `GOOGLE_MAPS_API_KEY`
2. Mapbox if `MAPBOX_ACCESS_TOKEN`
3. Nominatim `https://nominatim.openstreetmap.org/search` with a real User-Agent
4. Photon
5. ArcGIS World Geocoding `findAddressCandidates`

Autocomplete for start and end uses the same stack and returns cities and addresses, not only businesses.

**Route.** Duration for the chosen leave time. No flat mph for the ETA.

1. Google Routes, `routeModifiers.avoidTolls`, traffic for that departure
2. Mapbox Directions, `exclude=toll` when avoiding tolls
3. Valhalla `POST https://valhalla1.openstreetmap.de/route` with `costing: auto`, `costing_options.auto.use_tolls` 0 or 1, `shape_format: polyline6`. This is the open source that actually splits toll and non-toll. Decode precision 6.
4. OSRM public server does **not** honor `exclude=toll`. Do not pretend it did.

Run with-tolls and without-tolls together. Corridor speed is route miles divided by the traffic hours, used only for stop spacing.

**Places**

1. Google Places nearby, ratings, review counts, hours, review text, when the Google key exists
2. Otherwise ArcGIS `findAddressCandidates` with `category=Gas Station`, location bias along about 6 samples of the corridor, plus brand queries Sheetz, Pilot, Love's, Flying J, truck stop. Dedupe. This is the source that actually returns stops without a key.
3. Overpass `amenity=fuel` only as a fallback. Public Overpass mirrors time out. Do not block the plan on them.
4. Photon last

**Reviews.** Yelp Fusion when `YELP_API_KEY` exists: second rating, review count, categories, snippets. No Yelp key means reviews are none, and the UI says so.

**Prices.** A fuel-price feed only when a key exists. Otherwise omit the price. Never fabricate one.

**Basemap.** MapLibre GL raster tiles. The browser requests `/api/halt/tiles?z=&x=&y=`. The server appends `CARTO_API_KEY` and fetches CARTO. The key never appears in client JS, HTML, or the tile URL the browser sees. Do not hardcode a key. If the key is missing, the map must still show the route geometry rather than a blank pane. MapLibre's own CSS sets `.maplibregl-map { position: relative }`, which collapses a Tailwind `absolute inset-0` pane to 0 height. Override with `.halt-map.maplibregl-map { position: absolute; inset: 0; width: 100%; height: 100%; }`, call `resize()` on load, and observe the container. Clone the style object before handing it to MapLibre. Set the worker URL explicitly.

**Vehicles.** EPA menu, no key: `https://www.fueleconomy.gov/ws/rest/vehicle` with `Accept: application/json`. Years, then makes, then models, then options. Highway MPG fills the MPG field. Electric-only records do not overwrite MPG. Cache menu responses.

**Tanks.** Not from EPA. Compressed rows from auto-data.net spec pages. Each row is `[year0, year1, gallonsInTenths, displacementInTenths, hybridFlag]`. Match make, then the longest model key that fits the EPA model name. Prefer the hybrid row only when the model name says hybrid, PHEV, Energi, or plug-in. Prefer displacement within 0.2 L. If several tanks cover that year, **use the smaller**. Say the catalog range in the note. A missing model leaves gallons as the driver typed them. Do not invent a tank.

Known checks, do not "correct" these away:

- 2021 Hyundai Elantra 1.6 turbo: 12.4 gal
- Elantra hybrid: 11.1 gal
- F-150 has several tanks. Use the smaller and note the range.
- Silverado and Ram model keys must match the catalog strings, not a shortened name.

## Grok bot

Server only. `POST https://api.x.ai/v1/chat/completions` with `XAI_API_KEY`, model `grok-4.5`, temperature 0.2, `max_tokens` 500. OpenAI-compatible tool calls. If the key is missing, say Grok is unavailable. Do not mock a plan.

The client sends the last 8 user/assistant turns (no tool blobs), the saved vehicle, the device fix, and `{id, name}` for current stops plus selected ids. Cap each turn around 2000 characters.

System rules: never invent a route, price, rating, hour, or stop. Ask only for start, end, and fuel percent when missing. Leave time defaults to now. Defaults: every 2 hours, 5 mile detour, rating 4, any fuel, tolls allowed. Call `plan_drive` once start and end exist. Use the saved MPG and tank unless the driver overrides them. Fuel percent 0–100. After a plan, mention distance, whether the toll-free line differs, and up to three stops from the tool list with their mile. Apple Maps takes every chosen stop. Waze takes one at a time. Call `select_stops` with ids. Do not claim a stop changed unless the tool says so. Brief. No emoji.

Tools:

- `plan_drive`: start, end, leaveAt (`now` or ISO), fuelPercent, mpg, tankGallons, stationTypes, amenities, frequency `1|2|3|4`, tolls `with|without`, maxDetourMiles, minRating. The tool runs the real planner. Return a short brief to the model: miles, minutes, usable range, both route summaries, at most 16 stops spread along the route `{id, name, mile, milesOff, tags, suggested}`, and warnings. The HTTP response to the browser also includes the full plan for the map.
- `select_stops`: `{add, drop}` of `{id, name}`. Resolve an id first, then one exact name, then one unique substring. Several matches is an error, not a guess.

At most two model rounds per user message. Tool failures go back as `{error}` so the model can explain them. No retries in a loop. No call on keystroke or page load.

## Handoff

Apple Maps unified directions, iOS 18.4+:

`https://maps.apple.com/directions?source=lat,lng&waypoint=lat,lng&waypoint=lat,lng&destination=lat,lng&mode=driving`

Repeat `waypoint` in drive order. Cap **13** waypoints plus the destination. If more were chosen, say how many were omitted.

Waze has no waypoint parameter. One target:

`https://waze.com/ul?ll=lat,lng&navigate=yes`

After that stop, the driver returns to Halt for the next one. Say that in the UI.

Choosing stops reroutes through the same router with those points, preserving avoid-tolls. If the reroute fails, keep the corridor line and say the arrival is estimated.

## API

- `POST /api/halt/plan` body is `PlanRequest`, returns `PlanOk` or `{ok:false, error}`
- `POST /api/halt/reroute`
- `POST /api/halt/chat` as specified above
- `GET /api/halt/vehicles` year / make / model / id
- `GET /api/halt/suggest` for place autocomplete
- `GET /api/halt/tiles` proxies the basemap

`PlanOk` has two `routes` (`with` and `without`), each with coordinates, candidates, rejected, warnings, suggestionIds, and corridorMph. Top-level fields mirror the opened route so older readers still work.

Validate: MPG 1–200, tank 1–150, fuel 0–100, detour 0.5–25, rating 0–5, leave time is a real date.

## Tests you must include and pass

- Usable range is `mpg * gallons * percent`, not the stop interval.
- Stop spacing is `min(hours * mph, fuel cap)`.
- Fuel percent above 100 is rejected.
- Saved vehicle parse drops bad mpg or tank and does not store fuel percent.
- A device fix is used only when the start means here.
- Stop select by id works. Two stops with the same name is an error.
- Apple URL repeats waypoint and reports omitted count past 13.
- Waze URL is a single `ll`.
- Tank lookup: Elantra 1.6 turbo 12.4, hybrid 11.1, F-150 uses the smaller tank.
- With-tolls and without-tolls on a corridor that has a toll road (Pittsburgh to Harrisburg is a known split) are not the same geometry.

## Do not

- Do not build a game, a score, or a second visual system.
- Do not highlight a different exit on one frozen polyline and call it a toll option.
- Do not hide unrated stations, and do not recommend them.
- Do not invent prices, ratings, hours, or closed/open.
- Do not put any API key in client code, the prompt output, or a public repo.
- Do not call Grok except when the driver sends a message.
- Do not require an account. Sign-in is out of scope.

## Sources

Use these endpoints. Do not swap in a lookalike.

| Job | Source | Endpoint |
|---|---|---|
| Vehicles | EPA fuel economy, no key | `https://www.fueleconomy.gov/ws/rest/vehicle` |
| Tank sizes | auto-data.net spec pages, compressed offline | row `[year0, year1, gallonsTenths, displTenths, hybridFlag]` |
| Geocode | Nominatim | `https://nominatim.openstreetmap.org/search` and `/reverse` |
| Geocode | Photon | `https://photon.komoot.io/api/` |
| Geocode and gas stations | ArcGIS World Geocoding | `https://geocode.arcgis.com/arcgis/rest/services/World/GeocodeServer/findAddressCandidates` |
| Toll split | Valhalla public | `POST https://valhalla1.openstreetmap.de/route` |
| Route fallback | OSRM, no real toll exclusion | `https://router.project-osrm.org/route/v1/driving/` |
| Basemap | CARTO Dark Matter raster | `https://basemaps.cartocdn.com/rastertiles/dark_all/{z}/{x}/{y}@2x.png` with the key added only on the server |
| Apple handoff | Unified Maps URLs | `https://maps.apple.com/directions` |
| Waze handoff | Waze deep link, one stop | `https://waze.com/ul` |
| Chat | xAI, OpenAI-compatible | `POST https://api.x.ai/v1/chat/completions` model `grok-4.5` |
| Keyed upgrades | Google, Mapbox, Yelp | only when `GOOGLE_MAPS_API_KEY`, `MAPBOX_ACCESS_TOKEN`, or `YELP_API_KEY` is set |

User-Agent on those public calls: `HaltRoutePlanner/1.0`.
