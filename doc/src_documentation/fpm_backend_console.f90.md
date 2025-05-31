# Overview
The `fpm_backend_console` module provides a Fortran implementation for managing console output, specifically for printing lines and updating previously printed lines. This capability is crucial for the `fpm_backend_output` module, which uses it to display "pretty-printed" build status and progress indicators to the user during the fpm build process. The line updating feature relies on ANSI escape codes and assumes no other output interferes with `stdout`/`stderr` through the `console_t` object. All writes to `stdout` are protected by OpenMP `critical` regions to ensure thread safety during parallel builds.

# Key Components
- **`console_t`**: A derived type (class) that encapsulates the console state and provides methods for interacting with it.
  - `n_line`: An integer that keeps track of the number of lines printed to the console.
- **`console_write_line(console, str, line, advance)`**: A procedure bound to `console_t` (named `write_line`). It writes a string to the standard output.
  - `console`: The `console_t` object.
  - `str`: The string to be printed.
  - `line` (optional, out): If present, returns the line number assigned to this output, which can be used later with `update_line`.
  - `advance` (optional, in): A logical indicating whether to advance to a new line after printing (defaults to `.true.`).
- **`console_update_line(console, line_no, str)`**: A procedure bound to `console_t` (named `update_line`). It overwrites a previously written line on the console.
  - `console`: The `console_t` object.
  - `line_no`: The integer line number (obtained from a previous `write_line` call) to be updated.
  - `str`: The new string to display on that line.

# Important Variables/Constants
- **`ESC`**: `character(len=*), parameter :: ESC = char(27)` - The ASCII escape character.
- **`LINE_RESET`**: `character(len=*), parameter :: LINE_RESET = ESC//"[2K"//ESC//"[1G"` - ANSI escape code to erase the current line and move the cursor to the beginning of the line.
- **`LINE_UP`**: `character(len=*), parameter :: LINE_UP = ESC//"[1A"` - ANSI escape code to move the cursor up one line.
- **`LINE_DOWN`**: `character(len=*), parameter :: LINE_DOWN = ESC//"[1B"` - ANSI escape code to move the cursor down one line.
- **`COLOR_RED`**: `character(len=*), parameter :: COLOR_RED = ESC//"[31m"` - ANSI escape code for red foreground text color.
- **`COLOR_GREEN`**: `character(len=*), parameter :: COLOR_GREEN = ESC//"[32m"` - ANSI escape code for green foreground text color.
- **`COLOR_YELLOW`**: `character(len=*), parameter :: COLOR_YELLOW = ESC//"[93m"` - ANSI escape code for yellow foreground text color.
- **`COLOR_RESET`**: `character(len=*), parameter :: COLOR_RESET = ESC//"[0m"` - ANSI escape code to reset text color to default.

# Usage Examples
This module is primarily used by `fpm_backend_output` to display dynamic build information.

```fortran
! Conceptual example
use fpm_backend_console
implicit none

type(console_t) :: my_console
integer :: line1, line2

call my_console%write_line("Starting process...", line=line1)
! ... do some work ...
call my_console%update_line(line1, COLOR_GREEN // "Process 1: Complete" // COLOR_RESET)

call my_console%write_line("Starting process 2...", line=line2)
! ... do some other work ...
call my_console%update_line(line2, COLOR_YELLOW // "Process 2: In progress..." // COLOR_RESET)
! ... later ...
call my_console%update_line(line2, COLOR_GREEN // "Process 2: Complete" // COLOR_RESET)

call my_console%write_line("All processes finished.")
```

# Dependencies and Interactions
- **`iso_fortran_env`**: Standard Fortran module, used here for `output_unit` (stdout).
- **`fpm_backend_output`**: This is the primary consumer of `fpm_backend_console`. `fpm_backend_output` uses the `console_t` object and its methods to render build status updates.
- **OpenMP**: Uses `!$omp critical` directives to ensure that console output operations are thread-safe when fpm is built with OpenMP and performs parallel builds.
- **Terminal/Console**: Relies on the terminal supporting ANSI escape codes for features like line erasing, cursor movement, and text coloring. If the terminal does not support these, the output might include visible escape characters.
