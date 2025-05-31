# Overview
The `fpm_compiler` module is a cornerstone of the fpm build system. It defines abstractions for compilers (`compiler_t`) and archivers (`archiver_t`), providing a unified interface for fpm to interact with different toolchains. The module is responsible for identifying the type of compiler being used (e.g., GCC, Intel, NVHPC), generating appropriate command-line flags for various situations (debug/release profiles, module/include paths, OpenMP, specific Fortran features), and executing the compilation, linking, and archiving commands. It contains a significant amount of logic to handle compiler-specific and OS-specific behaviors.

# Key Components
- **`compiler_enum`**: An enumeration (`enum, bind(C)`) that defines unique identifiers for various compilers (e.g., `id_gcc`, `id_intel_classic_nix`, `id_nvhpc`).
- **`compiler_t`**: A derived type (extending `serializable_t`) representing a compiler.
  - `id`: Integer (`compiler_enum`), stores the identified compiler type.
  - `fc`, `cc`, `cxx`: Allocatable character strings for the paths/names of Fortran, C, and C++ compilers.
  - `echo`, `verbose`: Logical flags controlling command echoing and output verbosity.
  - Procedures:
    - `get_default_flags(release)`: Returns default compile flags for debug or release profiles.
    - `get_module_flag(path)`, `get_include_flag(path)`: Return compiler-specific flags for module output and include directories.
    - `get_feature_flag(feature)`: Returns flags for specific Fortran features like implicit typing or form (free/fixed).
    - `get_main_flags(language)`: Returns special linker flags if the main program is not Fortran.
    - `get_export_flags(target_dir, target_name)`: Returns flags for generating import/export libraries on Windows.
    - `compile_fortran(...)`, `compile_c(...)`, `compile_cpp(...)`: Subroutines to execute compilation for Fortran, C, and C++ source files, respectively. They construct the command and use `fpm_filesystem:run`.
    - `link_shared(...)`, `link_executable(...)` (`link`): Subroutines to link shared libraries and executables.
    - `is_unknown()`, `is_intel()`, `is_gnu()`: Functions to check the compiler type.
    - `enumerate_libraries(prefix, libs)`: Generates linker flags for a list of libraries.
    - `check_fortran_source_runs(...)`, `check_flags_supported(...)`: Test if a small Fortran program compiles and runs with given flags.
    - `with_xdp()`, `with_qp()`: Check for extended (80-bit) and quadruple (128-bit) precision support.
    - `name()`: Returns a string name of the compiler (e.g., "gfortran").
    - Serialization procedures: `compiler_is_same`, `compiler_dump`, `compiler_load`.
- **`archiver_t`**: A derived type (extending `serializable_t`) representing an archiver (for static libraries).
  - `ar`: Allocatable character string for the archiver command (e.g., `ar rs`, `lib /OUT:`).
  - `use_response_file`: Logical, indicates if response files should be used (common on Windows).
  - `echo`, `verbose`: Logical flags.
  - Procedures:
    - `make_archive(...)`: Subroutine to create a static library.
    - Serialization procedures: `ar_is_same`, `dump_to_toml`, `load_from_toml`.
- **`new_compiler(self, fc, cc, cxx, echo, verbose)`**: Subroutine to create and initialize a `compiler_t` instance. It identifies the compiler and sets default C/C++ compilers if not provided.
- **`new_archiver(self, ar, echo, verbose)`**: Subroutine to create and initialize an `archiver_t` instance. It determines the archiver command and default flags.
- **`get_compiler_id(compiler)`**: Function to identify the `compiler_enum` ID from a compiler command string. It handles MPI wrappers by trying to query the underlying compiler.
- **`get_macros(id, macros_list, version)`**: Function to generate compiler flags for defining preprocessor macros.
- **Flag Constants**: Numerous `character(*), parameter :: flag_...` constants define specific flags for different compilers and features (e.g., `flag_gnu_debug`, `flag_intel_openmp_win`).
- **`append_clean_flags(...)`, `append_clean_flags_array(...)`, `tokenize_flags(...)`**: Utility subroutines for managing and tokenizing flag strings, removing duplicates and empty entries.

# Important Variables/Constants
- **Compiler Flags**: The module defines a large set of named constants for compiler flags (e.g., `flag_gnu_coarray`, `flag_intel_debug_win`, `flag_nag_openmp`). These are used by `get_default_flags` and other procedures to construct command lines.
- **Compiler ID Logic**: The `get_compiler_id` and `get_id` functions contain the logic to match compiler executable names (and sometimes their output) to the internal `compiler_enum` IDs. This is crucial for selecting the correct flags and command syntax.
- **OS Specificity**: Many parts of the module, especially flag generation and command execution (e.g., archiver, shared library linking), have OS-dependent logic (e.g., using `/` for options on Windows vs. `-` on Unix-like systems).

# Usage Examples
The `fpm_model` creates instances of `compiler_t` and `archiver_t` based on user configuration (from `fpm.toml` or environment variables like `FPM_FC`). These instances are then passed to the `fpm_backend` module.

When `fpm_backend` needs to compile a source file (e.g., in `build_target`):
1. It calls `model%compiler%compile_fortran(source_file, object_file, combined_flags, log_file, status, compile_command_table)`.
2. Inside `compile_fortran`:
   a. The full command is assembled: `self%fc // " -c " // input // " " // args // " -o " // output`.
   b. `run(command, ...)` is called to execute it.
   c. If a `compile_command_table` is provided, the command is registered.

Similarly for linking or archiving. The `compiler_t` methods like `get_module_flag`, `get_include_flag`, and `get_default_flags` are used by `fpm_targets` or `fpm_model` to construct the `combined_flags` passed to the compile/link calls.

# Dependencies and Interactions
- **`iso_fortran_env`**: For `stderr`.
- **`fpm_environment`**: For OS type detection (`get_os_type`, OS constants) and library filename conventions.
- **`fpm_filesystem`**: For path manipulation (`join_path`, `basename`, `get_temp_filename`, `delete_file`, `unix_path`), running commands (`run`), and reading file lines (`getline`).
- **`fpm_strings`**: For string operations (`split`, `string_cat`, `string_t`, `str_ends_with`, etc.).
- **`fpm_manifest`**: For `package_config_t` (though not directly used in procedures shown, it's part of the broader context of how compiler settings might be influenced).
- **`fpm_error`**: For error handling (`error_t`, `fatal_error`).
- **`tomlf`, `fpm_toml`**: For `serializable_t` and TOML (de)serialization of compiler/archiver settings (though less critical for runtime operation than for configuration persistence).
- **`fpm_compile_commands`**: The `compile_fortran`, `compile_c`, `compile_cpp` procedures can register commands with a `compile_command_table_t`.
- **`shlex_module`**: For robust command-line splitting (`sh_split`, `ms_split`) and quoting (`ms_quote`), especially used in `tokenize_flags` and `get_export_flags`.
- **`fpm_backend`**: Consumes `compiler_t` and `archiver_t` objects to perform build actions.
- **`fpm_model`**: Typically owns and configures the `compiler_t` and `archiver_t` instances.
- **External Compilers/Archivers**: This module directly constructs and executes commands for external toolchain executables (like gfortran, icc, ar, lib).
