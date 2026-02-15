# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

libnest2d is a C++14 header-only library for 2D bin packing (nesting polygonal shapes into bins with minimal waste). Originally developed for PrusaSlicer's arrangement feature. It uses a template-based, pluggable architecture with swappable geometry backends, optimizers, and threading models.

## Build Commands

```bash
# Configure (out-of-source build, auto-download dependencies)
cmake -B build -DCMAKE_BUILD_TYPE=Release -DLIBNEST2D_HEADER_ONLY=OFF -DLIBNEST2D_BUILD_UNITTESTS=ON -DRP_ENABLE_DOWNLOADING=ON

# Build
cmake --build build --config Release

# Run tests
cd build && ctest --verbose

# Build and run example (after installing)
cmake -B build-example examples -DCMAKE_PREFIX_PATH="build/dist;build/dependencies"
cmake --build build-example
```

## Key CMake Options

- `LIBNEST2D_HEADER_ONLY` (default ON): Header-only mode; set OFF to build static/shared lib
- `LIBNEST2D_GEOMETRIES`: Geometry backend — `clipper` (default), `boost`, `eigen`
- `LIBNEST2D_OPTIMIZER`: Optimizer backend — `nlopt` (default), `optimlib`
- `LIBNEST2D_THREADING`: Threading — `std` (default), `tbb`, `omp`, `none`
- `LIBNEST2D_BUILD_UNITTESTS` (default OFF): Build unit tests (uses Catch2)
- `RP_ENABLE_DOWNLOADING`: Auto-download missing dependencies

## Architecture

The library is organized in three layers controlled by template parameters:

**Public API** — `nest()` free functions in `include/libnest2d/libnest2d.hpp`. This is the main entry point. It instantiates `_Nester<Placer, Selector>` from `nester.hpp`.

**Algorithms** (all in `include/libnest2d/`):
- **Placers** (`placers/`): Determine where to position each item in a bin
  - `nfpplacer.hpp` — No-Fit Polygon placer (recommended, supports arbitrary bin shapes)
  - `bottomleftplacer.hpp` — Simple bottom-left heuristic (rectangular bins only)
- **Selectors** (`selections/`): Determine bin assignment order
  - `firstfit.hpp` — Place each item in first available bin
  - `filler.hpp` — Fill bins completely before moving on
  - `djd_heuristic.hpp` — Distance-based heuristic

**Backend layer** — Pluggable via template specialization (no runtime polymorphism):
- **Geometry** (`backends/clipper/`): Clipper library for polygon boolean ops. Type traits in `geometry_traits.hpp` allow any geometry type to be adapted.
- **Optimizers** (`optimizers/nlopt/`): NLopt-based local/global optimization (simplex, subplex, genetic). Alternative: OptimLib particle swarm.
- **Threading** (`parallel.hpp`): Abstraction over std::thread, TBB, or OpenMP.

## Key Design Patterns

- **Traits-based type system**: `geometry_traits.hpp` defines `TPoint<Shape>`, `TCoord<Shape>`, `TCompute<Type>` etc. via template specialization. Any external geometry type can be integrated by specializing these traits — no inheritance required.
- **CRTP boilerplates**: `placer_boilerplate.hpp` and `selection_boilerplate.hpp` provide base implementations via Curiously Recurring Template Pattern.
- **Caching in `_Item`**: Items (defined in `nester.hpp`) cache area, bounding box, and transformed shapes. Cache invalidates on translation/rotation/inflation changes.
- **Compile-time configuration**: Preprocessor defines (`LIBNEST2D_GEOMETRIES_clipper`, `LIBNEST2D_OPTIMIZER_nlopt`, etc.) select backends. Template specialization eliminates unused code paths.

## Core Data Types (all in `nester.hpp` / `geometry_traits.hpp`)

- `_Item<RawShape>` — A polygon to be packed, with translation, rotation, inflation, priority, and bin assignment
- `_Box<P>`, `_Circle<P>`, `_Segment<P>` — Bin and geometric primitives
- `NfpPConfig` — Configuration for the NFP placer (rotations, alignment, accuracy, custom objective function)

## Testing

Tests use Catch2 framework. Test executable is `tests_clipper_nlopt`, built from `tests/test.cpp`. Test data consists of real 3D printer part outlines in `tools/printer_parts.hpp/cpp`.

## Dependencies

Managed via `RequirePackage()` (in `cmake_modules/RequirePackage.cmake`). External package recipes live in `external/`. With `RP_ENABLE_DOWNLOADING=ON`, missing deps are fetched automatically into `${PROJECT_BINARY_DIR}/dependencies`.

Core: Clipper, NLopt, Boost (geometry subset). Testing: Catch2. Optional: TBB, GTest.
