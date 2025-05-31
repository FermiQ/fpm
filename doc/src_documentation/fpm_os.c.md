# Overview
This C source file, `fpm_os.c`, provides a cross-platform wrapper function `c_realpath`. Its purpose is to offer a single C function that Fortran code (specifically the `fpm_os.F90` module) can call to get the canonicalized absolute path of a given input path. The C code uses preprocessor directives (`#ifndef _WIN32`, `#else`, `#endif`) to call the appropriate underlying system function: `realpath` on POSIX-compliant systems (like Linux, macOS) or `_fullpath` on Windows.

# Key Components
- **`c_realpath(char* path, char* resolved_path, int maxLength)`**:
  - Takes an input `path` (string).
  - Takes a buffer `resolved_path` where the resulting absolute path will be stored.
  - Takes `maxLength` which is the allocated size of `resolved_path`.
  - **On POSIX systems**: It calls `realpath(path, resolved_path)`. The `realpath` function resolves `.` and `..` components, symbolic links, and returns the null-terminated absolute path.
  - **On Windows systems**: It calls `_fullpath(resolved_path, path, maxLength)`. The `_fullpath` function converts a relative path to an absolute path. It doesn't resolve symbolic links in the same way `realpath` does but provides the absolute path.
  - Returns a pointer to `resolved_path` on success, or `NULL` on failure. The Fortran side checks if the returned pointer is associated.

# Important Variables/Constants
N/A - The logic is contained within the `c_realpath` function and relies on standard C library functions.

# Usage Examples
This C function is designed to be called from Fortran, specifically from the `get_realpath` subroutine in the `fpm_os.F90` module.

Conceptual Fortran interface (from `fpm_os.F90`):
```fortran
interface
    function c_realpath(path, resolved_path, maxLength) result(ptr) &
        bind(C, name="c_realpath")
        import :: c_ptr, c_char, c_int
        character(kind=c_char, len=1), intent(in) :: path(*)
        character(kind=c_char, len=1), intent(out) :: resolved_path(*)
        integer(c_int), value, intent(in) :: maxLength
        type(c_ptr) :: ptr
    end function c_realpath
end interface
```

Fortran usage (simplified from `fpm_os.F90`'s `get_realpath`):
```fortran
! ... Fortran code ...
character(kind=c_char, len=1), allocatable :: c_input_path(:), c_resolved_buffer(:)
character(len=:), allocatable :: fortran_input_path, fortran_resolved_path
type(c_ptr) :: result_ptr
integer(c_int), parameter :: buffer_len = 1000

fortran_input_path = "./some/relative/path"

! Convert Fortran string to C string
! (Simplified, actual conversion in fpm_os.F90 is more robust)
allocate(c_input_path(len_trim(fortran_input_path) + 1))
c_input_path(1:len_trim(fortran_input_path)) = transfer(fortran_input_path, c_input_path(1:len_trim(fortran_input_path)))
c_input_path(len_trim(fortran_input_path) + 1) = c_null_char

allocate(c_resolved_buffer(buffer_len))

result_ptr = c_realpath(c_input_path, c_resolved_buffer, buffer_len)

if (c_associated(result_ptr)) then
    ! Convert C string back to Fortran string
    ! (Simplified, actual conversion in fpm_os.F90 is more robust)
    call c_f_character(c_resolved_buffer, fortran_resolved_path)
    print *, "Absolute path: ", fortran_resolved_path
else
    print *, "Error resolving path."
end if
! ... Fortran code ...
```

# Dependencies and Interactions
- **Standard C Libraries**:
  - `stdlib.h`: Provides `realpath` (on POSIX) and `_fullpath` (on Windows).
  - `stdio.h`: Included, but not directly used by `c_realpath`.
- **`fpm_os.F90`**: This C file provides the implementation for the C procedure `c_realpath`, which is declared via an interface and called from the `get_realpath` subroutine within the `fpm_os.F90` Fortran module.
- **Operating System**: The core functionality is OS-dependent, switching between `realpath` and `_fullpath` based on the `_WIN32` preprocessor macro. This macro is typically defined by C compilers when targeting Windows.
