# SmartFarm Crop IoT Dashboard

A full-stack dashboard for GreenFields Farm. Farm staff manage Crop Cards stored in SQLite, while the React frontend joins each card to the latest matching reading from a validated, read-only JSON sensor feed.

## Requirements and installation

- Node.js 20 or newer
- npm 10 or newer

From the project root:

```bash
npm install
npm run install:all
npm run dev
```

Open the frontend at **http://localhost:5173**. The Express API runs at **http://localhost:3001**. The Vite development proxy forwards `/api` requests to Express. To run separately, use `npm run dev` inside both `backend` and `frontend` in separate terminals.

Run all automated checks with `npm test`. Build the production frontend with `npm run build`.

## Database creation and seed

`backend/db.js` opens `smartfarm.db`, creates the authoritative `crops` table if necessary, and checks its row count. It inserts Tomato, Lettuce and Wheat only when the table is empty, so restarting does not duplicate records. Maize is intentionally not seeded. For a clean marker run, stop the backend, delete `backend/smartfarm.db` if it exists, then restart it.

SQLite owns only the user-managed Crop Cards. It does not store readings, calculated condition, alerts, recommendations, latest readings, refresh time or Overall Farm Status.

## API

| Method | Route | Purpose |
|---|---|---|
| GET | `/api/crops` | Return all Crop Cards |
| GET | `/api/crops/:id` | Return one Crop Card |
| POST | `/api/crops` | Validate and create a card |
| PUT | `/api/crops/:id` | Update allowed fields; crop name is immutable |
| DELETE | `/api/crops/:id` | Delete a Crop Card only |
| GET | `/api/readings` | Re-read, validate and return raw sensor JSON |

Every error is JSON in the form `{ "error": "Clear message" }`. Invalid input returns 400, a duplicate name returns 409, a missing card returns 404, and structural sensor failure or unexpected failure returns 500.

## Data ownership and processing

Crop Cards are editable SQLite records. `backend/data/sensor-readings.json` simulates an external feed and is read-only in the application; no write routes exist for it. Dashboard results exist only in React state.

`crop_name` is the only join key. Matching uses JavaScript strict equality, so `Tomato` matches `Tomato` but not `tomato`. The Create dropdown takes unique names from valid readings and subtracts names already present in SQLite. Deleting a card therefore makes that name available again without changing its readings.

`getLatestReading` first filters by exact crop name, then sorts matching readings by timestamp descending. It never assumes array order. Fixed `YYYY-MM-DDTHH:mm:ss` strings sort chronologically, and each crop's submitted readings have distinct timestamps.

`analyseCrop` applies one priority sequence:

1. Offline or Faulty -> Sensor Problem, N/A, Check sensor.
2. Online with an out-of-range numeric value -> Invalid Data, N/A, identify the field, Check reading.
3. Moisture below minimum -> Dry, normal water, Water crop.
4. Moisture from minimum through maximum inclusive -> Healthy, 0 L, Monitor.
5. Moisture above maximum -> Too Wet, 0 L, Stop watering.

For a valid Online reading, temperature above 35 C adds High temperature and rainfall of at least 5 mm adds Rain detected. The same function is used for dashboard cards and history.

Overall status is No Crops when there are no cards; Sensor Feed Unavailable when cards exist but no sensor request has succeeded; Critical for any Sensor Problem or Invalid Data; Watch for Dry, Too Wet or High temperature; otherwise Normal. Rain by itself does not cause Watch.

## Refresh and failure behaviour

Initial load requests cards and readings together. A crop-card failure blocks editable UI and offers Retry. A first sensor failure keeps cards visible, shows N/A sensor results, displays Sensor Feed Unavailable, leaves refresh as Never and disables Create. A later refresh failure preserves the previous readings and refresh time and displays an error banner. A successful refresh replaces reading state and recalculates all cards.

## Sensor JSON generation and verification

AI tool used: OpenAI ChatGPT (Codex). Final prompt:

> Generate a valid JSON array containing exactly 20 simulated SmartFarm sensor readings. Use these crop_name values exactly and create exactly 5 readings for each: Tomato, Lettuce, Wheat, Maize. Every object must contain exactly these fields: crop_name, timestamp, soil_moisture, temperature, rainfall, sensor_status, notes. Use timestamps in YYYY-MM-DDTHH:mm:ss format, distinct within each crop, and mix the array order so latest is not always last. Use status only Online, Offline or Faulty. Include exactly one older structurally valid reading with exactly one out-of-range numeric value, and do not make it latest. Make latest Tomato Online/Dry/above 35 C, latest Lettuce Online/Healthy, latest Wheat Online/Too Wet/rain at least 5 mm, and latest Maize Faulty. Return only JSON.

The generated draft was manually corrected because chronological groups made it too easy to accidentally select the last array item. The final array was deliberately interleaved, and the invalid Wheat moisture reading was moved to an older timestamp so it appears in history without controlling the dashboard.

Verification is both manual and automated. `validateReadings` checks the array length, exact seven-key shape, types, allowed names/statuses, real calendar timestamps, five readings per crop, unique timestamps within each crop and exactly one out-of-range numeric value. Tests also verify strict matching and greatest-timestamp selection. The final latest cases are Tomato 42%/38 C (Dry + High temperature), Lettuce 72% (Healthy), Wheat 61%/8 mm (Too Wet + Rain detected) and Maize Faulty (Sensor Problem).

## AI use and implementation decision

AI assisted with the project structure, validation logic, sample data, React components, CSS and automated tests. Each generated part was checked against the decision table and acceptance tests. Crop-name uniqueness is enforced at three levels: the sensor-backed Create dropdown, backend validation, and SQLite `UNIQUE`. Exact matching remains case-sensitive in frontend utilities and backend feed validation.

One deliberate decision was to keep analysis entirely in `frontend/src/utils/analysis.js`. The backend returns raw readings only, and both the card and history components call the same analysis function. This prevents the priority rules from drifting between views and respects the assignment's ownership boundary.

## Limitation

The sensor feed is a local static JSON file rather than live IoT hardware. Refresh sees file changes, but the application has no automatic polling or real-time push updates.

## Suggested demonstration

Use `VIDEO_SCRIPT.md` for a timed 3-5 minute recording checklist. Start from a clean database so the first screen has exactly three cards and Maize appears in Add Crop Card.
