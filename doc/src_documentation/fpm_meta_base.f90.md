# Overview
The `fpm_meta_base` module provides the foundational derived type `metapackage_t` for all "meta-packages" within fpm. Meta-packages are fpm's mechanism for encapsulating pre-defined build configurations (compiler flags, linker flags, include directories, dependencies, etc.) for commonly used libraries or language features (e.g., OpenMP, MPI, HDF5). This base module defines the structure that all specific meta-package types will use and provides generic procedures to apply these configurations to different parts of the fpm build system.

# Key Components
- **`metapackage_t`**: A public derived type that serves as the base for specific meta-packages.
  - `name`: Allocatable character string, the name of the meta-package (e.g., "openmp", "stdlib").
  - `version`: Allocatable `version_t` (from `fpm_versioning`), for meta-packages that have a version.
  - **Logical Flags**: A series of logical flags (e.g., `has_link_libraries`, `has_build_flags`, `has_dependencies`, `has_run_command`) indicating which types of configurations this meta-package provides.
  - **Configuration Storage**:
    - `flags`, `fflags`, `cflags`, `cxxflags`, `link_flags`, `run_command`: `string_t` type for storing various command-line flags and runner commands.
    - `incl_dirs`, `link_libs`, `external_modules`: Allocatable arrays of `string_t` for include paths, library names, and external module names.
  - `fortran`: Allocatable `fortran_features_t` (from `fpm_model`) for specifying Fortran language features required/enabled by the meta-package.
  - `preprocess`: Allocatable `preprocess_config_t` (from `fpm_manifest_preprocess`) for preprocessor configurations.
  - `dependency`: Allocatable array of `dependency_config_t` (from `fpm_manifest_dependency`) to specify fpm package dependencies required by the meta-package.
  - **Procedures**:
    - `destroy()`: An elemental subroutine to reset/deallocate all components of a `metapackage_t` instance.
    - `resolve` (generic interface): A generic procedure that applies the meta-package's configurations. It has specific implementations for:
      - `resolve_cmd(self, settings, error)`: Applies configurations to `fpm_cmd_settings` (e.g., setting a default MPI runner in `fpm_run_settings`).
      - `resolve_model(self, model, error)`: Applies configurations to `fpm_model_t` (e.g., adding global compiler/linker flags, include directories, external modules).
      - `resolve_package_config(self, package, error)`: Applies configurations to `package_config_t` (e.g., adding dependencies, checking Fortran feature compatibility, merging preprocessor settings).

# Important Variables/Constants
N/A - This module primarily defines a data structure and its associated procedures.

# Usage Examples
This base module is not directly instantiated by users. Instead, specific meta-package modules (e.g., `fpm_meta_openmp.f90`, `fpm_meta_mpi.f90`) will typically:
1. Declare a type that extends or uses `metapackage_t`.
2. Provide an `init_<metapackage_name>` subroutine (e.g., `init_openmp`) that populates the fields of `metapackage_t` with the specific flags, libraries, etc., for that meta-package.
3. The `fpm_meta` module then calls this `init_` subroutine and subsequently uses the generic `resolve` procedures defined here to apply the settings.

**Conceptual example (how a specific meta-package might use it):**
```fortran
! In, for example, fpm_meta_openmp.f90
module fpm_meta_openmp
    use fpm_meta_base
    use fpm_compiler, only: compiler_t
    ! ... other imports
implicit none

contains
    subroutine init_openmp(this, compiler, all_meta, error)
        class(metapackage_t), intent(inout) :: this
        type(compiler_t), intent(in) :: compiler
        ! ... other args ...
        type(error_t), allocatable, intent(out) :: error

        call this%destroy() ! Initialize
        this%name = "openmp"
        this%has_fortran_flags = .true.
        this%fflags%s = compiler%get_feature_flag("openmp") ! Or a more complex detection
        ! ... set other flags or properties ...
    end subroutine init_openmp
end module fpm_meta_openmp
```
Then, `fpm_meta` would call `init_openmp` and then `meta%resolve(model, error)`, `meta%resolve(package_config, error)`, etc.

# Dependencies and Interactions
- **`fpm_error`**: For `error_t` and `fatal_error`.
- **`fpm_versioning`**: For `version_t`.
- **`fpm_model`**: For `fpm_model_t` and `fortran_features_t`.
- **`fpm_command_line`**: For `fpm_cmd_settings` and `fpm_run_settings`.
- **`fpm_manifest_dependency`**: For `dependency_config_t`.
- **`fpm_manifest_preprocess`**: For `preprocess_config_t`.
- **`fpm_manifest`**: For `package_config_t`.
- **`fpm_strings`**: For `string_t` and string utilities (`len_trim`, `split`, `join`).
- **`fpm_compiler`**: For `append_clean_flags` and `append_clean_flags_array` (used when merging flags).
- **`fpm_meta`**: This module is the primary consumer of `fpm_meta_base`. `fpm_meta` orchestrates the initialization (via specific `init_*` routines) and application (via the `resolve` methods) of meta-packages.
- **Specific `fpm_meta_*` modules**: These modules (e.g., `fpm_meta_openmp`, `fpm_meta_mpi`) are responsible for populating instances of `metapackage_t` with the correct configurations for their respective libraries/features.
The `metapackage_t` acts as a standardized container for configurations that can be uniformly applied to various parts of the fpm's build state.
