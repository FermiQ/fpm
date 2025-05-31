# Overview
This C source file, `isatty.c`, located in the `src/ptycheck/` directory, provides a wrapper function `c_isatty`. The purpose of this function is to determine whether the standard output (`stdout`) of the program is connected to an interactive terminal (a TTY) or if it's redirected to a file, pipe, or another non-interactive destination. This check is important for programs that want to behave differently based on whether they are interacting with a user directly (e.g., by printing colorized output or progress bars) or producing output for scripting or logging (e.g., by printing plain text).

The wrapper aims for better portability, especially for Windows environments using MinGW and MSYS2, where the standard `isatty` call might not be sufficient for terminals like MinTTY.

# Key Components
- **`c_isatty(void)`**:
  - This is the main function in the file.
  - It first calls the standard POSIX function `isatty(fileno(stdout))`. `fileno(stdout)` gets the file descriptor for standard output.
  - If `isatty` returns true (non-zero), `c_isatty` returns `1` (indicating it's a terminal).
  - If `isatty` returns false, additional checks are performed for specific environments:
    - **MinGW/MSYS2 (`__MINGW64__` macro)**: If compiled under MinGW (specifically for 64-bit, though the logic might apply more broadly), it then calls `is_cygpty(fileno(stdout))`. This function (defined in `iscygpty.h` and likely `iscygpty.c`) checks if `stdout` is connected to a Cygwin/MSYS pseudo-terminal like MinTTY.
      - If `is_cygpty` returns true, `c_isatty` returns `1`.
      - Otherwise (if not MinGW or `is_cygpty` is false), it returns `0`.
  - If not MinGW and the initial `isatty` was false, it returns `0`.
  - In summary, it returns `1` if `stdout` is a terminal, and `0` otherwise.

# Important Variables/Constants
N/A - The logic is self-contained within the function and relies on standard/external function calls.

# Usage Examples
This C function is intended to be called from Fortran code using C interoperability.

Conceptual Fortran interface (likely found in a module like `fpm_backend` or `fpm_backend_console`):
```fortran
interface
    function c_isatty() bind(C, name='c_isatty')
        use, intrinsic :: iso_c_binding, only: c_int
        integer(c_int) :: c_isatty
    end function c_isatty
end interface
```

Fortran usage:
```fortran
! In a Fortran module/program
use, intrinsic :: iso_c_binding, only: c_int
! ... (interface definition for c_isatty as above) ...

logical :: is_interactive_terminal

if (c_isatty() == 1_c_int) then
    is_interactive_terminal = .true.
    print *, "Output is to a terminal. Will use fancy output."
    ! Call routines that print colors, progress bars, etc.
else
    is_interactive_terminal = .false.
    print *, "Output is redirected. Will use plain text."
    ! Call routines that print simple, log-friendly text.
end if
```
This is typically used in fpm to decide whether to use "pretty" progress bars or plain line-by-line output during the build process.

# Dependencies and Interactions
- **Standard C Libraries**:
  - `unistd.h`: Provides the `isatty` function (POSIX standard).
  - `stdio.h`: Provides `fileno` and `stdout` (POSIX standard).
- **`iscygpty.h` (and its implementation `iscygpty.c`)**:
  - This is a local dependency within the `src/ptycheck/` directory.
  - It's included specifically when `__MINGW64__` is defined, indicating compilation with MinGW.
  - The `is_cygpty` function is used to correctly identify pseudo-terminals like MinTTY in MSYS2/Cygwin environments, which might not be recognized as TTYs by the standard `isatty` call alone when linked with certain Windows C runtimes.
- **Fortran code in fpm**: The `c_isatty` function is called from Fortran (e.g., in `fpm_backend.F90` or `fpm_backend_output.f90`) to determine the output mode for build progress display.
- **Compiler/Environment**: The behavior of the preprocessor directive `#ifdef __MINGW64__` depends on the C compiler and the environment (specifically, if it's a MinGW environment).
The function enhances portability for TTY detection, which is crucial for user experience in command-line applications.
