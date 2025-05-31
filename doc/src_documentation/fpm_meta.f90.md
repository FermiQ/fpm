# Overview
The `fpm_meta` module serves as a high-level coordinator for "meta-packages" within fpm. Meta-packages are fpm's way of providing pre-defined sets of compiler flags, linker libraries, and potentially other build configurations needed to easily use common external libraries or features (like OpenMP, MPI, HDF5, NetCDF, BLAS, and the Fortran-lang stdlib or Minpack).

This module itself doesn't contain the specific logic for each meta-package. Instead, it defines a framework to initialize and apply these meta-packages. It relies on specialized modules (e.g., `fpm_meta_openmp`, `fpm_meta_mpi`) to provide the actual implementation details for each supported library/feature. The primary goal is to simplify the process for users to enable these common dependencies in their `fpm.toml` manifest.

# Key Components
- **`resolve_metapackages` (interface to `resolve_metapackage_model`)**: The main public entry point of the module. This subroutine takes the overall fpm model, the current package configuration, and build settings, and applies the configurations from any requested meta-packages.
- **`init_from_request(this, request, compiler, all_meta, error)`**: A private subroutine that acts as a factory or dispatcher. Based on the name in a `metapackage_request_t`, it calls the appropriate initialization routine from a specific `fpm_meta_*` module (e.g., `init_openmp` for "openmp", `init_stdlib" for "stdlib").
  - `this`: The `metapackage_t` object to be initialized.
  - `request`: A `metapackage_request_t` containing the name and any options for the requested meta-package.
  - `compiler`: The `compiler_t` object, needed by individual meta-package initializers to determine compiler-specific flags.
  - `all_meta`: An array of all requested meta-packages, allowing for dependency resolution between them if needed.
  - `error`: For error reporting.
- **`add_metapackage_model(model, package, settings, meta, error)`**: A private subroutine that applies the configurations from an initialized `metapackage_t` object to the fpm model, the package configuration, and the build settings.
  - `model`: The global `fpm_model_t`.
  - `package`: The current `package_config_t`.
  - `settings`: The current command-line settings (e.g., `fpm_build_settings`).
  - `meta`: The initialized `metapackage_t` object to apply.
  - `error`: For error reporting.
- **`resolve_metapackage_model(model, package, settings, error)`**: The core implementation subroutine. It iterates through all meta-packages requested in the `package_config_t`, calls `init_from_request` for each, and then `add_metapackage_model` to apply its effects.

# Important Variables/Constants
N/A - This module primarily orchestrates calls to other modules. The important "constants" are the string names of the supported meta-packages (e.g., "openmp", "stdlib") used in the `select case` statement within `init_from_request`.

# Usage Examples
This module is used internally by fpm when it processes a package's `fpm.toml` manifest. If a user specifies meta-packages in their manifest, fpm will invoke `resolve_metapackages` to incorporate the necessary build settings.

**Example `fpm.toml` snippet:**
```toml
name = "my_scientific_app"
version = "0.1.0"

[build]
# Request OpenMP and MPI meta-packages
metapackages = ["openmp", "mpi"]
```

**Conceptual fpm workflow:**
1. fpm parses `fpm.toml` and identifies `metapackages = ["openmp", "mpi"]`.
2. During model setup, fpm calls `resolve_metapackages(current_model, current_package_config, current_build_settings, error_obj)`.
3. `resolve_metapackage_model` iterates:
    a. For "openmp":
        i. `init_from_request` calls `init_openmp` (from `fpm_meta_openmp`).
        ii. `init_openmp` populates a `metapackage_t` with OpenMP-specific flags (e.g., `-fopenmp`).
        iii. `add_metapackage_model` applies these flags to `current_model%compiler%flags`, `current_package_config%build%flags`, etc.
    b. For "mpi":
        i. `init_from_request` calls `init_mpi` (from `fpm_meta_mpi`).
        ii. `init_mpi` might try to find `mpif90` and extract its flags, populating another `metapackage_t`. It might also set `has_run_command = .true.` if an MPI runner like `mpiexec` is found.
        iii. `add_metapackage_model` applies these MPI flags and potentially updates run settings if it's an `fpm_run_settings` instance.
4. The build then proceeds with these added compiler/linker flags. If running an MPI application, fpm might use the detected MPI runner.

# Dependencies and Interactions
- **`fpm_compiler`**: For `compiler_t`, used to pass compiler information to individual meta-package initializers.
- **`fpm_manifest`**: For `package_config_t`, which contains the list of requested meta-packages.
- **`fpm_model`**: For `fpm_model_t`, to which meta-package configurations (like global flags) are applied.
- **`fpm_command_line`**: For `fpm_cmd_settings`, `fpm_build_settings`, `fpm_run_settings`, to which meta-package configurations can also be applied (e.g., MPI runner command).
- **`fpm_error`**: For error reporting types (`error_t`) and routines (`syntax_error`, `fatal_error`).
- **`fpm_meta_base`**: For the base `metapackage_t` type and its `destroy` procedure. This is the type that all specific meta-package initializers populate.
- **Specific `fpm_meta_*` modules**:
  - `fpm_meta_openmp` (for `init_openmp`)
  - `fpm_meta_stdlib` (for `init_stdlib`)
  - `fpm_meta_minpack` (for `init_minpack`)
  - `fpm_meta_mpi` (for `init_mpi`)
  - `fpm_meta_hdf5` (for `init_hdf5`)
  - `fpm_meta_netcdf` (for `init_netcdf`)
  - `fpm_meta_blas` (for `init_blas`)
  These modules contain the actual logic for determining flags and settings for each respective library/feature.
- **`fpm_manifest_metapackages`**: For `metapackage_request_t`, which is how requested meta-packages are represented internally before resolution.
- **`shlex_module`**, **`regex_module`**: Imported but not directly used in the provided code snippet of `fpm_meta.f90`. They might be used by the individual `fpm_meta_*` modules.
- **`iso_fortran_env`**: For `stdout`.
