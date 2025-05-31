# Overview
The `fpm_error` module provides fundamental error handling capabilities for the Fortran Package Manager (fpm). It defines a simple derived type, `error_t`, to encapsulate error messages. The module offers several constructor-like subroutines to create `error_t` objects for different error scenarios, such as fatal errors, syntax errors, file not found errors, and parsing errors. Additionally, it includes a utility subroutine `fpm_stop` to terminate the program execution and print an error or informational message.

# Key Components
- **`error_t`**: A derived type to represent an error.
  - `message`: An allocatable character string that holds the descriptive error message.

- **Error Constructor Subroutines**:
  - **`fatal_error(error, message)`**: Creates an `error_t` object for critical runtime errors.
    - `error`: Output, allocatable `error_t` object.
    - `message`: Input character string, the error description.
  - **`syntax_error(error, message)`**: Creates an `error_t` object for syntax-related errors, often used during parsing of manifest files or source code.
  - **`file_not_found_error(error, file_name)`**: Creates an `error_t` object when a specified file cannot be found. The message includes the name of the missing file.
  - **`file_parse_error(error, file_name, message, line_num, line_string, line_col)`**: Creates a detailed `error_t` object for errors encountered during file parsing.
    - `file_name`: The name of the file being parsed.
    - `message`: The specific parsing error message.
    - `line_num` (optional): The line number where the error occurred.
    - `line_string` (optional): The content of the line where the error occurred.
    - `line_col` (optional): The column number indicating the error position, used to print a `^` marker.
  - **`bad_name_error(error, label, name)`**: A logical function that creates an `error_t` if a given `name` is not a valid Fortran identifier (as checked by `fpm_strings:is_fortran_name`). Returns `.true.` if an error is generated.
    - `label`: A string to prepend to the error message (e.g., "package", "module").
    - `name`: The identifier string to validate.

- **`fpm_stop(value, message)`**: A subroutine to terminate the program.
  - `value`: Integer exit code for the `STOP` statement.
  - `message`: Character string message to be printed to `stderr`. If `value` is greater than 0, it's prefixed with `<ERROR>`; otherwise, with `<INFO>`.
  - It flushes `stderr` and `stdout` before printing the message and stopping.
  - *Note*: The comments suggest future enhancements like using `ERROR STOP` for verbose mode or adding colored output.

# Important Variables/Constants
N/A - The module primarily defines a type and procedures.

# Usage Examples
This module is used throughout fpm whenever an error condition needs to be signaled or the program needs to terminate due to an error.

**Creating and checking for an error:**
```fortran
use fpm_error
type(error_t), allocatable :: err

call check_some_condition(data, err)
if (allocated(err)) then
    call fpm_stop(1, "An error occurred: " // err%message)
    ! Or, if in a subroutine that can propagate the error:
    ! return ! (assuming 'err' is an intent(out) argument of the current sub)
end if

contains
    subroutine check_some_condition(input_data, error_obj)
        ! ...
        type(error_t), allocatable, intent(out) :: error_obj
        if (input_data < 0) then
            call fatal_error(error_obj, "Input data cannot be negative.")
        end if
    end subroutine
```

**Reporting a file parsing error:**
```fortran
! In a parsing routine
use fpm_error
type(error_t), allocatable :: parse_err

! ... (parsing logic) ...
if (unexpected_token) then
    call file_parse_error(parse_err, current_filename, "Unexpected token found.", &
                          current_line_number, current_line_content, token_column)
    ! Propagate parse_err
end if
```

# Dependencies and Interactions
- **`iso_fortran_env`**: For standard I/O units (`stdin`, `stdout`, `stderr`) and `new_line()`.
- **`fpm_strings`**: For `is_fortran_name` and `to_fortran_name` used in `bad_name_error`.
- **Almost all other fpm modules**: Any module that can encounter a recoverable or fatal error condition will typically use `fpm_error` to create an `error_t` object or call `fpm_stop` to terminate.
- The `error_t` objects are often passed as `allocatable, intent(out)` arguments in subroutines, allowing calling routines to check if an error occurred by using `if (allocated(error_variable)) then ...`.
This module provides a simple, consistent way to handle and report errors within the fpm application.
