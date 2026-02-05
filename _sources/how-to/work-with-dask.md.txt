# How to Work with Dask for Large Datasets

This guide covers using Dask with xugrid to handle datasets too large to fit
in memory, and to parallelize computations.

## Loading Data Lazily

By default, `xu.open_dataset()` loads data lazily using Dask:

```python
import xugrid as xu

# Data is NOT loaded into memory
uds = xu.open_dataset("large_model.nc")

# Check that data is lazy (Dask array)
print(uds["temperature"].data)
# <dask.array<...>>
```

Specify chunks for optimal performance:

```python
# Chunk along time dimension
uds = xu.open_dataset("large_model.nc", chunks={"time": 10})

# Or let xarray choose chunks
uds = xu.open_dataset("large_model.nc", chunks="auto")
```

## Loading from Zarr

Zarr is inherently chunked and works seamlessly with Dask:

```python
# Zarr stores are naturally lazy
uds = xu.open_zarr("large_model.zarr")

# Remote data (S3, GCS, Azure)
uds = xu.open_zarr("s3://bucket/model.zarr")
```

## Lazy Computations

Operations on Dask-backed arrays are lazy:

```python
uds = xu.open_dataset("large_model.nc", chunks={"time": 10})
temp = uds["temperature"]

# These operations build a task graph, no computation yet
temp_celsius = temp - 273.15
temp_mean = temp_celsius.mean(dim="time")
temp_anomaly = temp_celsius - temp_mean

# Compute when ready
result = temp_anomaly.compute()
```

## Triggering Computation

Several ways to trigger computation:

```python
# Explicit compute
result = lazy_array.compute()

# Load into memory
result = lazy_array.load()

# Save to file (computes as it writes)
lazy_array.ugrid.to_zarr("output.zarr")
lazy_array.ugrid.to_netcdf("output.nc")

# Plotting (computes the visible data)
lazy_array.ugrid.plot()
```

## Using a Dask Cluster

For large computations, use a distributed cluster:

```python
from dask.distributed import Client, LocalCluster

# Local cluster (uses all CPU cores)
cluster = LocalCluster()
client = Client(cluster)

# Now computations are distributed
uds = xu.open_dataset("large_model.nc", chunks={"time": 10})
result = uds["temperature"].mean(dim="time").compute()

# View progress in dashboard
print(client.dashboard_link)
```

For HPC or cloud:

```python
# Connect to existing scheduler
client = Client("tcp://scheduler-address:8786")

# Or use dask-jobqueue for HPC
from dask_jobqueue import SLURMCluster
cluster = SLURMCluster(cores=4, memory="16GB")
cluster.scale(jobs=10)
client = Client(cluster)
```

## Regridding with Dask

Regridders work with Dask arrays:

```python
source = xu.open_dataset("source.nc", chunks={"time": 10})["var"]
target = xu.data.disk()

# Create regridder
regridder = xu.OverlapRegridder(source=source, target=target)

# Regrid is lazy
result = regridder.regrid(source)
print(type(result.data))  # Dask array

# Save lazily
result.ugrid.to_zarr("regridded.zarr")
```

## Chunking Strategies

Unstructured grids have a single spatial dimension (faces), which affects
chunking strategies:

```python
# Good: chunk along non-spatial dimensions
uds = xu.open_dataset("model.nc", chunks={
    "time": 10,
    "layer": 5,
    # Don't chunk the face dimension - it's the spatial dimension
})

# The face dimension is typically not chunked because:
# 1. Spatial operations need all faces
# 2. Grid topology connects faces in complex patterns
```

For very large grids, consider partitioning instead of chunking:

```python
# Partition spatially (requires pymetis)
partitions = uds.ugrid.partition(n_parts=4)

# Process partitions separately
results = []
for part in partitions:
    result = process(part)
    results.append(result)

# Merge back
combined = xu.merge_partitions(results)
```

## Memory Management

Monitor and control memory usage:

```python
# Check chunk sizes
uds = xu.open_dataset("model.nc", chunks={"time": 10})
print(uds["temperature"].data)  # Shows chunk sizes

# Rechunk if needed
rechunked = uds["temperature"].chunk({"time": 5})

# Persist intermediate results to cluster memory
from dask.distributed import Client
client = Client()

intermediate = expensive_computation(uds)
intermediate = client.persist(intermediate)  # Keep in cluster memory

# Use for multiple downstream computations
result1 = analysis1(intermediate)
result2 = analysis2(intermediate)
```

## Writing Large Results

Write results efficiently:

```python
# Zarr is best for Dask - writes chunks in parallel
result.ugrid.to_zarr("output.zarr")

# NetCDF works but is sequential
result.ugrid.to_netcdf("output.nc")

# For very large results, compute in chunks
for t in range(0, n_times, chunk_size):
    chunk = result.isel(time=slice(t, t + chunk_size)).compute()
    chunk.ugrid.to_netcdf(f"output_{t:04d}.nc")
```

## Parallel I/O with Zarr

Zarr enables parallel reads and writes:

```python
# Write in parallel (each worker writes its chunks)
result.ugrid.to_zarr("output.zarr")

# Read in parallel (each worker reads needed chunks)
uds = xu.open_zarr("output.zarr")
result = uds["var"].mean().compute()  # Parallel reduction
```

## Debugging Dask Computations

Visualize the task graph:

```python
# Install graphviz for visualization
result.data.visualize("task_graph.png")

# Check task count
print(len(result.data.__dask_graph__()))

# Profile execution
from dask.diagnostics import ProgressBar
with ProgressBar():
    result.compute()
```

## Common Pitfalls

1. **Accidental compute**: Some operations trigger computation unexpectedly:

```python
# These compute immediately:
len(uda)           # Needs to count
float(uda.mean())  # Needs the value
uda.values         # Converts to numpy

# These stay lazy:
uda.mean()         # Returns lazy array
uda.data           # Returns Dask array
```

2. **Chunking spatial dimension**: Avoid chunking the face dimension:

```python
# Avoid this - breaks spatial operations
uds = xu.open_dataset("model.nc", chunks={"mesh2d_nFaces": 1000})
```

3. **Too many tasks**: Many small chunks create overhead:

```python
# Too granular
uds = xu.open_dataset("model.nc", chunks={"time": 1})

# Better
uds = xu.open_dataset("model.nc", chunks={"time": 100})
```

4. **Not using a scheduler**: For large jobs, always use distributed:

```python
from dask.distributed import Client
client = Client()  # Use local cluster at minimum
```
