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
