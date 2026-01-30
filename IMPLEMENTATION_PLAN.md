# Implementation Plan for Checkpointing

This document outlines the plan to implement a checkpointing and restart mechanism in the earthquake simulation code. The goal is to allow the simulation to be resilient to interruptions and to enable the reuse of expensive computations.

The implementation will be divided into two main features:

1.  **Fixing the restart feature:** The current restart feature is crashing when reading the `psi*.dat` file.
2.  **Green's Function Checkpointing:** This will allow the saving and loading of the Green's functions (H-matrix), which are computationally expensive to generate.

The changes will be made in a way that allows for separate commits for each logical change.

## 1. Fixing the restart feature (Completed)

### a. Analyze the reading and writing of `psi*.dat` (Completed)

**File:** `main_fv.f90`

**Purpose:** Identified that the issue was in `main_LH.f90`, not `main_fv.f90`.

### b. Correct the `psi` reading and writing (Completed)

**File:** `main_LH.f90`

**Purpose:** Removed the problematic read operation of `psi.dat` in the restart block, ensuring `psi` is recalculated. Corrected the file unit and status for writing `psi.dat` in both restart and no-restart blocks.

**Changes:**

*   Removed the explicit read operations for `psi` from `psi*.dat` in the restart block.
*   Ensured `psi` is recalculated using `psi=a*dlog(2*vref/vel*sinh(tau/sigma/a))` after `tau` and `sigma` are loaded during restart.
*   Modified the file opening for `psi.dat` in the restart block to `status='old', position='append'` and used `nout(5)`.
*   Modified the file opening for `psi.dat` in the "no restart" block to `status='replace'` and used `nout(5)`.

## 2. Green's Function Checkpointing (In Progress)

### a. Add Input Parameters for Green's Function Checkpointing (Completed)

**File:** `main_fv.f90`

**Purpose:** To control the saving and loading of the Green's functions from the input file.

**Changes:**

*   Added two new logical flags: `save_greens_functions` and `load_greens_functions`.
*   Added two new character variables: `greens_functions_file_s` and `greens_functions_file_n`.
*   Initialized `save_greens_functions` and `load_greens_functions` to `.false.` and the file names to empty strings.
*   Added cases in the parameter reading loop to read these new parameters.

### b. Implement Saving of Green's Functions (Completed)

**File:** `m_HACApK_base.f90`

**Purpose:** To create a subroutine to save the `st_leafmtxp` data to a file.

**Changes:**

*   Created a new subroutine `HACApK_save_leafmtxp` that serializes the complex `st_HACApK_leafmtxp` derived type, including its integer members, associated pointers (arrays of integers), and the array of `st_HACApK_leafmtx` derived types. It saves data unformatted, per-process.

**File:** `main_fv.f90`

**Purpose:** To call the new save subroutine after the Green's functions are generated.

**Changes:**

*   Added a conditional call to `HACApK_save_leafmtxp` for `st_leafmtxp_s` and `st_leafmtxp_n` after their generation, if `save_greens_functions` is true.

### c. Implement Loading of Green's Functions (Completed)

**File:** `m_HACApK_base.f90`

**Purpose:** To create a subroutine to load the `st_leafmtxp` data from a file.

**Changes:**

*   Created a new subroutine `HACApK_load_leafmtxp` that deserializes the `st_HACApK_leafmtxp` derived type, allocating memory for its components as needed and reading data unformatted, per-process.

**File:** `main_fv.f90`

**Purpose:** To call the new load subroutine instead of generating the Green's functions.

**Changes:**

*   Added a conditional call to `HACApK_load_leafmtxp` for `st_leafmtxp_s` and `st_leafmtxp_n` before their generation, if `load_greens_functions` is true, and skips the generation if successful.

## Commit Plan

1.  **fix: Correct the reading and writing of `psi` data during restart.**
    *   `main_LH.f90`: Removed explicit read of `psi.dat` in restart block. Corrected file unit and status for writing `psi.dat` in both restart and no-restart blocks.
2.  **feat: Add input parameters for Green's function checkpointing.**
    *   `main_fv.f90`: Added `save_greens_functions`, `load_greens_functions` logicals and `greens_functions_file_s`, `greens_functions_file_n` character variables. Initialized them and added reading from input file.
3.  **feat: Implement saving and loading of Green's functions.**
    *   `m_HACApK_base.f90`: Added `HACApK_save_leafmtxp` and `HACApK_load_leafmtxp` subroutines.
    *   `main_fv.f90`: Added conditional calls to `HACApK_save_leafmtxp` and `HACApK_load_leafmtxp`.
4.  **docs: Update documentation with the new checkpointing features.**
5.  **test: Add an example of how to use the checkpointing features.**
