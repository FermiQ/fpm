# Overview
The `fpm_meta_hdf5` module is a specialized meta-package handler within fpm designed to facilitate the use of the HDF5 (Hierarchical Data Format 5) library. Its main function is to automatically determine and apply the necessary compiler and linker flags required to build a Fortran project that depends on HDF5. This module primarily leverages the `pkg-config` utility to discover HDF5 Cflags and Libs. It also has specific logic to search for and include HDF5 Fortran high-level (HL) interface libraries, as these are sometimes not explicitly listed in the main HDF5 `pkg-config` file provided by some distributions.

# Key Components
- **`init_hdf5(this, compiler, all_meta, error)`**: The core public subroutine that initializes the HDF5 meta-package configuration.
  - It takes an uninitialized `metapackage_t` object (`this`) and populates it.
  - Asserts that `pkg-config` is available on the system; if not, it returns a fatal error.
  - **`pkg-config` Lookup**: It searches for an HDF5 package using `pkg-config` by trying a list of predefined candidate names (e.g., `hdf5_hl_fortran`, `hdf5-serial`, `hdf5`).
  - **Versioned `pkg-config` file search**: If no standard candidate is found, it lists all `pkg-config` modules and looks for one starting with "hdf5" (to catch names like `hdf5-1.2.3.pc`).
  - If a suitable HDF5 `pkg-config` entry is found, it calls `add_pkg_config_compile_options` (from `fpm_meta_util`) to populate the `metapackage_t` with flags.
  - **High-Level (HL) Library Search**: It then attempts to find and add HDF5 Fortran high-level (HL) libraries. This involves:
    - Iterating through the libraries already found (e.g., `libhdf5`).
    - For each base HDF5 library, it constructs names for potential HL Fortran libraries by appending suffixes like `_hl_fortran`, `hl_fortran`, `_fortran`, `_hl`.
    - It checks if these constructed library files exist in the library directory reported by `pkg-config`.
    - If an HL library exists and is not already in the link list, it's added to `this%link_libs`.
  - **External Modules**: It explicitly lists common HDF5 Fortran module names (e.g., `h5a`, `h5d`, `hdf5`) and adds them to `this%external_modules`. This tells fpm that these modules are provided by an external library and should not be sought within the project's own source files.
  - If no HDF5 package can be found via `pkg-config`, it returns a fatal error.

# Important Variables/Constants
- **`find_hl(*)`**: A private parameter array of character strings: `['_hl_fortran', 'hl_fortran', '_fortran', '_hl']`. These are suffixes used to probe for HDF5 high-level Fortran libraries.
- **`candidates(*)`**: A private parameter array of character strings: `['hdf5_hl_fortran', 'hdf5-hl-fortran', 'hdf5_fortran', 'hdf5-fortran', 'hdf5_hl', 'hdf5', 'hdf5-serial']`. This lists the `pkg-config` package names that `init_hdf5` will search for, in order of preference.

# Usage Examples
This module is used internally by `fpm_meta` when "hdf5" is specified in the `metapackages` array in the `fpm.toml` manifest.

**`fpm.toml` example:**
```toml
name = "my_data_analysis_project"
version = "1.0.0"

[build]
metapackages = ["hdf5"]
```

**Conceptual fpm workflow:**
1. `fpm_meta` identifies the request for the "hdf5" meta-package.
2. It calls `init_hdf5(meta_object, compiler_object, all_requests, error_obj)`.
3. `init_hdf5` uses `pkg-config` to find the HDF5 library (e.g., `hdf5-serial`).
4. `fpm_meta_util:add_pkg_config_compile_options` populates `meta_object` with Cflags (include paths) and Libs (library paths and base HDF5 libraries like `libhdf5`).
5. `init_hdf5` then searches for related HL Fortran libraries (e.g., `libhdf5_hl_fortran`) in the found library directory and adds them to `meta_object%link_libs`.
6. It also populates `meta_object%external_modules` with standard HDF5 module names.
7. The `fpm_meta` module then applies the configurations from `meta_object` to the main fpm build model.

# Dependencies and Interactions
- **`fpm_compiler`**: For `compiler_t` type and `get_include_flag`.
- **`fpm_strings`**: For `string_t` and string manipulation functions (`str_begins_with_str`, `str_ends_with`).
- **`fpm_filesystem`**: For `join_path` (used in constructing library file paths for checking existence) and `inquire` (via `exists` which is not directly used here but `inquire` is used directly).
- **`fpm_pkg_config`**: Essential for `assert_pkg_config`, `pkgcfg_has_package`, and `pkgcfg_list_all` to interact with the `pkg-config` system utility.
- **`fpm_meta_base`**: For the `metapackage_t` type and its `destroy` procedure.
- **`fpm_meta_util`**: For `add_pkg_config_compile_options` (to process `pkg-config` output) and `lib_get_trailing` (to help parse library names).
- **`fpm_manifest_metapackages`**: For `metapackage_request_t`.
- **`fpm_error`**: For `error_t` and `fatal_error`.
- **External `pkg-config` tool**: This module heavily relies on `pkg-config` being installed and correctly configured for HDF5.
- **System-installed HDF5 library**: The module's success depends on a system-installed HDF5 library that `pkg-config` can find, and the presence of corresponding library files (including HL Fortran libraries if needed by the project).
The module aims to provide a robust way to link against HDF5, including common but sometimes separately packaged Fortran high-level interfaces.
