# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

LarmorD is a distance-based NMR chemical shift predictor for RNA and proteins. It predicts 1H, 13C, and 15N chemical shifts from 3D molecular structures (PDB format) or trajectories (DCD format).

## Build Commands

```bash
# Build from source
make clean && make

# This builds:
#   lib/libmoletools.a - core molecular analysis library
#   bin/larmord - main chemical shift predictor
#   bin/larmord_extractor - feature extraction tool
```

## Running Tests

```bash
# Run test suite from repository root
cd tests && bash test.bash

# Example prediction on a single PDB
bin/larmord -csfile tests/measured_shifts.dat \
    -parmfile data/larmorD_alphas_betas_rna.dat \
    -reffile data/larmorD_reference_shifts_rna.dat \
    tests/file.pdb
```

## Architecture

### Core Components

- **lib/** - `libmoletools` static library with molecular handling classes:
  - `Molecule` - PDB/Mol2 parsing, atom selection, coordinate manipulation
  - `LARMORD` - Chemical shift prediction engine (ring current effects, distance-based parameters)
  - `Trajectory` - DCD trajectory file reading
  - `Atom`, `Residue`, `Chain` - Molecular hierarchy

- **src/** - Main executables:
  - `larmord.cpp` - Full chemical shift predictor with extensive CLI options
  - `larmord_extractor.cpp` - Extracts features for machine learning

### Data Files

All parameter files are in `data/`:
- `larmorD_alphas_betas_*.dat` - Distance-dependent parameters (α, β coefficients)
- `larmorD_reference_shifts_*.dat` - Random coil reference shifts
- `larmorD_accuracy_*.dat` - Prediction accuracy weights

Use `_rna` suffix files for RNA, `_proteins_` prefix for proteins.

### Prediction Model

Chemical shifts are predicted using: δ = δ_ref + Σ(α_i / r_ij^β_i)

Where α and β parameters are atom-type specific and distances are to neighboring atoms within a cutoff radius.

## Key CLI Options

- `-csfile` - Experimental chemical shifts for comparison
- `-parmfile` - α/β parameters
- `-reffile` - Reference (random coil) shifts
- `-cutoff` - Distance cutoff in Ångströms
- `-ring` - Enable ring current corrections
- `-trj` - Process trajectory file (DCD format)
- `-printError` - Output prediction errors (MAE, RMSE, etc.)

## Output Format

```
processor frame resid resname nucleus predicted measured random_coil id
```

## PyMOL Integration

`pymol/LARMORDPlugin.py` provides a PyMOL GUI for running predictions and visualizing chemical shift errors as spheres. Requires `LARMORD_BIN` environment variable pointing to bin directory.

## rna_predictor_new/

Contains RNA structure datasets organized by PDB ID (e.g., 1SCL/, 2LX1/) with:
- PDB structures
- Measured chemical shift files
- Feature extraction scripts

## Code Patterns and Conventions

### C++ Style
- **Memory Management**: Manual `new`/`delete` pattern throughout; no smart pointers
- **Error Handling**: Uses `exit(0)` or `exit(1)` for fatal errors - no exceptions
- **String Operations**: Heavy use of `std::map<std::string, ...>` for key-value lookups
- **File I/O**: Uses C++ streams with manual error checking via `is_open()`

### Key Design Patterns
- **Factory Pattern**: `Molecule::readPDB()` static factory for PDB parsing
- **Copy Pattern**: `Molecule::copy()` creates deep copies with selection filtering
- **Distance Matrix**: `Analyze::pairwiseDistance()` computes O(N²) distance matrices

### Parameter File Formats
| File Type | Columns | Format |
|-----------|---------|--------|
| Reference shifts | 3 | `resname nucleus shift` |
| Alpha/Beta params | 6 | `resname nucleus neighbor_resname neighbor_atom alpha beta` |
| Accuracy weights | 3 | `nucleus resname weight` |
| Chemical shifts | 5 | `resname resid nucleus shift error` |

### Residue Name Conventions
- RNA: `GUA`, `ADE`, `CYT`, `URA` (3-letter codes)
- Protein: Standard 3-letter codes (ALA, GLY, etc.)
- The code auto-renames some non-standard residues via `LARMORD::renameRes()`

## Code Hotspots (Most Complexity)

| File | Lines | Notes |
|------|-------|-------|
| `lib/LARMORD.cpp` | 2553 | Core prediction engine, ring current calculations |
| `lib/Trajectory.cpp` | 874 | DCD binary format parsing |
| `lib/Molecule.cpp` | 837 | PDB parsing, atom selection, coordinate manipulation |
| `src/larmord.cpp` | 725 | Main predictor with extensive CLI parsing |

## Technical Debt

### Known Issues
1. **Exit calls in library code**: `lib/LARMORD.cpp`, `lib/PDB.cpp`, etc. use `exit()` instead of returning errors
2. **Code duplication**: Significant overlap between `larmord.cpp` and `larmord_extractor.cpp`
3. **Memory leaks**: Some code paths don't clean up allocated `Molecule*` objects

### PyMOL Plugin
- Uses Python 2 syntax (`print` statements, `tkSimpleDialog`)
- Requires updating for Python 3 / modern PyMOL

## Debugging Tips

### Common Issues
1. **Missing parameter file**: Check that `-parmfile`, `-reffile` paths are correct
2. **Residue mismatch**: Use `-mismatchCheck` to verify PDB matches CS file
3. **Zero predictions**: Usually means random coil shift not found for residue:nucleus combo

### Debugging Workflow
```bash
# Verbose output during prediction
# (uncomment std::cout lines in LARMORD.cpp loadParmFile/loadRefFile)

# Check distance cutoff effects
bin/larmord -cutoff 5.0 ...  # vs default 99999.9
```

## Authors and History
- Primary authors: Aaron T. Frank, Sean M. Law (University of Michigan)
- Copyright: University of Michigan, licensed software
- Contributors: Blair Whittington
