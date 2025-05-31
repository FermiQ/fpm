# Overview
The `fpm_meta_util` module provides utility subroutines that support the functionality of other meta-package handler modules (like `fpm_meta_hdf5`, `fpm_meta_blas`). Its main roles are to process the output from `pkg-config` calls, extracting compiler flags, linker flags, include directories, and library names, and to help in determining library file prefixes and suffixes.

# Key Components
- **`add_pkg_config_compile_options(this, name, include_flag, libdir, error)`**:
  - This subroutine is a core utility used by meta-package modules that rely on `pkg-config`.
  - It takes an initialized `metapackage_t` object (`this`), the `pkg-config` package `name` (e.g., "hdf5", "openblas"), the compiler's include flag (e.g., "-I"), and outputs the detected library directory (`libdir`) and an error status.
  - **Version**: If `this%version` is not already allocated, it calls `pkgcfg_get_version` to get the package version from `pkg-config` and stores it.
  - **Libraries**: It calls `pkgcfg_get_libs` to get the linker flags. It then parses these flags:
    - Flags starting with `-l` (e.g., `-lmylib`) are processed to extract the library name ("mylib"), which is added to `this%link_libs`.
    - Flags starting with `-L` (e.g., `-L/path/to/lib`) or `/LIBPATH` (for Windows) are added to `this%link_flags`, and the path is extracted into the output `libdir`.
    - Other linker flags are also added to `this%link_flags`.
  - **Compiler Flags**: It calls `pkgcfg_get_build_flags` to get the compiler flags. It then parses these:
    - Flags matching the provided `include_flag` (e.g., `-I/path/to/include`) are processed to extract the include path, which is added to `this%incl_dirs`.
    - Other compiler flags are added to `this%flags`.
  - It sets the corresponding `has_*` logical flags in `this` (e.g., `has_link_libraries`, `has_include_dirs`) to true if such options are found.
- **`lib_get_trailing(lib_name, lib_dir, prefix, suffix, found)`**:
  - A utility to help determine the actual prefix (like "lib") and suffix (like ".a", ".so", ".dll") of a library file, given its base name and directory.
  - It takes a base `lib_name` (e.g., "hdf5") and the `lib_dir` where it's expected to be.
  - It tries various common library extensions (`.dll.a`, `.a`, `.dylib`, `.dll`) and also checks for a "lib" prefix.
  - It outputs the found `prefix` and `suffix` and sets `found` to true if a matching file is found on the filesystem. This is useful for constructing full library file paths when `pkg-config` might only provide the base name.

# Important Variables/Constants
- **`extensions(*)`** (in `lib_get_trailing`): A private parameter array `['.dll.a', '.a', '.dylib', '.dll']` listing common library file extensions that `lib_get_trailing` will check for.

# Usage Examples
This module's subroutines are primarily called by other `fpm_meta_*` modules.

**Conceptual usage within another meta-package module (e.g., `fpm_meta_customlib.f90`):**
```fortran
module fpm_meta_customlib
    use fpm_meta_base, only: metapackage_t, destroy
    use fpm_meta_util, only: add_pkg_config_compile_options
    use fpm_compiler, only: compiler_t, get_include_flag
    use fpm_pkg_config, only: assert_pkg_config, pkgcfg_has_package
    use fpm_error, only: error_t, fatal_error
    ! ... other imports

contains
    subroutine init_customlib(this, compiler, all_meta, error)
        class(metapackage_t), intent(inout) :: this
        type(compiler_t), intent(in) :: compiler
        ! ... other args ...
        type(error_t), allocatable, intent(out) :: error
        character(len=:), allocatable :: lib_directory_found
        character(len=10) :: pkg_name = "customlib"

        call this%destroy()
        this%name = pkg_name
        ! ... (initialize other fields if necessary)

        if (.not. assert_pkg_config()) then
            call fatal_error(error, pkg_name // " metapackage requires pkg-config")
            return
        end if

        if (.not. pkgcfg_has_package(pkg_name)) then
            call fatal_error(error, "pkg-config could not find " // pkg_name)
            return
        end if

        call add_pkg_config_compile_options(this, pkg_name, &
            get_include_flag(compiler, ""), lib_directory_found, error)

        if (allocated(error)) then
            ! Handle error
        end if
        ! Now 'this' is populated with flags from pkg-config for "customlib"
        ! 'lib_directory_found' might contain the path from -L flag
    end subroutine init_customlib
end module
```

# Dependencies and Interactions
- **`fpm_meta_base`**: For `metapackage_t` type.
- **`fpm_filesystem`**: For `join_path` and `inquire` (used in `lib_get_trailing`).
- **`fpm_strings`**: For `string_t`, `split`, and `str_begins_with_str`.
- **`fpm_error`**: For `error_t`.
- **`fpm_versioning`**: For `new_version`.
- **`fpm_pkg_config`**: This is a key dependency, as `add_pkg_config_compile_options` uses `pkgcfg_get_libs`, `pkgcfg_get_build_flags`, and `pkgcfg_get_version` to fetch information from the `pkg-config` tool.
- **Other `fpm_meta_*` modules**: Modules like `fpm_meta_hdf5`, `fpm_meta_blas`, and `fpm_meta_netcdf` use `add_pkg_config_compile_options` to handle their `pkg-config` interactions. `fpm_meta_hdf5` also uses `lib_get_trailing`.
The utilities in this module centralize common logic needed when meta-packages interface with `pkg-config`.
