# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Xugrid is an Xarray extension for working with 2D unstructured grids following UGRID conventions. It enables xarray-style operations (selection, plotting, I/O) on unstructured mesh data.

## Development Commands

```bash
# Run tests with coverage (disables Numba JIT for speed)
pixi run test

# Run specific test file
pixi run test -- tests/test_ugrid2d.py -v

# Run linting and formatting
pixi run pre-commit

# Build documentation
pixi run docs

# Run all checks (pre-commit, test, docs)
pixi run all

# Install git pre-commit hooks
pixi run install-pre-commit

# Test with different Python versions
pixi run --environment py310 test
pixi run --environment py313 test
```

## Architecture

### Core Design Pattern
Xugrid uses the **accessor pattern** to extend xarray objects with `.ugrid` functionality, plus **wrapper classes** that bundle xarray data with UGRID topology:

- `UgridDataArray` / `UgridDataset` - Wrap xarray objects with grid topology
- `.ugrid` accessor - Added to standard xarray DataArray/Dataset via `@register_*_accessor`

### Key Modules

**`xugrid/core/`** - Xarray integration:
- `wrap.py` - UgridDataArray/UgridDataset wrapper classes
- `dataarray_accessor.py`, `dataset_accessor.py` - The `.ugrid` accessor implementations
- `common.py` - I/O functions (open_dataset, open_zarr, concat, merge)

**`xugrid/ugrid/`** - Grid topology:
- `ugrid2d.py` - 2D unstructured grid (faces, nodes, edges)
- `ugrid1d.py` - 1D network topology
- `ugridbase.py` - Abstract base class
- `connectivity.py` - Adjacency/neighbor computations
- `burn.py`, `snapping.py`, `polygonize.py` - Geometry operations

**`xugrid/regrid/`** - Interpolation/regridding:
- `regridder.py` - Base classes and algorithms
- `structured.py` - Structured grid regridding
- `unstructured.py` - Unstructured-to-unstructured regridding

**`xugrid/plot/`** - Visualization for unstructured grids

**`xugrid/conversion.py`** - Shapely/GeoDataFrame conversions

### Performance Notes
- Heavy use of **Numba JIT** for geometry operations
- Set `NUMBA_DISABLE_JIT=1` during development/testing for faster iteration
- Uses **numba_celltree** for spatial indexing

## Code Style

- Formatter/linter: Ruff
- Docstrings: NumPy convention
- Pre-commit hooks handle formatting automatically

## Testing

- Framework: pytest with pytest-cases
- Tests are in `tests/` directory
- `conftest.py` sets matplotlib to 'Agg' backend (non-interactive)
- Regridding tests are in `tests/test_regrid/`
