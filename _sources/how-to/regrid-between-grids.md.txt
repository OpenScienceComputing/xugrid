# How to Regrid Between Grids

This guide covers interpolating data from one unstructured grid to another,
or between structured and unstructured grids.

## Choosing a Regridding Method

Xugrid provides several regridders for different use cases:

- **CentroidLocatorRegridder**: Fast, finds source face containing target centroid.
  Best for: Quick approximations, similar resolution grids.

- **OverlapRegridder**: Computes area-weighted overlap between faces.
  Best for: Conservation of quantities, different resolution grids.

- **RelativeOverlapRegridder**: Like OverlapRegridder but normalizes by target area.
  Best for: When target faces may extend beyond source grid.

- **BarycentricInterpolator**: Smooth interpolation using barycentric coordinates.
  Best for: Continuous fields, visualization.

## Basic Regridding

```python
import xugrid as xu

# Load source and target grids
source = xu.data.elevation_nl()
target = xu.data.disk()  # Different grid

# Create regridder
regridder = xu.CentroidLocatorRegridder(source=source, target=target)

# Apply to data
result = regridder.regrid(source)
```

## Area-Weighted Regridding

For conservative regridding that preserves integrated quantities:

```python
# OverlapRegridder computes area-weighted values
regridder = xu.OverlapRegridder(source=source, target=target)
result = regridder.regrid(source)
```

Specify aggregation method:

```python
# Area-weighted mean (default)
regridder = xu.OverlapRegridder(source, target, method="mean")

# Other methods
regridder = xu.OverlapRegridder(source, target, method="minimum")
regridder = xu.OverlapRegridder(source, target, method="maximum")
regridder = xu.OverlapRegridder(source, target, method="mode")  # Most common value
regridder = xu.OverlapRegridder(source, target, method="median")
regridder = xu.OverlapRegridder(source, target, method=("percentile", 90))
```

## Smooth Interpolation

For continuous fields where you want smooth transitions:

```python
regridder = xu.BarycentricInterpolator(source=source, target=target)
result = regridder.regrid(source)
```

## Regridding to Structured Grids

Convert unstructured data to a regular grid (rasterize):

```python
import numpy as np

# Define target resolution
x = np.linspace(source.ugrid.grid.bounds[0], source.ugrid.grid.bounds[2], 100)
y = np.linspace(source.ugrid.grid.bounds[1], source.ugrid.grid.bounds[3], 100)

# Rasterize
raster = source.ugrid.rasterize(x=x, y=y)
```

Or match an existing raster:

```python
import xarray as xr

# Load reference raster
reference = xr.open_dataarray("reference_raster.nc")

# Rasterize to match
raster = source.ugrid.rasterize_like(reference)
```

## Regridding from Structured to Unstructured

```python
import xarray as xr
import xugrid as xu

# Load structured data
raster = xr.open_dataarray("dem.nc")

# Load target unstructured grid
target_grid = xu.data.disk().ugrid.grid

# Convert raster to unstructured first
source_unstructured = xu.UgridDataArray.from_structured2d(raster)

# Then regrid to target
regridder = xu.OverlapRegridder(source_unstructured, target_grid)
result = regridder.regrid(source_unstructured)
```

## Regridding Multi-Dimensional Data

Regridders work with data that has additional dimensions (time, depth, etc.):

```python
# Load time-varying data
uds = xu.open_dataset("model_output.nc")
temperature = uds["temperature"]  # Shape: (time, faces)

# Create regridder (only needs grid info)
regridder = xu.OverlapRegridder(source=temperature, target=target_grid)

# Regrid all timesteps at once
temp_regridded = regridder.regrid(temperature)
```

## Saving and Reusing Regridder Weights

Computing regridder weights can be expensive. Save them for reuse:

```python
# Create regridder (computes weights)
regridder = xu.OverlapRegridder(source=source, target=target)

# Save weights
weights_ds = regridder.to_dataset()
weights_ds.to_netcdf("weights.nc")

# Later, reload
import xarray as xr
weights_ds = xr.open_dataset("weights.nc")
regridder = xu.OverlapRegridder.from_dataset(weights_ds)

# Apply to new data on same grids
new_result = regridder.regrid(new_data)
```

## Regridding with Dask

Regridders automatically work with Dask arrays:

```python
# Load lazy data
uds = xu.open_dataset("large_model.nc", chunks={"time": 10})
data = uds["variable"]

# Create regridder
regridder = xu.OverlapRegridder(source=data, target=target)

# Regrid returns Dask array (lazy)
result = regridder.regrid(data)

# Compute when ready
result_computed = result.compute()

# Or save directly (triggers computation)
result.ugrid.to_zarr("regridded.zarr")
```

## Handling Missing Data

Control how NaN values are handled:

```python
# By default, NaN in source propagates to result
regridder = xu.OverlapRegridder(source, target, method="mean")

# For data with many NaNs, consider filling first
filled = source.fillna(0)
result = regridder.regrid(filled)

# Or interpolate NaN values
interpolated = source.ugrid.laplace_interpolate(
    xy_weights=True,
    direct_solve=True
)
result = regridder.regrid(interpolated)
```

## Performance Tips

1. **Reuse regridders**: Creating the regridder computes weights once; `regrid()` is fast.

2. **Use appropriate method**: `CentroidLocatorRegridder` is fastest but least accurate.

3. **Chunk wisely with Dask**: Large spatial chunks, small time chunks work well.

4. **Save weights for repeated use**: Especially for expensive `OverlapRegridder`.

5. **Consider target grid resolution**: Regridding to much finer grids doesn't add information.
