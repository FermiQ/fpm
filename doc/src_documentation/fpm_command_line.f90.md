# Overview
The `fpm_command_line` module is responsible for defining and parsing the command-line interface (CLI) for the Fortran Package Manager (fpm). It uses the external `M_CLI2` library to handle the complexities of CLI argument parsing. This module defines all available `fpm` subcommands (like `build`, `run`, `new`, `test`, etc.), their respective options, and generates help messages. Based on the parsed command and options, it populates specific settings types (e.g., `fpm_build_settings`, `fpm_run_settings`) that the main fpm program then uses to drive its operations.

# Key Components
- **Abstract type `fpm_cmd_settings`**: A base type for all subcommand settings. It includes common options like `working_dir`, `path_to_config`, and `verbose`.
- **Specific settings types (extending `fpm_cmd_settings`)**:
  - `fpm_new_settings`: For the `new` subcommand (creating a new package). Options include project name, and flags for including lib, app, test, example, full, or bare structures.
  - `fpm_build_settings`: For the `build` subcommand. Options include listing targets, showing model, building tests, pruning, dumping model, compiler/linker selection, profiles, and flags.
  - `fpm_run_settings`: Extends `fpm_build_settings` for the `run` subcommand. Options include target names, arguments to pass, runner command, and example flag.
  - `fpm_test_settings`: Extends `fpm_run_settings` for the `test` subcommand.
  - `fpm_install_settings`: Extends `fpm_build_settings` for the `install` subcommand. Options include installation prefix, specific directories (bin, lib, include, test), and no-rebuild flag.
  - `fpm_update_settings`: For the `update` subcommand (managing dependencies). Options include specific dependency names, dump file, fetch-only, and clean.
  - `fpm_export_settings`: Extends `fpm_build_settings` for exporting model data. Options for dumping manifest, dependencies, or full model.
  - `fpm_clean_settings`: For the `clean` subcommand. Options for skipping prompt, cleaning all (including dependencies), or cleaning registry cache.
  - `fpm_publish_settings`: Extends `fpm_build_settings` for the `publish` subcommand. Options for showing package version/upload data, dry run, and authentication token.
- **`get_command_line_settings(cmd_settings)`**: The main subroutine that orchestrates the parsing of the command line. It determines the subcommand, sets up specific argument strings for `M_CLI2` based on the subcommand, parses the arguments, and then populates the corresponding settings type (e.g., `fpm_build_settings`) into the output `cmd_settings` variable.
- **`set_help()`**: A subroutine that initializes all the help text strings for `fpm` itself and each of its subcommands. This help text is used by `M_CLI2` when `--help` is invoked.
- **Help text variables**: Numerous `character(len=:), allocatable :: help_...(:)` arrays store the detailed help messages for each command and common option groups (e.g., `help_text_compiler`, `help_text_flag`).
- **`check_build_vals()`**: A utility subroutine to retrieve and set default/environment variable values for common build-related options like compiler and flags.
- **`get_fpm_env(env, default)`**: A function to retrieve environment variables, ensuring they are prefixed with `FPM_` (e.g., `FPM_FC`, `FPM_FFLAGS`).
- **`runner_command(cmd)`**: A function in `fpm_run_settings` to construct the full command string if a runner (e.g., `mpiexec`, `gdb`) is specified.
- **`name_ID(cmd, name)`**: A function in `fpm_run_settings` to find the index of a named target, supporting globbing.

# Important Variables/Constants
- **`manual(*)`**: A character array listing all valid fpm subcommands. Used for generating help for "fpm help manual".
- **Environment variable names (as parameters)**: `fc_env`, `cc_env`, `ar_env`, `fflags_env`, etc., define the environment variable names fpm looks for (e.g., `FPM_FC`, `FPM_FFLAGS`).
- **Default values (as parameters)**: `fc_default`, `cc_default`, `flags_default`, etc., provide default values if environment variables or command-line options are not set.
- **`common_args`, `compiler_args`, `run_args`**: Character strings constructed within `get_command_line_settings` to define the expected options for `M_CLI2` for different groups of commands.

# Usage Examples
This module is used at the very beginning of an `fpm` execution. The main program calls `get_command_line_settings` to parse the user's command and arguments. The populated `cmd_settings` object is then used to decide which fpm action (build, run, new, etc.) to perform and with what parameters.

Example of how `get_command_line_settings` works internally (simplified):
1. Calls `set_help()` to load all help strings.
2. Uses `M_CLI2:get_subcommand()` to identify the primary subcommand (e.g., "build", "run").
3. Based on the subcommand:
    a. Constructs a specific format string for `M_CLI2:set_args()` detailing all expected options, their types, and default values for that subcommand.
    b. Calls `M_CLI2:set_args()`. `M_CLI2` parses the actual command-line arguments provided by the user.
    c. Retrieves parsed values using `M_CLI2:sget()` (for strings), `M_CLI2:lget()` (for logicals), `M_CLI2:unnamed()` (for positional arguments), etc.
    d. Populates an instance of the appropriate settings type (e.g., `allocate(fpm_build_settings :: cmd_settings)`).
    e. Returns this populated `cmd_settings` object.

# Dependencies and Interactions
- **`M_CLI2`**: This is a fundamental external dependency. `fpm_command_line` heavily relies on `M_CLI2` for all CLI parsing tasks (`set_args`, `sget`, `lget`, `get_subcommand`, etc.).
- **`fpm_environment`**: Used to get OS type (`get_os_type`) and environment variables (`get_env`, used by `get_fpm_env`).
- **`fpm_strings`**: For string manipulation functions like `lower`, `split`, `to_fortran_name`, `is_fortran_name`, `glob`.
- **`fpm_filesystem`**: For path operations like `basename`, `canon_path`, and `which`.
- **`fpm_os`**: For getting the current directory (`get_current_directory`).
- **`fpm_error`**: For error handling (`fpm_stop`, `error_t`).
- **`fpm_release`**: To get the fpm version (`fpm_version`) for display in `--version` output.
- **ISO Fortran Env**: For standard I/O units.
- Interacts with the operating system environment to fetch environment variables (e.g., `FPM_FC`, `FPM_FFLAGS`).
- The output `cmd_settings` object is consumed by the main `fpm` program to determine subsequent actions.
- Can execute external `fpm-SUBCOMMAND` executables if a subcommand is not recognized internally but an external command exists (e.g. `fpm-myplugin`).
