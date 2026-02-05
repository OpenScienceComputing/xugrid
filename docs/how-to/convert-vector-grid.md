# How to Convert Between Vector and Grid Formats

This guide covers converting between vector geometries (points, lines, polygons)
and unstructured grids.

## Exporting Grid to GeoDataFrame

Convert grid faces to polygons:

```python
import xugrid as xu

uda = xu.data.elevation_nl()

# Export to GeoDataFrame with data values
gdf = uda.ugrid.to_geodataframe()

# Result is a GeoDataFrame with polygon geometry and data column
print(gdf.head())
```

Export grid topology without data:

```python
grid = uda.ugrid.grid

# Faces as polygons
gdf_faces = grid.to_geodataframe(dim=grid.face_dimension)

# Edges as lines
gdf_edges = grid.to_geodataframe(dim=grid.edge_dimension)

# Nodes as points
import geopandas as gpd
gdf_nodes = gpd.GeoDataFrame(
    geometry=gpd.points_from_xy(grid.node_x, grid.node_y),
    crs=uda.ugrid.crs
)
```

## Creating Grid from GeoDataFrame

Convert polygons to an unstructured grid:

```python
import geopandas as gpd
import xugrid as xu

# Load polygon data
gdf = gpd.read_file("regions.gpkg")

# Convert to UgridDataset
uds = xu.UgridDataset.from_geodataframe(gdf)

# Access specific columns as UgridDataArray
population = uds["population"]
```

## Burning Vector Geometries to Grid

Rasterize vector features onto an existing grid:

```python
import geopandas as gpd
import xugrid as xu
from shapely.geometry import box

# Load grid
uda = xu.data.elevation_nl()

# Create or load polygon(s)
polygon = box(100_000, 450_000, 150_000, 500_000)

# Burn to grid (returns boolean mask by default)
mask = xu.burn_vector_geometry(polygon, uda)

# Apply mask
subset = uda.where(mask)
```

Burn with specific values:

```python
# Burn with a specific value
burned = xu.burn_vector_geometry(polygon, uda, fill_value=1, all_touched=False)

# Burn multiple polygons with different values
gdf = gpd.GeoDataFrame({
    "geometry": [polygon1, polygon2, polygon3],
    "value": [1, 2, 3]
})
burned = xu.burn_vector_geometry(gdf, uda, column="value")
```

Control which faces are selected:

```python
# Only faces fully inside polygon
mask = xu.burn_vector_geometry(polygon, uda, all_touched=False)

# Any face that touches polygon (default)
mask = xu.burn_vector_geometry(polygon, uda, all_touched=True)
```

## Snapping Geometries to Grid

Snap vector geometries to grid coordinates:

```python
from shapely.geometry import Point, LineString

uda = xu.data.elevation_nl()
grid = uda.ugrid.grid

# Snap points to nearest nodes
points = [Point(155_000, 463_000), Point(160_000, 465_000)]
snapped = xu.snap_to_grid(points, grid, max_distance=1000)
```

For line features:

```python
line = LineString([(155_000, 463_000), (160_000, 465_000)])
snapped_line = xu.snap_to_grid(line, grid, max_distance=500)
```

## Polygonizing Grid Data

Convert grid faces back to vector polygons based on values:

```python
# Create categorical data
categories = (uda > 0).astype(int)  # 0 = below sea level, 1 = above

# Polygonize - merge adjacent faces with same value
gdf = xu.polygonize(categories)
```

## Triangulating Polygons

Convert polygons to triangular mesh:

```python
from shapely.geometry import Polygon
import xugrid as xu

# Define polygon
polygon = Polygon([(0, 0), (10, 0), (10, 10), (5, 15), (0, 10)])

# Triangulate
grid = xu.earcut_triangulate_polygons([polygon])

# Create data on the triangulated mesh
import numpy as np
import xarray as xr

data = np.random.rand(grid.n_face)
da = xr.DataArray(data, dims=[grid.face_dimension])
uda = xu.UgridDataArray(da, grid)
```

## Working with CRS (Coordinate Reference Systems)

Set and transform coordinate systems:

```python
# Set CRS
uda_with_crs = uda.ugrid.set_crs("EPSG:28992")  # Dutch RD

# Transform to different CRS
uda_wgs84 = uda_with_crs.ugrid.to_crs("EPSG:4326")

# Check CRS
print(uda.ugrid.crs)
```

Export with CRS preserved:

```python
# GeoDataFrame will have CRS
gdf = uda_with_crs.ugrid.to_geodataframe()
print(gdf.crs)  # EPSG:28992

# NetCDF will have grid_mapping
uda_with_crs.ugrid.to_netcdf("with_crs.nc")
```

## Integration with Shapely

Direct use of Shapely geometries:

```python
from shapely.geometry import Point, LineString, Polygon, MultiPolygon
from shapely.ops import unary_union

# Create complex selection geometry
poly1 = box(100_000, 450_000, 125_000, 475_000)
poly2 = box(150_000, 475_000, 175_000, 500_000)
combined = unary_union([poly1, poly2])

# Use with burn
mask = xu.burn_vector_geometry(combined, uda)
```

## Integration with GeoPandas

Full round-trip workflow:

```python
import geopandas as gpd
import xugrid as xu

# 1. Load vector data
regions = gpd.read_file("regions.gpkg")

# 2. Load unstructured grid data
uda = xu.data.elevation_nl()

# 3. Compute zonal statistics
results = []
for idx, row in regions.iterrows():
    mask = xu.burn_vector_geometry(row.geometry, uda)
    masked = uda.where(mask)
    results.append({
        "region": row["name"],
        "mean_elevation": float(masked.mean()),
        "max_elevation": float(masked.max()),
    })

# 4. Add results back to GeoDataFrame
stats_df = pd.DataFrame(results)
regions = regions.merge(stats_df, left_on="name", right_on="region")

# 5. Save
regions.to_file("regions_with_stats.gpkg")
```
