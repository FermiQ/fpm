# Overview
The `fpm_strings` module offers a comprehensive collection of procedures for string manipulation in fpm. It defines a custom `string_t` type (a simple wrapper around an allocatable character string) and provides a wide array of utility functions and subroutines that operate on both standard Fortran `CHARACTER` variables and `string_t` arrays. These utilities cover tasks such as case conversion, string splitting and joining, searching and pattern matching (including globbing), whitespace management (like tab expansion), hashing, and validation of Fortran identifiers and module naming conventions.

# Key Components
- **`string_t`**: A derived type `type string_t; character(len=:), allocatable :: s; end type` for holding allocatable strings.
  - Constructor: `new_string_t(s)` creates a `string_t` from a character string.
  - Equality operators `==` are overloaded for `string_t` and arrays of `string_t`.
- **Type Conversion**:
  - `f_string(c_string)` / `f_string_cptr(cptr)` / `f_string_cptr_n(cptr, n)`: Convert C-style null-terminated character arrays/pointers to Fortran `CHARACTER` strings.
  - `str(i)` / `str_int64(i)` / `str_logical(l)`: Convert integer or logical values to their string representations.
- **Case Manipulation**:
  - `lower(str, begin, end)`: Converts a string to lowercase.
  - `upper(str, begin, end)`: Converts a string to uppercase.
- **Parsing and Joining**:
  - `split(input_line, array, delimiters, order, nulls)`: Splits a string into an array of strings based on delimiters.
  - `split_first_last(string, set, first, last)`: Computes start/end indices of tokens in a string.
  - `split_lines_first_last(string, first, last)`: Computes start/end indices of lines (CR, LF, CRLF delimited).
  - `string_cat(strings, delim)`: Concatenates an array of `string_t` into a single character string with an optional delimiter.
  - `join(str, sep, trm, left, right, start, end)`: Appends elements of a character array into a single character variable with various formatting options.
- **Testing and Searching**:
  - `str_ends_with(s, e)` / `str_ends_with_any(s, e)` / `str_ends_with_any_string(s, e)`: Checks if a string ends with a given suffix or any of an array of suffixes.
  - `str_begins_with_str(s, e, case_sensitive)`: Checks if a string begins with a given prefix.
  - `string_array_contains(search_string, array)` (also `operator(.in.)`): Checks if an array of `string_t` contains a specific character string.
  - `glob(tame, wild)`: Compares a string against a pattern that can include `*` and `?` wildcards.
  - `is_fortran_name(line)`: Checks if a string is a valid Fortran identifier (<=63 chars, starts with letter, alphanumeric and underscore).
  - `to_fortran_name(string)`: Converts a string to a valid Fortran name by replacing hyphens with underscores.
- **Module Naming Validation**:
  - `is_valid_module_name(module_name, package_name, custom_prefix, enforce_module_names)`: Checks if a module name conforms to fpm's naming conventions (Fortran valid, optionally prefixed).
  - `is_valid_module_prefix(module_prefix)`: Checks if a custom module prefix is valid (alphanumeric, starts with letter).
  - `has_valid_custom_prefix(module_name, custom_prefix)`: Checks if a module name is correctly prefixed with a custom prefix.
  - `has_valid_standard_prefix(module_name, package_name)`: Checks if a module name is correctly prefixed using the standard (package name derived) prefix.
  - `module_prefix_template(project_name, custom_prefix)`: Generates a module prefix string based on project name and custom prefix.
  - `module_prefix_type(project_name, custom_prefix)`: Returns "custom" or "default" based on the prefix type.
- **Whitespace Management**:
  - `notabs(instr, outstr, ilen)` / `dilate(instr)`: Expands tab characters to spaces (assuming tab stops every 8 characters). `notabs` is a subroutine, `dilate` is a function.
  - `len_trim(string)` / `len_trim(strings)`: Overloaded to get the trimmed length of a `string_t` or the sum of trimmed lengths for an array of `string_t`.
  - `remove_newline_characters(string)`: Replaces newline characters (CR, LF) in a `string_t` with spaces.
  - `remove_characters_in_set(string, set, replace_with)`: Removes characters from a specified set within an allocatable string, optionally replacing them.
- **Miscellaneous**:
  - `fnv_1a(input, seed)` / `fnv_1a_string_t(input, seed)`: Calculates an FNV-1a hash of a character string or an array of `string_t`.
  - `replace(string, charset, target_char)`: Replaces characters from `charset` in `string` with `target_char`.
  - `resize(list, n)`: Resizes an array of `string_t`.

# Important Variables/Constants
N/A - The module primarily provides procedures.

# Usage Examples
This module is extensively used throughout the fpm codebase for various string processing tasks.
- **Parsing file content**: `split_lines_first_last` followed by string slicing.
- **Validating user input**: `is_fortran_name` for project or package names.
- **Checking module naming conventions**: `is_valid_module_name` and related functions.
- **Generating command line arguments**: `string_cat` or `join`.
- **Hashing file contents for incremental builds**: `fnv_1a`.
- **Normalizing paths or other strings**: `lower`, `replace`, `remove_characters_in_set`.
- **Comparing filenames with wildcards**: `glob`.

```fortran
use fpm_strings
implicit none

character(len=50) :: test_str = "  Hello World!  "
character(len=:), allocatable :: lower_str, tokens(:)
type(string_t) :: my_string
integer :: hash_val

! Case conversion
lower_str = lower(test_str) ! "  hello world!  "

! Splitting
call split(trim(lower_str), tokens, delimiters=" ") ! tokens = ["hello", "world!"]

! string_t usage
my_string = new_string_t("Example.F90")
if (str_ends_with(my_string%s, ".F90")) then
    print *, my_string%s, "is a Fortran file."
end if

! Hashing
hash_val = fnv_1a("some_content_to_hash")
print *, "Hash:", hash_val
```

# Dependencies and Interactions
- **`iso_fortran_env`**: For `int64` and standard I/O units.
- **`iso_c_binding`**: For C string interoperability in `f_string` functions.
- This module is foundational and used by nearly all other fpm modules that deal with text processing, file parsing, manifest handling, command line interpretation, and build logic.
