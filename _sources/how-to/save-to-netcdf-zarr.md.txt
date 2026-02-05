# How to Save UGRID Data

This guide covers exporting xugrid data to various formats while preserving
UGRID conventions.

## Saving to NetCDF

NetCDF is the standard format for UGRID data:

```python
import xugrid as xu

uda = xu.data.elevation_nl()

# Save UgridDataArray
uda.ugrid.to_netcdf("elevation.nc")

# Save UgridDataset
uds = xu.open_dataset("model.nc")
uds.ugrid.to_netcdf("output.nc")
```

The output file will include all UGRID topology variables and CF-compliant
metadata.

## Saving to Zarr

Zarr is preferred for large datasets and cloud storage:

```python
# Save to local Zarr store
uda.ugrid.to_zarr("elevation.zarr")

# Save to cloud storage (requires fsspec/s3fs)
uda.ugrid.to_zarr("s3://bucket/elevation.zarr")

# Append to existing store (e.g., for time series)
uda.ugrid.to_zarr("timeseries.zarr", mode="a", append_dim="time")
```

Zarr supports chunked, compressed storage suitable for parallel access.

## Converting to Xarray Dataset

To get a standard xarray Dataset (for custom I/O or further processing):

```python
# Convert UgridDataArray to Dataset
ds = uda.ugrid.to_dataset()

# Convert UgridDataset to Dataset
ds = uds.ugrid.to_dataset()

# Now you can use any xarray export method
ds.to_netcdf("custom.nc", encoding={...})
```

## Exporting to GeoDataFrame

For GIS workflows, export to geopandas:

```python
import geopandas as gpd

# Export faces as polygons
gdf = uda.ugrid.to_geodataframe()

# Export to various GIS formats
gdf.to_file("output.gpkg", driver="GPKG")
gdf.to_file("output.shp")
gdf.to_parquet("output.parquet")
```

You can also export specific grid elements:

```python
# Access the grid
grid = uda.ugrid.grid

# Export nodes as points
gdf_nodes = gpd.GeoDataFrame(
    geometry=gpd.points_from_xy(grid.node_x, grid.node_y)
)

# Export edges as lines
gdf_edges = grid.to_geodataframe(dim=grid.edge_dimension)
```

## Saving Regridder Weights

Regridder weights can be saved for reuse:

```python
# Create and use a regridder
regridder = xu.OverlapRegridder(source=uda, target=target_grid)
result = regridder.regrid(uda)

# Save weights to Dataset
weights_ds = regridder.to_dataset()
weights_ds.to_netcdf("regridder_weights.nc")

# Later, reload the regridder
weights_ds = xr.open_dataset("regridder_weights.nc")
regridder = xu.OverlapRegridder.from_dataset(weights_ds)
```

## Controlling Output Encoding

For fine-grained control over the output format:

```python
# Get xarray Dataset first
ds = uda.ugrid.to_dataset()

# Define encoding (compression, chunking, etc.)
encoding = {
    "elevation": {
        "dtype": "float32",
        "zlib": True,
        "complevel": 4,
    }
}

# Save with custom encoding
ds.to_netcdf("compressed.nc", encoding=encoding)
```

For Zarr with specific chunks:

```python
ds = uds.ugrid.to_dataset()

encoding = {
    "temperature": {"chunks": {"time": 10, "mesh2d_nFaces": 10000}},
}

ds.to_zarr("chunked.zarr", encoding=encoding)
```

## Best Practices

1. **Use Zarr for large datasets** - Better parallel read/write performance
2. **Preserve CRS information** - Set CRS before saving: `uda.ugrid.set_crs("EPSG:28992")`
3. **Use compression for NetCDF** - Reduces file size significantly
4. **Choose appropriate chunks for Zarr** - Balance between chunk size and access patterns
5. **Include metadata** - Add attributes to document your data:

```python
uda.attrs["source"] = "Model simulation v2.0"
uda.attrs["units"] = "meters"
uda.ugrid.to_netcdf("documented.nc")
```
