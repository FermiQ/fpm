# Overview
The `fpm_model` module defines the core data structures that fpm uses to represent a Fortran project and its build process. It encapsulates all necessary information, including details about source files, package configurations, compiler settings, dependencies, and Fortran language features. The main type, `fpm_model_t`, acts as a comprehensive container for the entire build context of the root package and all its dependencies. This module is fundamental to fpm's operation, as these data structures are populated by parsing manifest files and source code, and then used to determine build targets and drive the compilation and linking process.

# Key Components

- **Enumerations for Source Files**:
  - **`FPM_UNIT_*`**: Integer parameters defining the type of a source file.
    - `FPM_UNIT_UNKNOWN`: Unknown or not yet determined.
    - `FPM_UNIT_PROGRAM`: Contains a Fortran `PROGRAM`.
    - `FPM_UNIT_MODULE`: Contains one or more Fortran `MODULE`s (and nothing else that would prevent tree-shaking).
    - `FPM_UNIT_SUBMODULE`: Contains one or more Fortran `SUBMODULE`s.
    - `FPM_UNIT_SUBPROGRAM`: Contains non-module Fortran subroutines/functions.
    - `FPM_UNIT_CSOURCE`: C source file.
    - `FPM_UNIT_CHEADER`: C header file.
    - `FPM_UNIT_CPPSOURCE`: C++ source file.
  - **`FPM_SCOPE_*`**: Integer parameters defining the module-usage scope for a source file, which controls module dependency resolution.
    - `FPM_SCOPE_UNKNOWN`: Scope not determined.
    - `FPM_SCOPE_LIB`: Belongs to a library, can be used by other components.
    - `FPM_SCOPE_DEP`: Belongs to a dependency.
    - `FPM_SCOPE_APP`: Belongs to an application (executable).
    - `FPM_SCOPE_TEST`: Belongs to a test executable.
    - `FPM_SCOPE_EXAMPLE`: Belongs to an example executable.

- **`fortran_features_t`**: A derived type (extending `serializable_t`) to specify enabled Fortran language features.
  - `implicit_typing`: Logical, if true, allows implicit typing.
  - `implicit_external`: Logical, if true, allows implicit external interfaces.
  - `source_form`: Allocatable character string (e.g., "free", "fixed", "default") for the Fortran source form.
  - Serialization procedures: `fft_is_same`, `fft_dump_to_toml`, `fft_load_from_toml`.

- **`srcfile_t`**: A derived type (extending `serializable_t`) describing a single source file.
  - `file_name`: Path to the source file.
  - `exe_name`: Name of the executable if this source file is a program unit.
  - `unit_scope`: Integer, one of `FPM_SCOPE_*`.
  - `modules_provided`: Array of `string_t`, names of modules defined in this file.
  - `unit_type`: Integer, one of `FPM_UNIT_*`.
  - `parent_modules`: Array of `string_t`, parent modules for submodules.
  - `modules_used`: Array of `string_t`, names of modules used by this file.
  - `include_dependencies`: Array of `string_t`, files included by this source file.
  - `link_libraries`: Array of `string_t`, native libraries to link against for this specific file.
  - `digest`: `integer(int64)`, a hash/digest of the file content for incremental builds.
  - Serialization procedures: `srcfile_is_same`, `srcfile_dump_to_toml`, `srcfile_load_from_toml`.

- **`package_t`**: A derived type (extending `serializable_t`) describing a single package (either the root project or a dependency).
  - `name`: Package name.
  - `sources`: Array of `srcfile_t`, all source files belonging to this package.
  - `preprocess`: A `preprocess_config_t` object for managing preprocessor settings.
  - `version`: Package version string.
  - `enforce_module_names`: Logical, whether module names should be prefixed.
  - `module_prefix`: `string_t`, the prefix to use if `enforce_module_names` is true.
  - `features`: A `fortran_features_t` object for this package.
  - Procedures:
    - `has_library()`: Returns true if the package contains library source files.
  - Serialization procedures: `package_is_same`, `package_dump_to_toml`, `package_load_from_toml`.

