# GIS-Based Land Boundary Mapping Using Convex Hull Algorithms

A full-stack project that computes land parcel boundaries from GPS survey
points using the **Graham Scan convex hull algorithm**, and visualizes the
result on an interactive map.

## Project Structure

```
land-boundary-project/
├── backend/
│   ├── main.py            FastAPI app (REST API)
│   ├── convex_hull.py     Graham Scan + area/perimeter geometry (pure Python)
│   └── requirements.txt
├── frontend/
│   └── index.html         Leaflet-based map UI (vanilla JS, no build step)
└── README.md
```

## How It Works

1. **Input**: GPS survey points (lat/lng) for a land parcel — added by
   clicking on the map, uploaded, or generated synthetically for demos.
2. **Convex Hull (Graham Scan)**:
   - Find the point with the lowest latitude (the pivot).
   - Sort all other points by polar angle around the pivot.
   - Sweep through sorted points, using a stack, popping whenever the last
     three points make a clockwise turn (non-convex).
   - What remains is the minimal convex polygon enclosing all points —
     O(n log n) time complexity.
3. **Area**: computed via an equirectangular projection (lat/lng → local
   meters) + the Shoelace formula.
4. **Perimeter**: sum of haversine (great-circle) distances between
   consecutive hull vertices.
5. **Output**: a GeoJSON `Polygon` feature with area/perimeter properties,
   rendered directly on the Leaflet map.

## Running the Backend

```bash
cd backend
pip install -r requirements.txt
uvicorn main:app --reload --port 8000
```

Visit `http://localhost:8000/docs` for interactive Swagger API docs.

## Running the Frontend

No build tools needed — just open `frontend/index.html` in a browser
(or serve it with `python -m http.server 5500` from the `frontend/` folder).
Make sure the "API Base URL" field in the sidebar matches your backend
(default: `http://localhost:8000`).

## Using It

1. Open the frontend in your browser.
2. Either:
   - Click points directly on the map (each click = one GPS survey point), or
   - Click **"Generate Synthetic Points"** to auto-populate a demo cluster
     around the current map center.
3. Click **"Compute Convex Hull"** — the backend runs Graham Scan and returns
   the boundary polygon, which is drawn in green with area/perimeter stats.
4. Click **"Clear Parcel"** to reset and start a new parcel.

## API Reference

| Method | Endpoint                              | Description                          |
|--------|----------------------------------------|---------------------------------------|
| POST   | `/api/parcels/{id}/points`             | Add one GPS point to a parcel         |
| GET    | `/api/parcels/{id}/points`             | List all raw points for a parcel      |
| GET    | `/api/parcels/{id}/hull`               | Compute & return convex hull GeoJSON  |
| DELETE | `/api/parcels/{id}`                    | Clear all points for a parcel         |
| GET    | `/api/parcels`                         | List all parcel IDs                   |
| POST   | `/api/parcels/{id}/generate`           | Generate synthetic demo points        |

## Datasets: Real and Synthetic

Two ready-made scripts live in `backend/data/`. Both output a CSV
(`lat,lng,label`) that you can load into the app via the **Import CSV
Dataset** button in the frontend, or by POSTing the file to
`/api/parcels/{id}/import`.

### 1. Real dataset — `fetch_real_data.py`

Pulls **real building/land footprint coordinates** from OpenStreetMap using
the free Overpass API. Requires internet access.

```bash
cd backend/data
python fetch_real_data.py --lat 13.0827 --lng 80.2707 --radius 500 --out real_dataset.csv
```

- `--lat` / `--lng` — center point to search around
- `--radius` — search radius in meters
- `--limit` — how many nearby buildings to pull points from (default 1)
- `--out` — output CSV filename

This is genuine geographic data — good for showing your algorithm works on
real-world coordinates, not just made-up numbers.

### 2. Synthetic dataset — `generate_synthetic.py`

Generates a **synthetic point cloud** with no internet required — useful for
demos, reproducible testing, and controlled edge cases (e.g. interior points
that should get dropped by the hull algorithm).

```bash
cd backend/data
python generate_synthetic.py --lat 13.0827 --lng 80.2707 --count 25 --shape ring --out synthetic_dataset.csv
```

- `--shape ring` — points scattered near a rough circular boundary (looks
  like a realistic parcel edge survey)
- `--shape random` — points uniformly scattered (good for testing that
  interior points get correctly excluded from the hull)
- `--seed 42` — set for reproducible output
- `--out` — output CSV filename

### Loading a dataset into the app

**Option A — via the frontend:** click "Choose File" next to *Import
Dataset*, pick your CSV, then click **Import CSV Dataset**.

**Option B — via curl:**
```bash
curl -F "file=@real_dataset.csv" http://localhost:8000/api/parcels/my-parcel/import
```

### Other dataset sources worth knowing about

- **Survey of India / Bhuvan (ISRO)** — open cadastral/village boundary data
  (India-specific, may need shapefile conversion).
- **Kaggle** — search "land parcel," "cadastral boundary," "GPS survey."
- **Your own field survey** — walk a plot boundary with a phone GPS logger
  app, export as CSV (columns `lat,lng`), and import it the same way.

## Known Limitation (worth discussing in your report)

Convex hull always produces a **convex** polygon — it will "cut corners" on
any parcel that's actually concave (L-shaped plots, irregular rural land,
etc.), since it only keeps the outermost points. For concave boundary
tracing you'd need a different algorithm (e.g., **Alpha Shapes** or
**Concave Hull / k-nearest neighbors approach**). This is a good point to
raise in your project viva as a "limitations and future work" discussion.

## Suggested Extensions

- Swap the in-memory dict for **PostgreSQL + PostGIS** for persistent,
  spatially-indexed storage (use `ST_ConvexHull` to cross-check your manual
  implementation).
- Add **user authentication** so multiple surveyors can manage their own parcels.
- Add **concave hull** support and let the user toggle between the two.
- Add **CSV/GeoJSON upload** for bulk point import instead of manual clicking.
- Deploy backend to Render/Railway and frontend to Vercel/Netlify for a live demo link.
