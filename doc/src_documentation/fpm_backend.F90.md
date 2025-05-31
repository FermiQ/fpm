# Overview
This file defines the `fpm_backend` module, which is responsible for managing the build process of a Fortran package. It takes a list of build targets and a model of the package (defined by `fpm_model_t`) to schedule and execute compilation and linking tasks. The backend supports incremental compilation, meaning it avoids recompiling files if their dependencies haven't changed. It can also leverage OpenMP for parallel builds if available.

# Key Components
- **`build_package(targets, model, verbose, dry_run)`**: The main subroutine that orchestrates the entire build process. It sorts targets, schedules them, and then builds them.
  - `targets`: An array of `build_target_ptr` representing all items to be built.
  - `model`: An `fpm_model_t` instance containing package information.
  - `verbose`: A logical flag for verbose output.
  - `dry_run`: A logical flag to simulate the build without actual compilation/linking, but still generates compile commands.
- **`sort_target(target, mock)`**: A recursive subroutine that performs a topological sort of a given target based on its dependencies. It also checks cached source file digests to determine if a target is up-to-date and can be skipped.
  - `target`: The `build_target_t` to sort.
  - `mock`: Optional logical, if true, forces sorting of all targets (used in `dry_run`).
- **`schedule_targets(queue, schedule_ptr, targets)`**: Constructs a build schedule from the sorted targets. It groups targets into regions that can be built in parallel.
  - `queue`: Output allocatable array of `build_target_ptr` representing the build order.
  - `schedule_ptr`: Output allocatable integer array defining the boundaries of parallel build regions in the `queue`.
  - `targets`: Input array of `build_target_ptr` that have been sorted.
- **`build_target(model, target, verbose, dry_run, table, stat)`**: Subroutine to compile or link a single build target. It calls appropriate compiler/linker commands based on the target type. If successful and the target is source-based, it caches the source file digest.
  - `model`: The `fpm_model_t` instance.
  - `target`: The `build_target_t` to build.
  - `verbose`: Logical for verbose output.
  - `dry_run`: Logical to simulate build.
  - `table`: An `compile_command_table_t` to store compile commands.
  - `stat`: Output integer status of the build command.
- **`print_build_log(target)`**: Subroutine to read and print the build log file for a given target if an error occurs.
- **`c_isatty()`**: An interface to a C function that checks if the output is a TTY. This is used to determine if rich progress output or plain output should be used. (Not available in FPM_BOOTSTRAP mode).

# Important Variables/Constants
- **`FPM_TARGET_OBJECT`, `FPM_TARGET_C_OBJECT`, `FPM_TARGET_CPP_OBJECT`**: Constants representing Fortran, C, and C++ object file target types, respectively.
- **`FPM_TARGET_ARCHIVE`**: Constant representing a static library (archive) target type.
- **`FPM_TARGET_EXECUTABLE`**: Constant representing an executable target type.
- **`FPM_TARGET_SHARED`**: Constant representing a shared library target type.
- `target%digest_cached`: Stores the cached hash (digest) of a source file, used for incremental compilation.
- `target%skip`: A logical flag indicating if a target can be skipped because it's up-to-date.
- `target%sorted`: A logical flag indicating if a target has been processed by the `sort_target` subroutine.
- `target%schedule`: An integer indicating the scheduling region for a target.
- `compile_command_table_t`: A derived type to hold compile commands, likely for generating `compile_commands.json`.

# Usage Examples
This module is used internally by the `fpm` executable when a build is triggered (e.g., `fpm build`). It's not typically used directly by a package developer, but its behavior is configured through the `fpm.toml` manifest file and command-line options.

Conceptual workflow:
1. `fpm` parses `fpm.toml` and command-line arguments to create an `fpm_model_t`.
2. It identifies all build targets (source files, executables, libraries).
3. It calls `build_package` with these targets and the model.
4. `build_package` then:
    a. Creates necessary output directories.
    b. Calls `sort_target` for each target to determine dependencies and if they need recompilation.
    c. Calls `schedule_targets` to create an optimized build queue.
    d. Iterates through the queue, calling `build_target` for each, potentially in parallel (if OpenMP is enabled).
    e. `build_target` invokes the appropriate compiler/linker (Fortran, C, C++, archiver) based on `target%target_type`.
    f. After a successful build of a source-based target, its source digest is cached (e.g., in `target%output_file//'.digest'`).

# Dependencies and Interactions
- **`iso_fortran_env`**: Standard Fortran module for input/output units.
- **`fpm_error`**: For error handling (`fpm_stop`, `error_t`).
- **`fpm_filesystem`**: For filesystem operations like `basename`, `dirname`, `join_path`, `exists`, `mkdir`, `run`, `getline`.
- **`fpm_model`**: Defines `fpm_model_t` (the package model) and related types.
- **`fpm_strings`**: For string manipulation (`string_t`, `.in.` operator).
- **`fpm_targets`**: Defines `build_target_t`, target type constants, etc.
- **`fpm_backend_output`**: Handles the visual output of the build process (e.g., progress bars).
- **`fpm_compile_commands`**: For managing and storing compile commands, likely for `compile_commands.json`.
- **C Standard Library (via `c_isatty`)**: Interacts with a C function to check TTY status for output formatting.
- **Compiler/Linker**: This module orchestrates calls to external compiler and linker tools (configured in `fpm_model_t`) to perform the actual compilation and linking.
- **OpenMP**: Can utilize OpenMP directives (`!$omp`) for parallel execution of build tasks if fpm is compiled with OpenMP support.
- It relies on source file digests (hashes) cached on disk (e.g., `*.digest` files) for incremental compilation.
