# How to Select and Subset Data

This guide covers various methods for extracting subsets of unstructured grid
data based on spatial criteria.

## Coordinate-Based Selection

Select data at specific coordinates using `.ugrid.sel()`:

```python
import xugrid as xu

uda = xu.data.elevation_nl()

# Select at a single x coordinate (returns cross-section)
cross_section = uda.ugrid.sel(x=155_000)

# Select at a single y coordinate
cross_section = uda.ugrid.sel(y=463_000)

# Select a single point (returns scalar)
value = uda.ugrid.sel(x=155_000, y=463_000)
```

## Point Extraction

Extract values at multiple specific points:

```python
import numpy as np

# Define points of interest
x_points = np.array([155_000, 160_000, 165_000])
y_points = np.array([463_000, 465_000, 467_000])

# Extract values at those points
values = uda.ugrid.sel_points(x=x_points, y=y_points)
```

Control behavior for points outside the grid:

```python
# Raise error for out-of-bounds points (default)
values = uda.ugrid.sel_points(x=x_points, y=y_points, out_of_bounds="raise")

# Return NaN for out-of-bounds points
values = uda.ugrid.sel_points(x=x_points, y=y_points, out_of_bounds="warn")

# Silently return NaN
values = uda.ugrid.sel_points(x=x_points, y=y_points, out_of_bounds="ignore")

# Drop out-of-bounds points from result
values = uda.ugrid.sel_points(x=x_points, y=y_points, out_of_bounds="drop")
```

## Bounding Box Selection

Clip data to a rectangular region:

```python
# Define bounding box (xmin, ymin, xmax, ymax)
subset = uda.ugrid.clip_box(
    xmin=100_000,
    ymin=450_000,
    xmax=200_000,
    ymax=500_000
)
```

The result includes all faces that intersect the bounding box.

## Line Intersection

Extract data along a line (cross-section):

```python
from shapely.geometry import LineString

# Define a line
line = LineString([(100_000, 450_000), (200_000, 500_000)])

# Get intersection
section = uda.ugrid.intersect_line(
    start=line.coords[0],
    end=line.coords[1]
)
```

For complex paths with multiple segments:

```python
# Multi-segment line
linestring = LineString([
    (100_000, 450_000),
    (150_000, 475_000),
    (200_000, 500_000)
])

section = uda.ugrid.intersect_linestring(linestring)
```

## Selection with Polygon Mask

Use vector geometries to mask data:

```python
from shapely.geometry import box, Polygon
import geopandas as gpd

# Create a polygon
polygon = box(100_000, 450_000, 200_000, 500_000)

# Or load from file
gdf = gpd.read_file("region.gpkg")
polygon = gdf.geometry.iloc[0]

# Burn polygon to grid (creates mask)
mask = xu.burn_vector_geometry(polygon, uda)

# Apply mask
subset = uda.where(mask)
```

## Boolean Indexing

Use boolean arrays for custom selection:

```python
# Select faces where elevation > 0
above_sea_level = uda.where(uda > 0)

# Select faces within distance of a point
grid = uda.ugrid.grid
center_x, center_y = 155_000, 463_000
distance = np.sqrt(
    (grid.face_x - center_x)**2 +
    (grid.face_y - center_y)**2
)
nearby = uda.where(distance < 10_000)
```

## Temporal and Non-Spatial Selection

For multi-dimensional data, use standard xarray indexing for non-spatial
dimensions:

```python
# Load time-varying data
uds = xu.open_dataset("model_output.nc")
temp = uds["temperature"]

# Select specific time
temp_t0 = temp.isel(time=0)

# Select time range
temp_range = temp.sel(time=slice("2020-01", "2020-06"))

# Combine spatial and temporal selection
subset = temp.sel(time="2020-03-15").ugrid.clip_box(
    xmin=100_000, ymin=450_000,
    xmax=200_000, ymax=500_000
)
```

## Selecting by Grid Topology

Select based on grid connectivity:

```python
grid = uda.ugrid.grid

# Get face indices that touch a specific node
node_idx = 100
connected_faces = grid.node_face_connectivity[node_idx]
connected_faces = connected_faces[connected_faces != grid.fill_value]

# Select those faces
subset = uda.isel({grid.face_dimension: connected_faces})
```

## Getting Indices Instead of Data

Sometimes you need indices for further processing:

```python
# Get face indices within bounding box
grid = uda.ugrid.grid
face_x = grid.face_x
face_y = grid.face_y

in_box = (
    (face_x >= 100_000) & (face_x <= 200_000) &
    (face_y >= 450_000) & (face_y <= 500_000)
)
indices = np.where(in_box)[0]

# Use indices for multiple arrays
subset_a = uda_a.isel({grid.face_dimension: indices})
subset_b = uda_b.isel({grid.face_dimension: indices})
```
