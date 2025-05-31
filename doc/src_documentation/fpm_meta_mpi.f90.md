# Overview
The `fpm_meta_mpi` module is a specialized meta-package handler within fpm designed to simplify the use of MPI (Message Passing Interface) in Fortran projects. It automatically attempts to detect an existing MPI installation and configure the fpm build with the necessary compiler and linker flags. The module searches for common MPI wrapper compilers (e.g., `mpif90`, `mpicc`, `mpiicc`), queries them to extract compilation and linking information, and tries to identify the underlying MPI implementation (OpenMPI, MPICH, Intel MPI, MS-MPI). It also handles Windows-specific MPI (MS-MPI) detection.

# Key Components
- **MPI Implementation Type Constants**:
  - `MPI_TYPE_NONE`, `MPI_TYPE_OPENMPI`, `MPI_TYPE_MPICH`, `MPI_TYPE_INTEL`, `MPI_TYPE_MSMPI`: Integer parameters to identify different MPI implementations.
- **`init_mpi(this, compiler, all_meta, error)`**: The main public subroutine.
  - It initializes a `metapackage_t` object (`this`) for MPI.
  - Calls `mpi_wrappers` to get a list of potential MPI wrapper compilers.
  - Calls `wrapper_compiler_fit` to find the best MPI wrapper that matches the current fpm compiler.
  - If a suitable wrapper is found, it calls `init_mpi_from_wrappers` to populate `this` with flags obtained by querying the wrapper.
  - If no wrapper fits, and on Windows, it attempts `msmpi_init` to detect a local MS-MPI installation.
  - If all detection methods fail, it returns a fatal error.
  - It adds "mpi" and "mpi_f08" to `this%external_modules`.
  - For non-Intel MPI, it may set `this%fortran%implicit_typing = .true.` and `this%fortran%implicit_external = .true.`.
- **`mpi_wrappers(compiler, fort_wrappers, c_wrappers, cpp_wrappers)`**: Collects a list of candidate MPI wrapper command names from environment variables (`MPICC`, `MPIFC`, etc.) and common defaults, including OS-specific and compiler-specific variants.
- **`assert_mpi_wrappers(wrappers, compiler, verbose)`**: Filters a list of wrapper names, keeping only those that appear to be valid MPI wrappers by calling `which_mpi_library`.
- **`which_mpi_library(wrapper, compiler, verbose)`**: Tries to determine the type of MPI library (OpenMPI, MPICH, Intel MPI) a given wrapper command belongs to by checking its response to specific flags (`--showme`, `-show`).
- **`wrapper_compiler_fit(fort_wrappers, c_wrappers, cpp_wrappers, compiler, wrap, mpi, error)`**: Tries to find the best Fortran, C, and C++ MPI wrappers from the provided lists that are compatible with the fpm `compiler`.
- **`mpi_compiler_match(language, wrappers, compiler, which_one, mpilib, error)`**: For a given language, finds a wrapper from the `wrappers` list that matches the fpm `compiler`.
- **`init_mpi_from_wrappers(this, compiler, mpilib, fort_wrapper, c_wrapper, cxx_wrapper, error)`**: Populates the `metapackage_t` by querying the chosen MPI wrappers for compile flags (`--showme:compile`, `-compile-info`), link flags (`--showme:link`, `-link-info`), version, and runner command.
- **`msmpi_init(this, compiler, error)`**: Handles detection of MS-MPI on Windows, checking environment variables (`MSMPI_LIB64/32`, `MSMPI_INC`, `MSMPI_BIN`) and MSYS2 paths.
- **`mpi_wrapper_query(mpilib, wrapper, command, verbose, error)`**: The core function for querying an MPI wrapper. Takes a `command` type (e.g., 'compiler', 'flags', 'link', 'version', 'runner') and executes the wrapper with appropriate arguments based on `mpilib` type to get the desired information.
- **`filter_build_arguments(compiler, command)` / `filter_link_arguments(compiler, command)`**: Subroutines to remove unnecessary or problematic flags (e.g., optimization flags, platform-specific flags that fpm handles separately) from the output of MPI wrappers.
- **Utility functions**: `get_mpi_runner`, `find_command_location`, `compiler_get_path`, `compiler_get_version`, `is_64bit_environment`, `MPI_TYPE_NAME`.

# Important Variables/Constants
- `MPI_TYPE_*` parameters: Used to differentiate MPI implementations and tailor queries.
- Hardcoded query flags for different MPI types within `mpi_wrapper_query` (e.g., `--showme:` for OpenMPI, `-compile-info`/`-link-info` for MPICH).

# Usage Examples
This module is used internally by `fpm_meta` when "mpi" is specified in `fpm.toml`.

**`fpm.toml` example:**
```toml
name = "my_parallel_simulation"
version = "0.1.0"

[build]
metapackages = ["mpi"]
```

**Conceptual fpm workflow:**
1. `fpm_meta` calls `init_mpi`.
2. `init_mpi` searches for MPI wrappers (e.g., `mpif90`).
3. It determines the MPI type (e.g., OpenMPI).
4. It queries `mpif90 --showme:compile` for compile flags and `mpif90 --showme:link` for link flags.
5. These flags are filtered (e.g., to remove `-O2`) and stored in the `metapackage_t` object.
6. `fpm_meta` applies these flags to the project's build model.
7. `init_mpi` also tries to find `mpiexec` or `mpirun` and sets it as `this%run_command`.

# Dependencies and Interactions
- **`fpm_compiler`**: For `compiler_t`, compiler ID constants, and flag generation/querying.
- **`fpm_filesystem`**: For path operations, file existence checks, running commands.
- **`fpm_os`**: For path canonicalization.
- **`fpm_error`**: For error handling.
- **`fpm_versioning`**: For `version_t` and parsing version strings.
- **`fpm_strings`**: For `string_t` and string manipulations.
- **`fpm_environment`**: For OS detection and environment variable access.
- **`fpm_meta_base`**: For `metapackage_t`.
- **`fpm_manifest_metapackages`**: For `metapackage_request_t`.
- **`fpm_pkg_config`**: `run_wrapper` from this module is used.
- **`shlex_module`**: For splitting command line strings.
- **External MPI Installation**: The module relies entirely on a pre-existing MPI installation on the system, providing wrapper compilers that are in the PATH or discoverable via environment variables.
The module's complexity arises from the variability in how different MPI implementations and their wrapper compilers expose build information.