- **`fpm_model_t`**: The main derived type (extending `serializable_t`) that encapsulates the entire build model.
  - `package_name`: Name of the root package.
  - `packages`: Array of `package_t`, including the root package and all its resolved dependencies.
  - `compiler`: A `compiler_t` object.
  - `archiver`: An `archiver_t` object.
  - `fortran_compile_flags`, `c_compile_flags`, `cxx_compile_flags`, `link_flags`: Global command-line flags.
  - `build_prefix`: Base directory for build artifacts.
  - `include_dirs`: Array of `string_t`, additional include directories.
  - `link_libraries`: Array of `string_t`, global native libraries to link against.
  - `external_modules`: Array of `string_t`, modules that are considered external (provided by the compiler or system).
  - `deps`: A `dependency_tree_t` object representing the project's dependency graph.
  - `include_tests`: Logical, whether to build tests.
  - `enforce_module_names`, `module_prefix`: Global settings for module name enforcement.
  - Procedures:
    - `get_package_libraries_link(...)`: Generates linker flags for linking against package dependencies.
  - Serialization procedures: `model_is_same`, `model_dump_to_toml`, `model_load_from_toml`.

- **Utility Functions**:
  - `FPM_SCOPE_NAME(flag)`, `FPM_UNIT_NAME(flag)`: Convert scope/unit integer constants to string representations.
  - `parse_scope(name)`, `parse_unit(name)`: Convert string representations of scope/unit back to integer constants.
  - `info_model(model)`, `info_package(p)`, `info_srcfile(source)`: Functions to generate string representations of the model objects, useful for debugging (`show_model`).
  - `show_model(model)`: Prints a human-readable representation of the `fpm_model_t`.

# Important Variables/Constants
- The `FPM_UNIT_*` and `FPM_SCOPE_*` integer parameters are crucial for classifying source files and determining their role in the build.

# Usage Examples
The `fpm_model_t` is typically constructed by:
1. Reading the `fpm.toml` manifest file.
2. Discovering source files in specified directories (e.g., `src/`, `app/`, `test/`).
3. Parsing these source files (using `fpm_source_parsing`) to identify modules, dependencies, program units, etc., and populating `srcfile_t` instances.
4. Resolving dependencies and populating the `packages` array with information about each dependency.
5. Configuring compiler and archiver objects.

Once the `fpm_model_t` is fully populated, it's used by:
- `fpm_targets`: To generate a list of build targets (object files, executables, libraries).
- `fpm_backend`: To orchestrate the actual compilation and linking using the information stored in the model (compiler paths, flags, source file details).

```fortran
! Conceptual: How a model might be used
use fpm_model
use fpm_targets
use fpm_backend
! ... other modules ...

type(fpm_model_t) :: model
type(build_target_ptr), allocatable :: targets(:)

! 1. Populate the 'model' (details omitted - involves parsing manifests, sources)
! call build_model(manifest_path, settings, model, error_obj)

! 2. Display model information (for debugging)
! call show_model(model)

! 3. Generate build targets from the model
! targets = targets_from_sources(model)

! 4. Build the targets
! call build_package(targets, model, verbose_flag, dry_run_flag)
```

# Dependencies and Interactions
- **`iso_fortran_env`**: For `int64`.
- **`fpm_compiler`**: For `compiler_t` and `archiver_t` types.
- **`fpm_dependency`**: For `dependency_tree_t` type.
- **`fpm_strings`**: For `string_t` and string operations.
- **`tomlf`, `fpm_toml`**: For `serializable_t` and TOML (de)serialization capabilities.
- **`fpm_error`**: For error handling.
- **`fpm_environment`**: For OS constants (`OS_WINDOWS`, `OS_MACOS`).
- **`fpm_manifest_preprocess`**: For `preprocess_config_t`.
- **`fpm_manifest`**: The information from the manifest file (`fpm.toml`) is a primary source for populating the `fpm_model_t`.
- **`fpm_sources`**: Responsible for discovering source files that are then described by `srcfile_t`.
- **`fpm_source_parsing`**: Responsible for parsing individual source files to extract information like `modules_provided`, `modules_used`, `unit_type`, etc.
- **`fpm_targets`**: Consumes the `fpm_model_t` to generate a list of build actions.
- **`fpm_backend`**: Consumes the `fpm_model_t` to execute the build.
This module is at the heart of fpm's internal representation of a project.
