# Ukraine Grid and Sector Visualization with PostGIS and Python

This project demonstrates how to visualize Ukraine’s borders, build geospatial grids, and generate directional sectors using Python (Folium, GeoPandas, Shapely) and PostgreSQL with PostGIS.  
It was completed as a test project for Vodafone to showcase geospatial database design, geometry handling, and map visualization.

---

## 📘 Database Schema

```sql
ukraine_borders(id SERIAL, region TEXT, geom GEOMETRY(Point,4326))
ukraine_grid_points(id SERIAL, geom GEOMETRY(Point,4326))
ukraine_grid_sectors(id SERIAL, point_id INT, azimuth INT, sector GEOMETRY(Polygon,4326))
grid_point_sector_intersections(id SERIAL, point_id INT, sector_id INT)
```

---

## 🧩 Libraries Used

- **psycopg2** — PostgreSQL driver for executing SQL queries.  
- **folium** — Interactive map visualization (Leaflet.js).  
- **geopandas** — Extension of Pandas for spatial data.  
- **shapely** — Geometric operations for polygons and points.  
- **math** — Used for trigonometric calculations (sin/cos).  
- **IPython.display** — To show HTML maps in Jupyter Notebook.

---

## 🗺️ Tasks 1–2: Loading and Visualizing Ukraine’s Border

**Goal:** Import Ukraine’s border coordinates into PostgreSQL and visualize them in Python or via Leaflet.

**Steps:**  
1. Download coordinates as text files:
   - `coord_border_ukraine.txt` — national border  
   - `ua_regions_full.txt` — regional borders with `#Region` markers  
2. Load into PostgreSQL with PostGIS geometry type `GEOMETRY(Point,4326)` (EPSG:4326 = WGS84).

```python
cur.execute("""
    INSERT INTO ukraine_borders (region, geom)
    VALUES (%s, ST_SetSRID(ST_MakePoint(%s, %s), 4326));
""", ("Ukraine", longitude, latitude))
```

![Figure 1 – Database creation example](images/figure1.png)

**Visualization in Folium:**

```sql
SELECT ST_X(geom), ST_Y(geom)
FROM ukraine_borders
WHERE region = 'Ukraine';
```

The average coordinate is used to center the map.  
Results are saved as `map_ukraine.html`.

![Figure 3 – Ukraine border visualization](images/figure3.png)

---

## 🧮 Tasks 3–5: Grid Generation

**Goal:** Split Ukraine’s area into equal grid squares (~1 km or 10 km side length).  
Store grid vertices in the database and display them on a map.

**Algorithm Overview:**
- Define bounding box of Ukraine using min/max coordinates.  
- Create square cells using `ST_MakeEnvelope()` and `generate_series()`.  
- Intersect or clip the grid with the country polygon.

**Example SQL:**

```sql
WITH poly AS (
  SELECT ST_MakePolygon(ST_MakeLine(geom ORDER BY id))::geometry(Polygon,4326) AS geom
  FROM ukraine_borders
  WHERE region = 'Ukraine'
),
ua AS (
  SELECT ST_Transform(geom, 32636) AS g FROM poly
),
grid AS (
  SELECT ST_MakeEnvelope(x, y, x+10000, y+10000, 32636) AS cell
  FROM bounds,
       generate_series((SELECT minx FROM bounds)::bigint, (SELECT maxx FROM bounds)::bigint, 10000) AS x,
       generate_series((SELECT miny FROM bounds)::bigint, (SELECT maxy FROM bounds)::bigint, 10000) AS y
)
SELECT ST_AsEWKB(cell) FROM grid;
```

10×10 km grids were chosen for full-country coverage without memory overload.  
1×1 km grids were built for Kyiv region only.

![Figure 6 – Grid visualization example](images/figure6.png)

---

## 🧭 Tasks 6–7: Sector Generation and Intersections

**Goal:** From each vertex, draw 3 directional sectors (azimuths 0°, 120°, 240°) with 60° spread and 5 km radius.  
Save intersections between sectors and grid vertices in the database.

```python
def build_sector(point, azimuth_deg, spread=60, radius=5000, n=30):
    # Builds a circular sector polygon from a given point
    angles = [azimuth_deg - spread/2 + i*(spread/n) for i in range(n+1)]
    arc_points = []
    for ang in angles:
        ang_rad = math.radians(ang)
        dx = radius * math.sin(ang_rad)
        dy = radius * math.cos(ang_rad)
        arc_points.append((lon + dx/111320, lat + dy/111320))
    return Polygon([point] + arc_points + [point])
```

```sql
CREATE TABLE ukraine_grid_sectors (
    id SERIAL PRIMARY KEY,
    point_id INT,
    azimuth INT,
    sector geometry(Polygon,4326)
);
```

**Intersection Detection:**

```sql
SELECT p.id AS point_id, s.id AS sector_id
FROM ukraine_grid_points p
JOIN ukraine_grid_sectors s
  ON ST_Contains(s.sector, p.geom);
```

Results are saved into `grid_point_sector_intersections` table.

**Visualization:**  
- Squares → blue borders (GeoJSON)  
- Sectors → colored polygons (0° red, 120° green, 240° orange)

![Figure 9 – Sector visualization example](images/figure9.png)

---

## 📊 Results

Two visual configurations were tested:  
- `radius=5000` → compact demonstration sectors  
- `radius=50000` → larger-scale visualization for clarity

Each sector successfully overlays grid points, showing geometric coverage.

---

## 🗂️ Output Files

| File | Description |
|------|--------------|
| `map_ukraine.html` | Border map of Ukraine |
| `map_kyivska.html` | Kyiv region visualization |
| `map_ukraine_grid_points.html` | Grid vertices map |
| `map_grid_with_sectors.html` | Full sector overlay map |

---

## 🧠 Technologies

| Category | Tools |
|-----------|--------|
| Database | PostgreSQL, PostGIS |
| Visualization | Folium (Leaflet.js), GeoPandas |
| Geometry | Shapely |
| Scripting | Python 3.12 |
| Data Storage | WKB/WKT geometry serialization |

---

## 📷 Suggested Image Files

```
/images/
  figure1.png
  figure2.png
  figure3.png
  figure4.png
  figure5.png
  figure6.png
  figure7.png
  figure8.png
  figure9.png
  figure10.png
```

---

© 2025 — Prepared by **Angelina Ivanova** as a demonstration of GIS and database integration for geospatial visualization.
