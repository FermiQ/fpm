# Overview
The `fpm_os` module provides Fortran interfaces for interacting with the operating system at a low level, primarily focusing on directory and path manipulation. It allows fpm to change the current working directory, retrieve the current working directory, and resolve relative paths to their absolute and canonical forms. The module achieves this by using Fortran's C interoperability features to call underlying C library functions (like `chdir`, `getcwd`, `realpath`, `_fullpath`). It handles platform differences (POSIX vs. Windows) through conditional compilation in C (via `_WIN32` macro) or by calling platform-specific C functions.

# Key Components
- **C Function Interfaces**: The module defines Fortran interfaces to several C functions:
  - `chdir_` (maps to `chdir` on POSIX, `_chdir` on Windows): Changes the current working directory.
  - `getcwd_` (maps to `getcwd` on POSIX, `_getcwd` on Windows): Gets the current working directory.
  - `realpath` (POSIX only): Resolves a path to its canonicalized absolute pathname.
  - `fullpath` (Windows only, `_fullpath`): Resolves a path to its absolute pathname.
  - `c_realpath`: A custom C wrapper (likely in `fpm_os.c`) that calls either `realpath` or `_fullpath` based on the OS. This interface is used by `get_realpath` and is not available in `FPM_BOOTSTRAP` mode.
- **`change_directory(path, error)`**: A subroutine that changes the current working directory to `path`. Uses `chdir_`.
- **`get_current_directory(path, error)`**: A subroutine that retrieves the current working directory into `path`. Uses `getcwd_`.
- **`get_realpath(path, real_path, error)`**: A subroutine to get the canonicalized absolute path. It uses the `c_realpath` C function. Not available in `FPM_BOOTSTRAP` mode.
- **`get_absolute_path(path, absolute_path, error)`**: A subroutine that converts a given `path` (which can be relative, or start with `~`) into an absolute path.
  - If `FPM_BOOTSTRAP` is defined, it uses `get_absolute_path_by_cd`.
  - Otherwise, it handles `~` expansion manually (using `get_home` from `fpm_filesystem`) and then calls `get_realpath`.
- **`get_absolute_path_by_cd(path, absolute_path, error)`**: An alternative way to get an absolute path by temporarily changing to the target directory and then calling `get_current_directory`. This is used in bootstrap mode where `c_realpath` might not be available.
- **`convert_to_absolute_path(path, error)`**: A convenience subroutine that takes an allocatable `path` and replaces it with its absolute, canonical version.
- **Helper Subroutines for C Strings**:
  - `f_c_character(rhs, lhs, len)`: Converts a Fortran string (`rhs`) to a C-style null-terminated character array (`lhs`).
  - `c_f_character(rhs, lhs)`: Converts a C-style null-terminated character array (`rhs`) to an allocatable Fortran string (`lhs`).

# Important Variables/Constants
- `buffersize`: An integer parameter (`1000_c_int`) used for allocating character buffers for C function calls.
- `pwd_env`: A character parameter, set to `"PWD"` on POSIX-like systems and `"CD"` on Windows. (Note: This constant is defined but not explicitly used in the provided Fortran code snippet for `fpm_os.F90`, it might be intended for use elsewhere or was part of a previous implementation detail).

# Usage Examples
- **Changing directory**:
  ```fortran
  use fpm_os
  type(error_t), allocatable :: err
  call change_directory("../another_dir", err)
  if (allocated(err)) then
      print *, "Failed to change directory: ", err%message
  end if
  ```
- **Getting current directory**:
  ```fortran
  use fpm_os
  type(error_t), allocatable :: err
  character(len=:), allocatable :: cwd
  call get_current_directory(cwd, err)
  if (allocated(err)) then
      print *, "Failed to get CWD: ", err%message
  else
      print *, "Current directory is: ", cwd
  end if
  ```
- **Getting absolute path**:
  ```fortran
  use fpm_os
  type(error_t), allocatable :: err
  character(len=:), allocatable :: abs_path
  call get_absolute_path("./relative_file.txt", abs_path, err)
  ! abs_path will now hold the full, canonical path to relative_file.txt
  ```

# Dependencies and Interactions
- **`iso_c_binding`**: Essential for all C interoperability.
- **`fpm_filesystem`**: For `exists`, `join_path`, and `get_home` (used in `get_absolute_path` for `~` expansion).
- **`fpm_environment`**: For `os_is_unix` to determine platform-specific behavior (e.g., in `get_absolute_path` for `~` expansion separator).
- **`fpm_error`**: For error reporting (`error_t`, `fatal_error`).
- **C Standard Library / OS System Calls**: The module's core functionality relies on C functions like `chdir`, `getcwd`, `realpath` (POSIX) and `_chdir`, `_getcwd`, `_fullpath` (Windows) to interact with the operating system.
- **`fpm_os.c`**: This Fortran module likely depends on a corresponding C file (`fpm_os.c`) that implements the `c_realpath` wrapper function.
- Various fpm modules might use `fpm_os` to manage working directories or resolve paths, especially before calling external tools or when dealing with user-provided paths.
- **Bootstrap Mode (`FPM_BOOTSTRAP`)**: The behavior of `get_absolute_path` changes in bootstrap mode, falling back to a method (`get_absolute_path_by_cd`) that doesn't rely on the `c_realpath` C function, which might not be compiled/available during the initial bootstrap phase of fpm itself.
