# Overview
The `fpm_installer` module provides the framework for installing fpm project artifacts to the system. It defines an `installer_t` derived type that encapsulates installation paths (prefix, bindir, libdir, includedir, testdir) and provides methods to install different types of build products like executables, libraries, and headers into their respective subdirectories within the installation prefix. The module uses system commands for copying files and creating directories.

# Key Components
- **`installer_t`**: A derived type to manage installation settings and operations.
  - `prefix`: Allocatable character string, the main installation root directory (e.g., `/usr/local`, `~/.local`).
  - `bindir`: Allocatable character string, subdirectory for executables (e.g., "bin").
  - `libdir`: Allocatable character string, subdirectory for libraries (e.g., "lib").
  - `testdir`: Allocatable character string, subdirectory for test programs (e.g., "test").
  - `includedir`: Allocatable character string, subdirectory for headers/module files (e.g., "include").
  - `unit`, `verbosity`: For controlling output messages.
  - `copy`, `move`: Allocatable character strings, commands used for copying and moving files (default to `cp -f` or `copy /Y`, and `mv` or `move`).
  - `os`: Integer, stores the detected OS type.
  - Procedures:
    - `install_executable(executable, error)`: Installs an executable to the `bindir`. Appends `.exe` on Windows if not present.
    - `install_library(library_target, error)`: Installs a library (static or shared) to the `libdir`. Handles Windows-specific import library (`.dll.a` or `.lib`) installation for shared libraries.
    - `install_header(header, error)`: Installs a header or module file to the `includedir`.
    - `install_test(test_executable, error)`: Installs a test executable to the `testdir`.
    - `install(source, destination, error)`: A generic routine to install a `source` file to a `destination` subdirectory within the `prefix`. It creates the destination directory if it doesn't exist and then copies the file.
    - `run(command, error)`: Executes an OS command (used internally for copy/move).
    - `make_dir(dir, error)`: Creates a directory if it doesn't exist.
- **`new_installer(...)`**: A subroutine to create and initialize an `installer_t` instance with specified or default paths and commands.
  - Default paths are platform-dependent (e.g., `prefix` defaults to `~/.local` on Unix).
  - Default copy commands include force options (e.g., `cp -f`, `copy /Y`) to overwrite existing files without prompting.

# Important Variables/Constants
- **Default Directory Names**:
  - `default_bindir = "bin"`
  - `default_libdir = "lib"`
  - `default_testdir = "test"`
  - `default_includedir = "include"`
- **Default Copy/Move Commands**:
  - `default_copy_unix = "cp"`, `default_force_copy_unix = "cp -f"`
  - `default_copy_win = "copy"`, `default_force_copy_win = "copy /Y"`
  - `default_move_unix = "mv"`
  - `default_move_win = "move"`

# Usage Examples
This module is used by the `fpm install` command or similar functionality that deploys built artifacts.

Conceptual workflow for `fpm install`:
1.  An `fpm_cmd_settings%fpm_install_settings` object is populated from command-line arguments (e.g., `--prefix`, `--bindir`).
2.  An `installer_t` instance is created using `new_installer`, passing these settings.
    ```fortran
    use fpm_installer
    type(installer_t) :: installer
    ! install_settings would come from parsed command line options
    call new_installer(installer, prefix=install_settings%prefix, bindir=install_settings%bindir, &
                       libdir=install_settings%libdir, includedir=install_settings%includedir, &
                       testdir=install_settings%testdir)
    ```
3.  fpm determines the list of artifacts to install (e.g., executables from `build/app/`, libraries from `build/archive/`).
4.  For each artifact, the appropriate `install_*` method of the `installer_t` object is called.
    ```fortran
    ! Assuming 'my_app_path' is the path to the built executable
    ! and 'my_lib_target' is a build_target_t for the library
    type(error_t), allocatable :: err
    call installer%install_executable(my_app_path, err)
    ! Check err
    call installer%install_library(my_lib_target, err)
    ! Check err
    ! ... install headers/modules ...
    ```

# Dependencies and Interactions
- **`iso_fortran_env`**: For `output_unit`.
- **`fpm_environment`**: For OS detection (`get_os_type`, `os_is_unix`) to select default paths and commands.
- **`fpm_error`**: For `error_t` and `fatal_error`.
- **`fpm_targets`**: For `build_target_t` type and target type constants (e.g., `FPM_TARGET_ARCHIVE`, `FPM_TARGET_SHARED`), used by `install_library`.
- **`fpm_filesystem`**: For path operations (`join_path`, `unix_path`, `windows_path`), directory creation (`mkdir`), file existence checks (`exists`), and getting default local prefixes (`get_local_prefix`). The intrinsic `execute_command_line` is used by the `run` method.
- **System Commands**: Relies on external OS commands for file operations (`cp`, `copy`, `mv`, `move`). The specific commands are determined based on the OS and can be overridden when `new_installer` is called.
- **fpm build process**: The installer acts on the artifacts produced by the build process (executables, libraries, etc.).
This module provides a platform-aware mechanism for deploying the products of an fpm build to user-specified or standard locations.
