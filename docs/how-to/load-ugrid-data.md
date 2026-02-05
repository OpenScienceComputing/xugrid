# How to Load UGRID Data

This guide covers the various ways to load unstructured grid data into xugrid.

## Loading from NetCDF Files

The most common way to load UGRID data is from NetCDF files that follow
UGRID conventions.

**Load a single file (lazy, uses Dask):**

```python
import xugrid as xu

# Returns UgridDataset with lazy Dask arrays
uds = xu.open_dataset("model_output.nc")
```

**Load a single file into memory:**

```python
# Loads all data into memory immediately
uds = xu.load_dataset("model_output.nc")
```

**Load a single DataArray:**

```python
# When you know the file contains a single data variable
uda = xu.open_dataarray("elevation.nc")
```

## Loading Multiple Files

For model output split across multiple files (e.g., time steps):

```python
# Combine multiple files along a dimension
uds = xu.open_mfdataset("output_*.nc")

# Or with explicit file list
files = ["output_001.nc", "output_002.nc", "output_003.nc"]
uds = xu.open_mfdataset(files)
```

## Loading from Zarr

Zarr is ideal for cloud storage and parallel access:

```python
# Local Zarr store
uds = xu.open_zarr("model_output.zarr")

# Remote Zarr store (e.g., S3)
uds = xu.open_zarr("s3://bucket/model_output.zarr")
```

## Creating from Xarray Objects

If you have an xarray Dataset that contains UGRID topology variables:

```python
import xarray as xr
import xugrid as xu

# Standard xarray load
ds = xr.open_dataset("ugrid_file.nc")

# Convert to UgridDataset (auto-detects UGRID topology)
uds = xu.UgridDataset(ds)
```

## Creating from Structured Data

Convert regular gridded (raster) data to unstructured format:

```python
import xarray as xr
import xugrid as xu

# Load structured data
da = xr.open_dataarray("raster.nc")

# Convert to unstructured
uda = xu.UgridDataArray.from_structured2d(da)
```

For curvilinear grids, provide bounds:

```python
# With explicit coordinate bounds
uda = xu.UgridDataArray.from_structured2d(
    da,
    x_bounds=x_bounds_array,
    y_bounds=y_bounds_array
)
```

## Creating from Scratch

Build a grid manually from numpy arrays:

```python
import numpy as np
import xarray as xr
import xugrid as xu

# Define node coordinates
nodes_x = np.array([0.0, 1.0, 2.0, 0.5, 1.5])
nodes_y = np.array([0.0, 0.0, 0.0, 1.0, 1.0])

# Define faces as node indices (triangles in this case)
# Each row is a face, values are node indices, -1 for fill
face_node_connectivity = np.array([
    [0, 1, 3, -1],
    [1, 2, 4, -1],
    [1, 4, 3, -1],
])

# Create the grid topology
grid = xu.Ugrid2d(
    node_x=nodes_x,
    node_y=nodes_y,
    fill_value=-1,
    face_node_connectivity=face_node_connectivity,
)

# Create data on faces
face_data = np.array([1.0, 2.0, 3.0])
da = xr.DataArray(face_data, dims=[grid.face_dimension])

# Combine into UgridDataArray
uda = xu.UgridDataArray(da, grid)
```

## Using Sample Data

Xugrid includes sample datasets for learning and testing:

```python
import xugrid as xu

# Netherlands elevation data
uda = xu.data.elevation_nl()

# San Diego ADH model
uds = xu.data.adh_san_diego()

# Simple disk mesh
uda = xu.data.disk()
```

## Inspecting Loaded Data

After loading, inspect the grid topology:

```python
uds = xu.open_dataset("model.nc")

# View grid names
print(uds.ugrid.names)

# Access specific grid
grid = uds.ugrid.grids[0]

# Grid properties
print(f"Nodes: {grid.n_node}")
print(f"Edges: {grid.n_edge}")
print(f"Faces: {grid.n_face}")
print(f"Bounds: {grid.bounds}")
```
