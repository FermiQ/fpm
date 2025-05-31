# Overview
The `fpm_sources` module is responsible for discovering and categorizing source files within a Fortran project. It scans specified directories for Fortran (`.f90`, `.f`, and custom extensions) and C/C++ (`.c`, `.h`, `.cpp`, `.hpp`) files. Once files are found, it utilizes the `fpm_source_parsing` module to analyze their content and determine their properties (like unit type, provided/used modules). This module plays a key role in populating the `fpm_model_t` with `srcfile_t` objects, which represent the individual source files and their characteristics.

# Key Components
- **`fortran_suffixes`**: Parameter array, default Fortran file extensions (`.f90`, `.f`).
- **`c_suffixes`**: Parameter array, default C/C++ file extensions (`.c`, `.h`, `.cpp`, `.hpp`).
- **`parse_source(source_file_path, custom_f_ext, error)`**: A private function that acts as a wrapper around parsing routines from `fpm_source_parsing`. It determines whether to call `parse_f_source` or `parse_c_source` based on the file extension. If a Fortran source defines a `PROGRAM`, it sets the `exe_name` in the `srcfile_t`.
- **`list_fortran_suffixes(suffixes, with_f_ext)`**: A private subroutine that creates a combined list of default Fortran suffixes and any user-defined custom Fortran extensions (often used for preprocessed files).
- **`add_sources_from_dir(sources, directory, scope, with_executables, with_f_ext, recurse, error)`**:
  - Scans a given `directory` for source files (Fortran and C/C++).
  - Uses `fpm_filesystem:list_files` for directory traversal (optionally recursive).
  - Filters out hidden files and files already present in the input `sources` array.
  - Calls `parse_source` for each valid source file.
  - Assigns the specified `scope` (e.g., `FPM_SCOPE_LIB`, `FPM_SCOPE_APP`) to the discovered `srcfile_t` objects.
  - Optionally ignores Fortran `PROGRAM` units unless `with_executables` is true.
  - Appends the newly found and parsed `srcfile_t` objects to the `sources` array.
- **`add_executable_sources(sources, executables, scope, auto_discover, with_f_ext, error)`**:
  - Specifically handles the discovery and addition of sources for executables (applications or tests) defined in the manifest (`executable_config_t`).
  - First, it calls `add_sources_from_dir` for each unique `executable%source_dir` to gather all files in those directories (if `auto_discover` is true for general files in those app/test directories).
  - Then, it iterates through the `executables` list from the manifest:
    - If a source file matching `executable%main` in `executable%source_dir` was already found (auto-discovered), it updates its `exe_name` (from `executable%name`) and `link_libraries`.
    - If not found (or `auto_discover` was false for that specific main file), it parses the `executable%main` file directly and adds it to the `sources` list, setting its `exe_name`, `link_libraries`, `unit_type` (to `FPM_UNIT_PROGRAM`), and `scope`.
- **`get_executable_source_dirs(exe_dirs, executables)`**: A private subroutine that extracts a list of unique source directories from an array of `executable_config_t`.
- **`get_exe_name_with_suffix(source)`**: A function that returns the executable name from a `srcfile_t`, appending `.exe` on Windows.

# Important Variables/Constants
- `fortran_suffixes`, `c_suffixes`: Define the recognized file extensions for source discovery.

# Usage Examples
This module is primarily used by fpm when constructing the `fpm_model_t`.
1. When processing the `[library]` section of `fpm.toml`, fpm might call `add_sources_from_dir` with the library's source directory (e.g., "src") and `scope = FPM_SCOPE_LIB`.
2. When processing `[[executable]]` entries in `fpm.toml`, fpm calls `add_executable_sources` with the list of executable configurations and `scope = FPM_SCOPE_APP`.
3. Similarly for `[[test]]` entries, using `scope = FPM_SCOPE_TEST`.

Conceptual flow:
```fortran
! In fpm model building logic
use fpm_sources
use fpm_model
! ... other modules

type(srcfile_t), allocatable :: all_project_sources(:)
type(executable_config_t), allocatable :: app_configs(:) ! Populated from manifest
type(error_t), allocatable :: err

! Discover library sources
call add_sources_from_dir(all_project_sources, "src", FPM_SCOPE_LIB, &
                          with_executables=.false., error=err)
! Handle error...

! Discover application sources defined in manifest
call add_executable_sources(all_project_sources, app_configs, FPM_SCOPE_APP, &
                            auto_discover=.true., error=err)
! Handle error...

! 'all_project_sources' now contains srcfile_t objects for all discovered and parsed files.
```

# Dependencies and Interactions
- **`fpm_error`**: For `error_t`.
- **`fpm_model`**: For `srcfile_t` type and `FPM_UNIT_*`, `FPM_SCOPE_*` constants.
- **`fpm_filesystem`**: For `basename`, `canon_path`, `dirname`, `join_path`, `list_files`, `is_hidden_file`.
- **`fpm_environment`**: For OS detection (`get_os_type`, `OS_WINDOWS`) used in `get_exe_name_with_suffix`.
- **`fpm_strings`**: For string operations (`lower`, `str_ends_with`, `string_t`, `.in.` operator).
- **`fpm_source_parsing`**: This is a critical dependency. `fpm_sources` uses `parse_f_source` and `parse_c_source` from this module to analyze the content of discovered files.
- **`fpm_manifest_executable`**: For `executable_config_t` type, which represents executable/test entries from the manifest.
- **`fpm_model` building process**: This module is a key part of populating the `fpm_model_t` with information about the project's source files. The resulting list of `srcfile_t` objects is then used for dependency analysis and build target generation.
