# Overview
The `fpm_filesystem` module provides a comprehensive suite of procedures for interacting with the file system. These routines are essential for fpm's operations, enabling it to manage project files and directories, execute external commands, and handle platform-specific filesystem behaviors. Capabilities include path manipulation, directory listing, file reading/writing, checking for existence, creating directories, and running system commands. It also uses C interoperability for some directory operations, likely for performance or to access features not directly available in standard Fortran.

# Key Components
- **Path Manipulation**:
  - `basename(path, suffix)`: Extracts the filename from a path, optionally removing a suffix.
  - `canon_path(path)`: Canonicalizes a path string (e.g., resolves `.` and `..`), converting to Unix-style separators internally.
  - `dirname(path)`: Extracts the directory part of a path.
  - `parent_dir(path)`: Extracts the parent directory of a path.
  - `join_path(a1, a2, a3, a4, a5)`: Joins up to five path components using the OS-specific separator.
  - `windows_path(path)`, `unix_path(path)`: Convert path separators to Windows (`\`) or Unix (`/`) style.
  - `is_absolute_path(path, is_unix)`: Checks if a path is absolute.
  - `get_dos_path(path, error)`: Converts a Windows path to its 8.3 short name if it contains spaces.
- **File/Directory Checks and Creation**:
  - `is_dir(dir)`: Checks if a given path is an existing directory (uses `test -d` or `cmd /c if exist ...\`).
  - `is_hidden_file(file_basename)`: Checks if a filename starts with a dot (`.`).
  - `exists(filename)`: Checks if a file or directory exists using `INQUIRE`.
  - `mkdir(dir, echo)`: Creates a directory, including parent directories if needed (uses `mkdir -p` or `mkdir`).
- **File Reading/Writing**:
  - `read_lines(filename)`: Reads all lines from a file into an array of `string_t`.
  - `read_lines_expanded(filename)`: Similar to `read_lines` but expands tabs to spaces.
  - `read_text_file(filename)`: Reads the entire content of a text file into a single string.
  - `getline(unit, line, iostat, iomsg)`: Reads a single line of arbitrary length from an open Fortran unit.
  - `delete_file(file)`: Deletes a file.
  - `fileopen(filename, lun, ier)`, `fileclose(lun, ier)`, `filewrite(filename, filedata)`: Basic file open, close, and write operations for text files.
  - `warnwrite(fname, data)`: Writes data to a file only if the file does not already exist.
- **Directory Listing**:
  - `list_files(dir, files, recurse)`: Lists files and directories within a given directory.
    - Has two implementations: one using C interoperability (`c_opendir`, `c_readdir`, etc., if not `FPM_BOOTSTRAP`) and another using system commands (`ls -A` or `dir /b`) as a fallback or for bootstrapping.
    - Can recurse into subdirectories.
- **Command Execution**:
  - `run(cmd, echo, exitstat, verbose, redirect)`: Executes a system command. Provides options for echoing the command, retrieving exit status, controlling verbosity, and redirecting output to a file. This is the preferred way to run commands in fpm.
  - `execute_and_read_output(cmd, output, error, verbose)`: Executes a command and captures its standard output into a string.
  - `which(command)`: Searches for a command in the system's `PATH` and returns its full path if found.
- **System Information**:
  - `get_temp_filename()`: Returns a unique temporary filename using C's `tempnam`.
  - `get_local_prefix(os)`: Determines the default local installation prefix (e.g., `~/.local` on Unix, `%APPDATA%\local` on Windows).
  - `get_home(home, error)`: Gets the user's home directory.
- **C Interoperability Functions (interfaces)**:
  - `c_opendir`, `c_readdir`, `c_closedir`, `c_get_d_name`, `c_is_dir`: Interfaces to C functions (presumably in `filesystem_utilities.c`) for directory operations. These are conditionally compiled if `FPM_BOOTSTRAP` is not defined.

# Important Variables/Constants
- `separator()` (function from `fpm_environment`): Used by `join_path` to determine the correct OS-specific path separator.
- OS type constants (from `fpm_environment`): Used to switch behavior in `is_dir`, `mkdir`, `list_files` (fallback), `run`, `get_local_prefix`, `is_absolute_path`.

# Usage Examples
- **Joining paths**: `my_path = join_path("base_dir", "subdir", "file.txt")`
- **Creating a directory**: `call mkdir(join_path("build", "output"), echo=.false.)`
- **Checking if a directory exists**: `if (is_dir("src")) then ...`
- **Listing files**:
  ```fortran
  type(string_t), allocatable :: all_project_files(:)
  call list_files(".", all_project_files, recurse=.true.)
  ```
- **Running a command**:
  ```fortran
  integer :: status
  call run("my_program_exe --input data.txt", exitstat=status, redirect="output.log")
  if (status == 0) then print *, "Program ran successfully."
  ```
- **Reading a file**:
  ```fortran
  type(string_t), allocatable :: config_lines(:)
  config_lines = read_lines("config.ini")
  ```

# Dependencies and Interactions
- **`iso_fortran_env`**: For standard I/O units.
- **`iso_c_binding`**: For C interoperability features (`c_char`, `c_ptr`, `c_null_char`, etc.).
- **`fpm_environment`**: Crucial for OS detection (`get_os_type`, `os_is_unix`), getting the path separator (`separator`), and environment variables (`get_env`).
- **`fpm_strings`**: For `string_t` type and various string manipulation functions (`replace`, `split`, `split_lines_first_last`, `dilate`, `str_begins_with_str`).
- **`fpm_error`**: For error handling (`fpm_stop`, `error_t`, `fatal_error`).
- **C Standard Library (via C interop)**:
  - `tempnam`, `free` (for `get_temp_filename`).
  - Directory operations (`opendir`, `readdir`, `closedir`, `stat`) via `filesystem_utilities.c` (if not `FPM_BOOTSTRAP`).
- **System Commands**: The module relies on external system commands for some of its functionality, especially in the fallback `list_files` implementation (`ls`, `dir`) and in `is_dir` (`test -d`, `cmd /c if exist`), `mkdir` (`mkdir -p`), and `os_delete_dir` (`rm -rf`, `rmdir /s/q`).
- This module is heavily used by almost all other parts of fpm that need to interact with files or run external processes, including `fpm_model`, `fpm_backend`, `fpm_source_parsing`, `fpm_compiler`, etc.
