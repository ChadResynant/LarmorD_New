# LarmorD Refactoring Plan

## Overview
This document outlines larger refactoring tasks identified during code analysis.

## Priority 1: Code Quality

### 1.1 Replace exit() calls with error handling
**Files affected:** `lib/LARMORD.cpp`, `lib/PDB.cpp`, `lib/Mol2.cpp`, `lib/Select.cpp`, `lib/Trajectory.cpp`, `src/larmord.cpp`, `src/larmord_extractor.cpp`

**Current behavior:** Library code calls `exit(0)` or `exit(1)` on errors, terminating the entire program.

**Proposed solution:**
- Add error return codes or exceptions to library functions
- Have callers handle errors appropriately
- Example pattern:
```cpp
// Before
if (!file.is_open()) {
    std::cerr << "Error: Unable to open file" << std::endl;
    exit(0);
}

// After
if (!file.is_open()) {
    std::cerr << "Error: Unable to open file" << std::endl;
    return false;  // or throw std::runtime_error("...")
}
```

**Locations (11 total):**
- `lib/LARMORD.cpp:167, 234, 260, 293, 319`
- `lib/PDB.cpp:268`
- `lib/Mol2.cpp:157`
- `lib/Select.cpp:50`
- `lib/Trajectory.cpp:128`
- `src/larmord.cpp:51`
- `src/larmord_extractor.cpp:45`

### 1.2 Extract shared CLI parsing code
**Files affected:** `src/larmord.cpp`, `src/larmord_extractor.cpp`

**Current behavior:** ~70% code duplication between the two executables.

**Proposed solution:**
- Create `src/cli_common.hpp` with shared argument parsing
- Extract common trajectory loop into shared function
- Keep only mode-specific logic in each executable

## Priority 2: Memory Safety

### 2.1 Replace VLAs with std::vector
**File:** `lib/Trajectory.cpp`

**Current behavior:** Uses C99 variable-length arrays (VLAs), which are a Clang extension in C++.

**Locations:**
- Line 552: `double pbc[ftrjin->pb.size()]`
- Lines 562-564, 585-587: `float x/y/z[...]`

**Proposed solution:**
```cpp
// Before
double pbc[ftrjin->pb.size()];

// After
std::vector<double> pbc(ftrjin->pb.size());
```

### 2.2 Review memory cleanup
**Files:** `src/larmord.cpp`, `src/larmord_extractor.cpp`

Some code paths may not properly delete allocated `Molecule*` objects. Audit all `new Molecule` calls and ensure proper cleanup.

**Long-term:** Consider smart pointers (`std::unique_ptr<Molecule>`) for automatic cleanup.

## Priority 3: Modernization

### 3.1 Update PyMOL plugin for Python 3
**File:** `pymol/LARMORDPlugin.py`

**Changes needed:**
- `print` statements -> `print()` function
- `tkSimpleDialog` -> `tkinter.simpledialog`
- `tkMessageBox` -> `tkinter.messagebox`
- `tkFileDialog` -> `tkinter.filedialog`
- `tkColorChooser` -> `tkinter.colorchooser`
- `Tkinter` -> `tkinter`

### 3.2 C++ modernization (optional)
- Use `nullptr` instead of `NULL`
- Use range-based for loops where applicable
- Use `auto` for iterator declarations
- Consider `std::string_view` for read-only string parameters

## Implementation Order

1. **Quick wins (1-2 hours):**
   - VLA replacement in Trajectory.cpp

2. **Medium effort (4-8 hours):**
   - PyMOL plugin Python 3 update
   - Memory cleanup audit

3. **Larger refactoring (1-2 days):**
   - exit() replacement with proper error handling
   - CLI code extraction

## Testing Strategy

After each change:
1. `make clean && make` - verify compilation
2. `bash tests/test.bash` - verify basic functionality
3. Test with RNA data: `bin/larmord -parmfile data/larmorD_alphas_betas_rna.dat -reffile data/larmorD_reference_shifts_rna.dat tests/file.pdb`
4. Test with protein data: Use A003.pdb test case
