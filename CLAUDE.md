# CLAUDE.md


<!-- GOVERNANCE-PREFLIGHT-v1 -->
## Governance Pre-Flight (summary — binding rules live in governance/)

All agents — Claude, Codex, Grok, Gemini, Hermes — before starting a task:
- Complete the startup audit and the **PHASE 0.5 pre-flight restatement**: restate to the
  user a 3–5 step plan plus the three most relevant governance policies, before doing the work.
- Use the **canonical document template** for any document — do not invent a format.

This is a summary; the binding rules and full checklists live in governance (source of truth):
- `~/repos/governance/policies/AGENT_INTERACTION_POLICY.md` — startup sequence + PHASE 0.5
- `~/repos/governance/standards/DOCUMENT_TEMPLATE_REGISTRY.md` — which template to use
- `~/repos/governance/INDEX.md` — master registry of all contracts, policies, gates
<!-- /GOVERNANCE-PREFLIGHT-v1 -->


This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Governance Prerequisite (Non-Negotiable)

**Before any work in this repository, read and comply with:** [`~/repos/governance/INDEX.md`](../governance/INDEX.md)

All cross-repo contracts, policies, and enforcement gates in `~/repos/governance/` are binding. Repo-specific rules below may extend but never override governance contracts.


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
cd ~/repos/LarmorD_New/tests && bash test.bash

# Example: protein prediction with accuracy weights
bin/larmord -csfile tests/measured_shifts_A003.dat \
    -parmfile data/larmorD_alphas_betas.dat \
    -reffile data/larmorD_reference_shifts.dat \
    -accfile data/larmorD_accuracy_weight.dat \
    -cutoff 15.0 tests/A003.pdb

# Example: RNA prediction (no cutoff)
bin/larmord -csfile tests/measured_shifts.dat \
    -parmfile data/larmorD_alphas_betas_rna.dat \
    -reffile data/larmorD_reference_shifts_rna.dat \
    -accfile data/larmorD_accuracy_weight_rna.dat \
    -cutoff 9999.0 tests/file.pdb
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
- `larmorD_accuracy_weight*.dat` - Accuracy weighting files for scoring
- `larmord_parameter_v*.dat` - Versioned parameter sets (v3.0–v8.0)

**File naming conventions:**
- `_rna` suffix → RNA (use with `-cutoff 9999.0`)
- `larmorD_proteins_` prefix → proteins with specific cutoffs
- `_cutoff_N` suffix → parameters optimized for N Å cutoff
- Default protein files (`larmorD_alphas_betas.dat`) → use with `-cutoff 15.0`

### Prediction Model

Chemical shifts are predicted using: δ = δ_ref + Σ(α_i / r_ij^β_i)

Where α and β parameters are atom-type specific and distances are to neighboring atoms within a cutoff radius.

## Key CLI Options

- `-csfile` - Experimental chemical shifts for comparison
- `-parmfile` - α/β parameters (alphas_betas file)
- `-reffile` - Reference (random coil) shifts
- `-accfile` - Accuracy weighting file (used in scoring; see `data/larmorD_accuracy_weight*.dat`)
- `-cutoff` - Distance cutoff in Ångströms (use 9999.0 for no cutoff)
- `-ring` - Enable ring current corrections
- `-cutoffRing` - Distance cutoff for ring current calculations
- `-trj` - Process trajectory file (DCD format)
- `-printError` - Output prediction errors (MAE, RMSE, etc.)
- `-mismatchCheck` - Verify PDB residues match chemical shift file

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
