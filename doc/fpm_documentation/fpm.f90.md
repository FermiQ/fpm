# Overview
The `fpm` module serves as a high-level coordinator for several key fpm operations, particularly the `build`, `run`, and `clean` commands. It integrates functionalities from various other fpm modules to achieve these tasks. This includes parsing the package manifest (`fpm.toml`), building the internal project model (`fpm_model_t`), discovering and parsing source files, generating build targets, and invoking the build backend. For the `run` command, it also handles the execution of compiled programs, including setting up appropriate environment variables for shared libraries.

# Key Components
- **`build_model(model, settings, package, error)`**:
  - Constructs the `fpm_model_t` which represents the complete build configuration.
  - Initializes compiler and archiver objects (`new_compiler`, `new_archiver`).
  - Sets up compiler flags based on `settings` (profile, custom flags) and defaults.
  - Resolves meta-packages (e.g., for OpenMP, MPI) using `resolve_metapackages`.
  - Initializes and populates the dependency tree (`model%deps`) by calling `model%deps%add(package, error)` and `model%deps%update(error)`.
  - Iterates through all resolved dependencies (including the root package) to:
    - Load their manifest data (`get_package_data`).
    - Populate `model%packages(i)` with information like name, Fortran features, version, and preprocessor settings.
    - Discover library sources using `add_sources_from_dir`.
    - Collect include directories and external module/library linkage information from manifests.
  - Applies C++ preprocessor flags if C++ sources are detected.
  - Discovers executable, example, and test sources based on default directories (`app/`, `example/`, `test/`) if auto-discovery is enabled in the manifest, or from explicit manifest entries, using `add_sources_from_dir` and `add_executable_sources`.
  - Performs checks for invalid module names (`check_module_names`) and duplicate modules (`check_modules_for_duplicates`).
- **`new_compiler_flags(model, settings)`**: A helper subroutine for `build_model` to determine the final compiler flags by combining defaults (for release/debug profiles) and user-provided flags.
- **`check_modules_for_duplicates(model, duplicates_found)`**: Scans all modules provided by all sources in all packages within the model. If duplicate module names are found, it prints a warning and sets `duplicates_found` to true.
- **`check_module_names(model, error)`**: Checks if module names conform to fpm's naming conventions (e.g., prefixing with package name if `module-naming` is enforced). Reports errors if invalid names are found.
- **`cmd_build(settings)`**:
  - Implements the logic for the `fpm build` command.
  - Calls `get_package_data` to load the `fpm.toml`.
  - Calls `build_model` to construct the `fpm_model_t`.
  - Calls `targets_from_sources` to generate the list of `build_target_ptr`.
  - Optionally dumps the model to a file if `settings%dump` is specified.
  - If `settings%list` is true, lists the output files of the targets.
  - If `settings%show_model` is true, prints the model representation.
  - Otherwise, calls `fpm_backend:build_package` to execute the build.
- **`cmd_run(settings, test)`**:
  - Implements the logic for `fpm run` and `fpm test` commands.
  - `settings`: `fpm_run_settings` (or `fpm_test_settings`).
  - `test`: Logical, true if this is for `fpm test`.
  - Loads manifest, builds model, generates targets similar to `cmd_build`.
  - Determines the `run_scope` (e.g., `FPM_SCOPE_APP`, `FPM_SCOPE_TEST`, `FPM_SCOPE_EXAMPLE`).
  - Filters the generated targets to find executables matching the specified scope and names (if any from `settings%name`).
  - Handles errors if no executables are found or if specified names don't match.
  - If `settings%list` is true, lists the matched executables.
  - Otherwise, calls `fpm_backend:build_package` to ensure targets are up-to-date.
  - Modifies the environment (e.g., `LD_LIBRARY_PATH` on Linux, `PATH` on Windows) using `set_library_path` to include directories of any locally built shared libraries, then executes the target programs using `fpm_filesystem:run`.
  - Restores the original library path using `restore_library_path`.
  - Reports errors if any executed program returns a non-zero exit code.
- **`cmd_clean(settings)`**:
  - Implements the `fpm clean` command.
  - Handles cleaning of the registry cache if `settings%registry_cache` is true.
  - If `settings%clean_all` is true, removes the entire `build/` directory.
  - If `settings%clean_skip` is true, calls `delete_skip` to remove build artifacts but not dependencies.
  - Otherwise, prompts the user before calling `delete_skip`.
- **Helper subroutines for `cmd_run`**:
  - `sort_executables`: Sorts executables based on name ID.
  - `should_be_run`: Checks if an executable target should be run based on scope and name matching.
  - `save_library_path`, `set_library_path`, `restore_library_path`: Manage platform-specific environment variables for shared library loading during execution.
- **`delete_skip(is_unix)`**: Helper for `cmd_clean` to delete build subdirectories except the "dependencies" directory.

# Important Variables/Constants
N/A - Logic is primarily driven by procedure calls and types from other modules.

# Usage Examples
This module contains the entry points for fpm's main commands. It's called by the main fpm program after command-line arguments have been parsed.

- `call cmd_build(build_settings_from_cli)`
- `call cmd_run(run_settings_from_cli, .false.)` for `fpm run`
- `call cmd_run(test_settings_from_cli, .true.)` for `fpm test`
- `call cmd_clean(clean_settings_from_cli)`

# Dependencies and Interactions
- **All `fpm_*` modules**: This module is a central orchestrator and thus depends on almost all other core fpm modules:
  - `fpm_strings`, `fpm_backend`, `fpm_command_line`, `fpm_dependency`, `fpm_filesystem`, `fpm_model`, `fpm_compiler`, `fpm_sources`, `fpm_targets`, `fpm_manifest`, `fpm_meta`, `fpm_error`, `fpm_toml`, `fpm_environment`, `fpm_settings`.
- **ISO C Binding**: For C interoperability (though less direct usage within this specific module's procedures compared to others).
- **`iso_fortran_env`**: For standard I/O units.
The `fpm` module ties together most of the package manager's functionality to execute user commands.
