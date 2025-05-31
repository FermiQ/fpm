# Overview
The `fpm_release` module provides a way to access the version information of the fpm (Fortran Package Manager) executable itself. The version is primarily determined by a preprocessor macro `FPM_RELEASE_VERSION`, which is expected to be defined at compile time (e.g., during the fpm build process, possibly via a build script or CI/CD pipeline). This module ensures that the version information is available programmatically within fpm.

# Key Components
- **`fpm_version()`**: A public function that returns the current fpm version as a `version_t` object (defined in `fpm_versioning`).
  - It uses a preprocessor macro `FPM_RELEASE_VERSION` to get the version string.
  - **Macro Stringification**: It employs a common preprocessor technique to convert the macro `FPM_RELEASE_VERSION` into a Fortran character string. It includes conditional compilation (`#ifdef __GFORTRAN__`) to handle differences in how gfortran's traditional preprocessor and other standard C preprocessors perform stringification.
  - **Fallback Version**: If `FPM_RELEASE_VERSION` is not defined during compilation, it defaults to "0.12.0" as a fallback.
  - The retrieved version string is then parsed into a `version_t` object using `new_version` from the `fpm_versioning` module.
  - If parsing the version string fails, it calls `fpm_stop` to terminate fpm with an error message.

# Important Variables/Constants
- **`FPM_RELEASE_VERSION` (Preprocessor Macro)**: This is the most critical part. It's not a Fortran variable but a C-preprocessor macro that should be defined when fpm itself is compiled. Its value (e.g., "0.8.0", "1.0.0-alpha") determines the version reported by `fpm --version`.
- **`STRINGIFY_START(X)` and `STRINGIFY_END(X)` (Preprocessor Macros)**: These are helper macros used to correctly convert the `FPM_RELEASE_VERSION` macro's value into a string literal that Fortran can use. They handle potential differences between gfortran's traditional preprocessor and standard C preprocessors.

# Usage Examples
This module is primarily used by fpm itself, for example, when the `--version` command-line option is invoked.

**Conceptual fpm main program snippet:**
```fortran
program fpm
  use fpm_command_line, only: get_command_line_settings, fpm_cmd_settings
  use fpm_release, only: fpm_version
  ! ... other modules ...

  class(fpm_cmd_settings), allocatable :: settings
  ! ...
  ! call get_command_line_settings(settings)
  ! ...

  ! If user requested version (e.g. via a specific settings type or a global flag):
  ! if (settings%show_version_flag_is_set) then
  !    print *, "fpm version ", fpm_version()%s()
  !    stop
  ! end if

  ! ... rest of fpm logic ...
end program fpm
```
When fpm is built, the build system would typically pass a flag like `-DFPM_RELEASE_VERSION=X.Y.Z` to the preprocessor.

# Dependencies and Interactions
- **`fpm_versioning`**: For the `version_t` type and the `new_version` subroutine used to parse the version string into a structured `version_t` object.
- **`fpm_error`**: For `error_t` type and the `fpm_stop` subroutine used to halt execution if the version string is invalid.
- **C Preprocessor**: The module relies heavily on C preprocessor directives (`#ifndef`, `#define`, `#ifdef`) to manage the version string.
- **Build System**: The fpm build system is responsible for defining the `FPM_RELEASE_VERSION` macro when compiling fpm. This is how the version information is injected into the executable.
This module provides a standardized way for fpm to report its own version.
