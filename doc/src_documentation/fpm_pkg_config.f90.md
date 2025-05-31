# Overview
The `fpm_pkg_config` module provides a Fortran interface to the `pkg-config` utility. `pkg-config` is a helper tool used on Unix-like systems to retrieve information about installed libraries, such as compiler flags (Cflags) and linker flags (Libs) required to use them. This fpm module wraps calls to the `pkg-config` command-line tool, allowing fpm to query it for information about specific packages (libraries) and use that information to configure the build process.

# Key Components
- **`assert_pkg_config()`**: A logical function that checks if `pkg-config` is available on the system by trying to run `pkg-config -h`. Returns `.true.` if available.
- **`pkgcfg_get_version(package, error)`**: A function that returns the version string of a specified `package` as reported by `pkg-config --modversion <package>`. Returns an empty `string_t` if not found or an error occurs.
- **`pkgcfg_has_package(name)`**: A logical function that checks if a package with the given `name` is known to `pkg-config` by running `pkg-config --exists <name>`.
- **`pkgcfg_get_libs(package, error)`**: A function that returns an array of `string_t` containing the linker flags (libraries and library paths) for the specified `package`. It parses the output of `pkg-config --libs <package>`.
- **`pkgcfg_get_build_flags(name, allow_system, error)`**: A function that returns an array of `string_t` containing the compiler flags (include paths, defines) for the specified `package`. It parses the output of `pkg-config --cflags <name>`.
  - `allow_system`: A logical flag. If true, it attempts to set the `PKG_CONFIG_ALLOW_SYSTEM_CFLAGS` environment variable to `1` before running `pkg-config`. This is sometimes necessary for compilers like gfortran that might not search system paths by default for certain types of flags. The original environment variable state is restored afterwards.
- **`pkgcfg_list_all(error, descriptions)`**: A function that returns an array of `string_t` (`modules`) containing the names of all packages known to `pkg-config`. Optionally, it can also return an array of `string_t` (`descriptions`) with their corresponding descriptions. It parses the output of `pkg-config --list-all`.
- **`run_wrapper(wrapper, args, verbose, exitcode, cmd_success, screen_output)`**: A general-purpose private subroutine used by all other public functions to execute the `pkg-config` command. It constructs the command line, runs it using `execute_command_line` (from `fpm_filesystem` or intrinsic), and captures its output and exit status.

# Important Variables/Constants
N/A - The module primarily relies on the external `pkg-config` tool and string parsing.

# Usage Examples
This module is typically used internally by fpm when a package manifest (`fpm.toml`) specifies a dependency that should be located using `pkg-config`.

Conceptual workflow:
1. fpm parses `fpm.toml` and finds a dependency like `pkgconfig::mylibrary >= 1.0`.
2. fpm calls `assert_pkg_config()` to ensure `pkg-config` is available.
3. It might call `pkgcfg_has_package("mylibrary")` to see if the library is known.
4. If a version is specified, it calls `pkgcfg_get_version("mylibrary", error)` to check it.
5. It then calls `pkgcfg_get_build_flags("mylibrary", ..., error)` and `pkgcfg_get_libs("mylibrary", error)` to get the necessary flags.
6. These flags are then added to the compiler and linker command lines for the fpm project.

```fortran
! Conceptual example
use fpm_pkg_config
use fpm_error, only: error_t
implicit none

character(len=20) :: lib_name = "glib-2.0" ! Example package
type(string_t), allocatable :: cflags(:), libs(:)
type(string_t) :: version_str
type(error_t), allocatable :: err

if (assert_pkg_config()) then
    print *, "pkg-config is available."
    if (pkgcfg_has_package(lib_name)) then
        print *, lib_name, " is known to pkg-config."
        version_str = pkgcfg_get_version(lib_name, err)
        if (allocated(err)) then
            print *, "Error getting version: ", err%message; deallocate(err)
        else
            print *, "Version: ", version_str%s
        end if

        cflags = pkgcfg_get_build_flags(lib_name, .true., err) ! Allow system cflags
        if (allocated(err)) then
            print *, "Error getting Cflags: ", err%message; deallocate(err)
        else
            print *, "Cflags: ", (trim(cflags(i)%s) // " ", i=1,size(cflags))
        end if

        libs = pkgcfg_get_libs(lib_name, err)
        if (allocated(err)) then
            print *, "Error getting Libs: ", err%message; deallocate(err)
        else
            print *, "Libs: ", (trim(libs(i)%s) // " ", i=1,size(libs))
        end if
    else
        print *, lib_name, " is NOT known to pkg-config."
    end if
else
    print *, "pkg-config is NOT available."
end if
```

# Dependencies and Interactions
- **`fpm_strings`**: For `string_t` type and string manipulation functions (`str_begins_with_str`, `len_trim`, `remove_newline_characters`, `split`).
- **`fpm_error`**: For error handling (`error_t`, `fatal_error`, `fpm_stop`).
- **`fpm_filesystem`**: For `get_temp_filename` and `getline` (used within `run_wrapper` to capture command output). The intrinsic `execute_command_line` is also used by `run_wrapper`.
- **`fpm_environment`**: For `get_env`, `os_is_unix`, `set_env`, `delete_env` (used in `pkgcfg_get_build_flags` to manage `PKG_CONFIG_ALLOW_SYSTEM_CFLAGS`).
- **`shlex_module`**: For `shlex_split` used to parse the output of `pkg-config --libs` and `pkg-config --cflags`.
- **External `pkg-config` tool**: This module is entirely dependent on the `pkg-config` executable being installed and accessible in the system's PATH.
- Other fpm modules (like those handling dependency resolution or manifest parsing) would use this module to integrate libraries found via `pkg-config` into the build.
