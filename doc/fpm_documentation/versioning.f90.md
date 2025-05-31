# Overview
The `fpm_versioning` module provides tools for handling and comparing software version numbers within fpm. It defines a `version_t` derived type, which stores version numbers as an array of integers (e.g., "1.2.3" becomes `[1, 2, 3]`). The module offers procedures to parse version strings into `version_t` objects, convert `version_t` objects back to string representations, and perform comparisons between versions (equality, inequality, greater than, less than, greater/less than or equal to, and a specific "match" for semantic versioning-like checks). It also includes a utility to extract version strings from general text using regular expressions.

# Key Components
- **`version_t`**: A derived type to store version information.
  - `num`: An allocatable array of integers, holding the components of the version number (e.g., major, minor, patch).
  - **Overloaded Operators**:
    - `==` (`equals`): Checks if two versions are identical.
    - `/=` (`not_equals`): Checks if two versions are different.
    - `>` (`greater`): Checks if one version is strictly greater than another.
    - `<` (`less`): Checks if one version is strictly less than another.
    - `>=` (`greater_equals`): Checks if one version is greater than or equal to another.
    - `<=` (`less_equals`): Checks if one version is less than or equal to another.
    - `.match.` (`match`): A custom operator to check if a version `lhs` falls within a range implied by `rhs` (e.g., if `rhs` is "1.2.0", it checks if `1.2.0 <= lhs < 1.3.0`). This is useful for compatibility checks.
  - `s()`: A procedure that converts the `version_t` object back into a string representation (e.g., "1.2.3").
- **`new_version` (interface)**: A generic interface for creating `version_t` objects.
  - `new_version_from_string(self, string, error)`: Parses a version string (e.g., "1.2.3", "0.10.0-alpha") into a `version_t` object. It handles dot-separated numerical components.
  - `new_version_from_int(self, num)`: Creates a `version_t` object directly from an array of integers.
- **`max_limit`**: An integer parameter (value `3`) defining the maximum number of version components (e.g., major.minor.patch) that the parser `new_version_from_string` will handle.
- **`regex_version_from_text(text, what, error)`**: A function that uses regular expressions (`regex_module:regex`) to find and extract the first version-like string (e.g., "1.0.4" or "1.0") from a larger `text`.
  - `what`: A string used in error messages to describe what version was being sought (e.g., "MPI library").
- **Private Helper Subroutines**:
  - `next(string, istart, iend, is_number, error)`: A tokenizer used by `new_version_from_string` to parse components of a version string.
  - `token_error(error, string, istart, iend, message)`: Creates a formatted error message for parsing failures, highlighting the problematic part of the string.
  - `ndigits(self)`: Returns the number of components in a `version_t` object.

# Important Variables/Constants
- `max_limit`: Limits the number of parsed version components to 3 (major, minor, patch).

# Usage Examples
This module is used extensively by fpm for managing package versions, dependency version constraints, and fpm's own version.

**Parsing and Comparing Versions:**
```fortran
use fpm_versioning
use fpm_error, only: error_t
implicit none

type(version_t) :: v1, v2, v_constraint
type(error_t), allocatable :: err
character(len=50) :: version_string1 = "1.2.3"
character(len=50) :: version_string2 = "1.2.0"
character(len=50) :: constraint_string = "1.2.0" ! For .match.

call new_version(v1, version_string1, err)
if (allocated(err)) then; print *, "Error v1: ", err%message; stop; end if
call new_version(v2, version_string2, err)
if (allocated(err)) then; print *, "Error v2: ", err%message; stop; end if
call new_version(v_constraint, constraint_string, err)
if (allocated(err)) then; print *, "Error v_constraint: ", err%message; stop; end if

print *, "v1: ", v1%s()  ! Output: 1.2.3
print *, "v2: ", v2%s()  ! Output: 1.2.0

if (v1 > v2) then
    print *, v1%s(), " is greater than ", v2%s()
end if

if (v1 .match. v_constraint) then
    ! This means 1.2.0 <= 1.2.3 < 1.3.0 (effectively)
    print *, v1%s(), " matches constraint ", v_constraint%s()
end if

if (v2 .match. v_constraint) then
    ! This means 1.2.0 <= 1.2.0 < 1.3.0
    print *, v2%s(), " matches constraint ", v_constraint%s()
end if
```

**Extracting version from text:**
```fortran
use fpm_versioning
use fpm_strings, only: string_t
type(string_t) :: extracted_ver
type(error_t), allocatable :: err
character(len=100) :: text_with_version = "Some library version 2.5.1 compiled on date"

extracted_ver = regex_version_from_text(text_with_version, "Some library", err)
if (allocated(err)) then
    print *, "Error extracting: ", err%message
else
    print *, "Extracted version: ", extracted_ver%s ! Output: 2.5.1
end if
```

# Dependencies and Interactions
- **`fpm_error`**: For `error_t` and `syntax_error`.
- **`fpm_strings`**: For `string_t`.
- **`regex_module`**: For the `regex` function used in `regex_version_from_text`. This is likely an external dependency or another fpm utility module providing regular expression capabilities.
- **`fpm_manifest` and `fpm_dependency`**: These modules use `version_t` to store and compare package versions specified in manifests and resolved for dependencies.
- **`fpm_release`**: Uses `version_t` to represent fpm's own version.
- **`fpm_meta_*` modules**: Some meta-package handlers might use `version_t` to check versions of system libraries (e.g., via `pkg-config` output parsed by `regex_version_from_text`).
The module provides a robust and consistent way to handle semantic-like versioning, which is crucial for dependency resolution and compatibility checks.
