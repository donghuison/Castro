# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Castro is an adaptive mesh, astrophysical radiation/MHD hydrodynamics simulation code designed for massively parallel computing. It's built on the AMReX framework and uses Microphysics for equations of state and reaction networks.

## Key Dependencies

- **AMReX**: Adaptive mesh refinement library (in `external/amrex/`)
- **Microphysics**: Equations of state, reaction networks (in `external/Microphysics/`)
- **Compiler Requirements**: C++17 or later (GCC >= 9.0 for CUDA)
- **Python**: >= 3.10
- **GNU Make**: >= 3.82

## Common Development Commands

### Building Castro

Navigate to a specific problem directory in `Exec/` and configure the `GNUmakefile`:

```bash
# Example: Building the Sedov test problem
cd Exec/hydro_tests/Sedov

# Key GNUmakefile options to modify:
# DIM = 1, 2, or 3          (problem dimensionality)
# COMP = gnu                (compiler: gnu, intel, etc.)
# USE_MPI = TRUE/FALSE      (enable MPI parallelization)
# USE_OMP = TRUE/FALSE      (enable OpenMP)
# DEBUG = TRUE/FALSE        (enable debug mode)
# USE_CUDA = TRUE/FALSE     (enable NVIDIA GPU support)
# USE_HIP = TRUE/FALSE      (enable AMD GPU support)
# USE_MHD = TRUE/FALSE      (enable MHD)

# Build the executable
make -j 4

# Clean build
make clean
make realclean  # removes all temporary files
```

### Running Tests

```bash
# Run a test problem (example with Sedov in 3D)
./Castro3d.gnu.ex inputs.3d.sph max_step=10

# Common runtime parameters (specified on command line or in inputs file):
# max_step = N              (maximum number of timesteps)
# stop_time = T             (stop at time T)
# amr.max_level = N         (maximum AMR level)
# amr.plot_files_output = 0/1
# amr.checkpoint_files_output = 0/1
```

### Running Single Test Problems

```bash
# Extract 1D slices from plotfiles (for verification)
# First build fextract tool in amrex/Tools/Postprocessing/C_Src/
fextract3d.exe -d 1 -s output.out -p plt00034

# Compare with analytic solutions (in Verification/ directory)
```

### Testing Workflow

The CI/CD uses GitHub Actions for automated testing:
- Tests are triggered on pull requests
- Key test problems: Sedov, Sod, acoustic pulse, bubble convergence
- Comparison against stored benchmarks in `ci-benchmarks/`

## High-Level Architecture

### Source Directory Structure

```
Source/
├── driver/           # Main Castro class and time integration
│   ├── Castro.H      # Main Castro class definition
│   ├── Castro.cpp    # Core functionality
│   └── Castro_advance*.cpp  # Time advancement methods
├── hydro/            # Hydrodynamics solvers
│   ├── Castro_ctu.cpp    # Corner Transport Upwind
│   ├── Castro_mol.cpp    # Method of Lines
│   └── riemann_*.cpp     # Riemann solvers
├── mhd/              # MHD (constrained transport)
├── gravity/          # Self-gravity (Poisson solver)
├── radiation/        # Multigroup radiation diffusion
├── reactions/        # Nuclear reaction networks
├── diffusion/        # Thermal diffusion
├── rotation/         # Rotating reference frame
├── sdc/              # Spectral Deferred Corrections
├── sources/          # Source terms
└── problems/         # Problem-specific routines
```

### Problem Directory Structure

Each problem in `Exec/` contains:
- `GNUmakefile`: Build configuration
- `inputs*`: Runtime parameter files
- `_prob_params`: Problem-specific runtime parameters
- `problem_initialize.H`: Initial conditions
- `problem_bc_fill.H`: Custom boundary conditions (optional)

### Key Design Patterns

1. **StateType Arrays**: Castro uses different state arrays for different physics:
   - `State_Type`: Main conserved variables (density, momentum, energy, species)
   - `Rad_Type`: Radiation energy groups
   - `PhiGrav_Type`: Gravitational potential
   - `Mag_Type_*`: Magnetic field components

2. **Time Integration Methods**:
   - CTU (Corner Transport Upwind): Default 2nd-order unsplit method
   - SDC (Spectral Deferred Corrections): For stiff reaction coupling
   - MOL (Method of Lines): Alternative integration scheme

3. **AMR Structure**: Built on AMReX's adaptive mesh refinement with:
   - Subcycling between levels
   - 2x or 4x refinement ratios
   - Flux correction at coarse-fine interfaces

4. **Parallelization**:
   - MPI for distributed memory
   - OpenMP for shared memory (CPU)
   - CUDA/HIP for GPU acceleration
   - Tiling strategy for cache optimization

### Key Physics Modules

- **Hydrodynamics**: 2nd-order (4th-order for uniform grids) finite-volume methods
- **MHD**: Constrained transport for divergence-free magnetic fields
- **Gravity**: Full Poisson solver with isolated boundary conditions
- **Radiation**: Multigroup flux-limited diffusion
- **Reactions**: Strang or SDC coupling to nuclear networks
- **Equation of State**: General EOS interface via Microphysics

### Common Development Patterns

1. **Adding a new problem**: Copy an existing problem directory from `Exec/`, modify initial conditions in `problem_initialize*.H`

2. **Custom boundary conditions**: Implement in `problem_bc_fill.H`

3. **Problem-specific parameters**: Define in `_prob_params` file

4. **Runtime parameters**: Set in `inputs` files or command line

5. **Diagnostics**: Use `problem_diagnostics.H` for custom analysis

6. **Tagging for refinement**: Implement in `problem_tagging.H`

## Important Build Flags

- `BL_NO_FORT = TRUE`: Castro is now fully C++ (no Fortran)
- `USE_TRUE_SDC`: Enable true SDC (4th order)
- `USE_SIMPLIFIED_SDC`: Enable simplified SDC (2nd order)
- `USE_GRAV = TRUE`: Enable self-gravity
- `USE_REACT = TRUE`: Enable reactions

## Debugging Tips

- Use `DEBUG = TRUE` in GNUmakefile for debug builds with bounds checking
- Set `castro.verbose = 1` for detailed runtime output
- Use `amr.plot_files_output = 1` to generate plotfiles for visualization
- Check `grid_diag.out` for conservation diagnostics

## Visualization

- **yt**: Primary tool, native AMReX support (`yt.load()` on plotfiles)
- **Amrvis**: Quick 2D/3D visualization of AMReX plotfiles
- **VisIt**: Alternative with AMReX support

## Testing Standards

- Regression tests compare against analytic solutions when available
- Convergence tests verify expected order of accuracy
- Conservation tests check mass, momentum, energy conservation
- CI runs key test problems on every PR