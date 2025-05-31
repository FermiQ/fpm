# Overview
The `fpm_source_parsing` module provides core functionality for analyzing Fortran and C/C++ source files. Its primary purpose is to perform a basic parse of these files to extract essential information needed for dependency tracking and build management within fpm. This includes identifying defined modules, used modules, included files, and the general type of the source unit (program, module, submodule, etc.). Additionally, it calculates a hash (digest) of each file's content, which is crucial for fpm's incremental build capabilities, allowing it to skip recompilation of unchanged files.

The module exposes two main public functions: `parse_f_source` for Fortran files and `parse_c_source` for C/C++ files. Both return a populated `srcfile_t` object (defined in `fpm_model`).

# Key Components
- **`parse_f_source(f_filename, error)`**:
  - Parses a free-form Fortran source file.
  - Identifies `module`, `submodule`, and `program` statements to determine `modules_provided` and `unit_type`.
  - Parses `use` statements to populate `modules_used`, distinguishing intrinsic modules from project/dependency modules.
  - Parses `include` statements (Fortran `INCLUDE` and C-style `#include` if present and simple enough) to populate `include_dependencies`.
  - Determines the `unit_type` (e.g., `FPM_UNIT_MODULE`, `FPM_UNIT_PROGRAM`, `FPM_UNIT_SUBPROGRAM`, `FPM_UNIT_SUBMODULE`). A file is considered `FPM_UNIT_MODULE` only if it exclusively contains modules, to aid in tree-shaking.
  - Calculates an `fnv_1a` hash of the file content and stores it in `f_source%digest`.
  - Handles line continuations for `use ... only:` statements but has limitations for other continued statements.
  - Returns a `srcfile_t` object.
- **`parse_c_source(c_filename, error)`**:
  - Parses C or C++ source files (`.c`, `.h`, `.cpp`).
  - Identifies `#include "..."` preprocessor directives to populate `include_dependencies`. (It specifically looks for quotes, implying it prioritizes local/project includes over system includes with angle brackets for dependency tracking).
  - Sets the `unit_type` to `FPM_UNIT_CSOURCE`, `FPM_UNIT_CHEADER`, or `FPM_UNIT_CPPSOURCE`.
  - Calculates an `fnv_1a` hash of the file content.
  - Returns a `srcfile_t` object.
- **`parse_use_statement(f_filename, i, line, use_stmt, is_intrinsic, module_name, error)`**:
  - A private subroutine called by `parse_f_source` to analyze a single line for a Fortran `USE` statement.
  - Determines if the line contains a `use` statement.
  - Extracts the `module_name`.
  - Checks if the module is an intrinsic Fortran module (e.g., `iso_c_binding`, `iso_fortran_env`, `ieee_arithmetic`, `omp_lib`) or explicitly declared `intrinsic`. Intrinsic modules are not treated as build dependencies.
- **Helper Parsing Functions**:
  - `split_n(string, delims, n, stat)`: Splits a string by delimiters and returns the nth part.
  - `parse_subsequence(string, t1, t2, t3, t4)`: Searches for a sequence of blank-separated tokens within a larger string.
  - `parse_sequence(string, t1, t2, t3, t4)`: Checks if a string starts with a specific sequence of blank-separated tokens.

# Important Variables/Constants
- **`INTRINSIC_NAMES`** (in `parse_use_statement`): A parameter array of character strings listing common Fortran intrinsic module names. This list is used to identify modules that don't need to be tracked as explicit dependencies.

# Usage Examples
This module is used internally by fpm when it scans the source files of a package and its dependencies.

Conceptual workflow:
1. fpm discovers a source file (e.g., `my_module.f90`).
2. It calls `parse_f_source("my_module.f90", error_obj)`.
3. `parse_f_source` reads the file, identifies that it defines `module my_module_impl`, uses `iso_fortran_env` and `another_custom_module`, and includes `"config.inc"`.
4. It populates a `srcfile_t` instance:
   - `file_name = "my_module.f90"`
   - `modules_provided = [string_t("my_module_impl")]`
   - `modules_used = [string_t("another_custom_module")]` (intrinsic modules are skipped)
   - `include_dependencies = [string_t("config.inc")]`
   - `unit_type = FPM_UNIT_MODULE` (assuming no other top-level program/subprograms)
   - `digest = <hash_value>`
5. This `srcfile_t` object is then added to the `package_t` structure within the `fpm_model_t`, which is later used to build the dependency graph and compile targets.

# Dependencies and Interactions
- **`fpm_error`**: For error reporting types and routines (`error_t`, `file_parse_error`, `fatal_error`, `file_not_found_error`).
- **`fpm_strings`**: For `string_t` type, string manipulation functions (`string_cat`, `len_trim`, `split`, `lower`, `str_ends_with`, `is_fortran_name`), and the hashing function `fnv_1a`.
- **`fpm_model`**: Defines the `srcfile_t` type that this module populates, as well as the `FPM_UNIT_*` and `FPM_SCOPE_*` constants.
- **`fpm_filesystem`**: For `read_lines`, `read_lines_expanded` to read file contents, and `exists` to check file existence.
- **`fpm_sources`**: This module likely works in conjunction with `fpm_sources` (which would be responsible for discovering the source files in a project). `fpm_source_parsing` then analyzes the content of these discovered files.
- **`fpm_backend`**: Consumes the `digest` calculated by this module to implement incremental compilation.
- **Build Process**: The information extracted by this module (module dependencies, include dependencies, unit types) is fundamental for constructing the correct build order and command-line arguments for the compiler.

**Limitations noted in the source:**
- Fortran statements, other than `use ... only:`, must not be continued onto another line for the parser to correctly identify them.
