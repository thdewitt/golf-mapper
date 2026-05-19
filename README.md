# golf-mapper

A single-file web app for tracing golf course geometry on a satellite map and
emitting a JSON document the **scorekeeper app** can decode and use for
distance / target lookups during play.

- Live app: <https://thdewitt.github.io/golf-mapper/>
- Repo: <https://github.com/thdewitt/golf-mapper>
- Source: `index.html` (HTML/CSS/JS in one file, served by GitHub Pages from `main`)

## What it produces

A `*.coursemap.json` file conforming to the schema below. The scorekeeper app
loads these and uses the green / fairway / hazard geometry to compute things
like distance-to-green-center, carry-over-bunker, etc.

Tee box positions are **not** mapped — yardage in the scorekeeper is computed
from the player's current GPS, not from a per-tee point, so the per-tee
position carries no value. The mapper has tee tools but they're hidden by
default; flip the `.tool-btn[data-tool^="tee-"] { display: none }` rule in
`index.html` to bring them back.

## What's important on each hole

| Geometry            | Why it matters                                            |
|---------------------|-----------------------------------------------------------|
| Green perimeter     | Visual outline + source for center/front/back             |
| Green center / front / back | Distance-to-pin targets (front/back computed from approach axis) |
| Fairway perimeter   | Visual context; also gives autodetected "approach axis" for green front/back |
| Bunkers (`features`, type=`Bunker`, shape=`Polygon`) | Carry distance, layup risk |
| Water (`features`, type=`Water`, shape=`Polygon`/`Line`)   | Same                       |
| Wash  (`features`, type=`Wash`, shape=`Line`) | Penalty boundary                        |
| OB    (`features`, type=`OB`, shape=`Line`)   | Penalty boundary                        |

## CourseMap JSON schema

Top-level course object:

```jsonc
{
  "id": "UPPERCASE-UUID",                  // required, generated on New Course
  "clubName": "Canyata Golf Club",
  "courseName": "Canyata",
  "createdAt": "2026-05-18T00:00:00.000Z", // ISO-8601 UTC string
  "updatedAt": "2026-05-19T00:00:00.000Z",
  "teeColorsUsed": [],                     // currently always [], reserved
  "holes": [ /* 1..36 Hole objects */ ]
}
```

Hole object (every field is required, even if its array/object is empty):

```jsonc
{
  "id": "UPPERCASE-UUID",
  "number": 1,                  // 1-based hole number
  "par": 4,
  "strokeIndex": 9,             // handicap stroke index 1-18 (placeholder 1..N when unknown)
  "teeBoxes": [ /* TeeBox or [] */ ],
  "fairwayPerimeter": [ /* MapPoint[] or [] */ ],
  "green": { /* Green */ },
  "greenOverlay": {},           // reserved; always {} until used
  "features": [ /* Feature[] */ ],
  "waypoints": []               // reserved; always []
}
```

Green object:

```jsonc
{
  "perimeter": [ /* MapPoint[] */ ],
  "center": { /* MapPoint */ }, // optional pre-draw; mapper auto-fills on Draw Green
  "front":  { /* MapPoint */ }, // optional; mapper auto-fills if fairway exists
  "back":   { /* MapPoint */ }  // optional; same
}
```

TeeBox object (mapper currently leaves the array empty; included for parser compatibility):

```jsonc
{
  "id": "UPPERCASE-UUID",
  "color": "Rojo",              // free-form label
  "perimeter": [ /* MapPoint[] or [] */ ],
  "center": { /* MapPoint */ }
}
```

Feature object (bunkers, water, wash, OB, etc):

```jsonc
{
  "id": "UPPERCASE-UUID",
  "type": "Bunker",             // Bunker | Water | Wash | OB | (extensible)
  "name": "FB #1",              // auto-named by the mapper, user-editable
  "shape": "Polygon",           // Polygon | Line
  "points": [ /* MapPoint[] */ ]
}
```

MapPoint — **every** lat/lng anywhere in the document must have all six fields:

```jsonc
{
  "latitude": 39.469867856328875,
  "longitude": -87.70179279148579,
  "accuracy": 3,                // meters; satellite-trace nominal = 3
  "sampleCount": 1,
  "capturedAt": "2026-05-18T19:16:25.669Z",
  "source": "satellite"         // "satellite" for user-tapped, "derived" for computed
}
```

A partial `{latitude, longitude}` object will cause the scorekeeper's decoder
to fail with "could not parse that json as a coursemap". The mapper's Save
button now walks the course and normalizes every lat/lng-bearing object into
a full MapPoint before writing.

## Auto-detected green center / front / back

When the user finishes tracing a green polygon, the mapper computes:

- `center` = polygon centroid (always, immediately).
- `front` / `back` only if a fairway perimeter exists on the same hole. Approach
  axis = vector from fairway centroid to green centroid. Each perimeter vertex
  is projected onto that axis; the most-toward-fairway projection becomes
  `front`, the most-away becomes `back`. Distances are computed in local
  east/north meters (`localOffsetMeters`), not raw degrees, so the math is
  correct at any latitude.

The user can still manually drop Drop-Green-Front/Center/Back to override.

## Map / drawing UX notes (for cross-app debugging)

- Pencil + iPad: the map has a **Lock Map** button. While unlocked, taps on
  the map do NOT add vertices; only pan/zoom/rotate are active. Tap **Lock
  Map** before placing precise vertices; tap again to unlock and pan. This
  defeats Apple Pencil drift and palm/Pencil pinch-gesture artifacts that
  otherwise shift the map underneath the user.
- Polyline draws (Wash / OB) are time-gated against a synthetic-click bug in
  leaflet-draw 1.0.4; see the comments inline in `drawPolyline()`.
- Map rotation: N up / E up / S up / W up buttons (top-right) via the
  leaflet-rotate plugin. Bearings: N=0, E=90, S=180, W=270.

## Deeplinks

`?course=URL` on the page URL auto-fetches and loads the given course JSON at
startup. Example:

<https://thdewitt.github.io/golf-mapper/?course=courses/canyata.coursemap.json>

Bundled course templates live under `courses/`.

## Field origin reference for the scorekeeper app

| Field used by scorekeeper           | Set by mapper how                            |
|-------------------------------------|----------------------------------------------|
| `course.id` / `hole.id` / `*.id`   | `crypto.randomUUID().toUpperCase()` on creation |
| `hole.par`                          | Default 4 on New Course; should be set per scorecard manually |
| `hole.strokeIndex`                  | Placeholder 1..N; set per scorecard manually |
| `green.center/front/back`           | Auto-detected; manually overridable          |
| `features[].name`                   | Auto-generated (e.g. "FB #2 Right", "GB #1 Front") |
| `MapPoint.source`                   | "satellite" for user-tapped, "derived" for computed |
