# Overview
The `fpm_manifest` module is responsible for reading and interpreting the fpm package manifest file (typically `fpm.toml`). It acts as an orchestrator, utilizing various specialized `fpm_manifest_*` sub-modules (e.g., `fpm_manifest_package`, `fpm_manifest_library`, `fpm_manifest_executable`, `fpm_manifest_dependency`) to parse different sections of the TOML file into corresponding Fortran derived types. This module provides the main entry point, `get_package_data`, to load the manifest and also includes logic to apply default configurations if standard project layouts (like `src/`, `app/main.f90`, `test/main.f90`) are detected and not explicitly defined in the manifest.

# Key Components
- **Re-exported Types**:
  - `package_config_t`: The primary derived type representing the entire package configuration, defined in `fpm_manifest_package`.
  - `dependency_config_t`: Represents a dependency declaration, defined in `fpm_manifest_dependency`.
  - `preprocess_config_t`: Represents preprocessor configurations, defined in `fpm_manifest_preprocess`.
- **`get_package_data(package, file, error, apply_defaults)`**: The main public subroutine for loading package configuration.
  - `package`: Output, a `package_config_t` object that will be populated.
  - `file`: Input character string, the path to the manifest file (e.g., "fpm.toml").
  - `error`: Output, allocatable `error_t` for error reporting.
  - `apply_defaults` (optional): Logical. If true (and present), the subroutine will call `package_defaults` to infer default library, executable, or test configurations based on directory/file existence.
  - It calls `fpm_toml:read_package_file` to parse the TOML file into a `toml_table`.
  - Then, it calls `fpm_manifest_package:new_package` to populate the `package_config_t` from the `toml_table`.
- **Default Configuration Subroutines**:
  - **`default_library(self)`**: Populates a `library_config_t` with default settings (source directory "src", include directory "include").
  - **`default_executable(self, name)`**: Populates an `executable_config_t` with default settings (name based on package, source directory "app", main file "main.f90").
  - **`default_example(self, name)`**: Populates an `example_config_t` with default settings (name based on package with "-demo" suffix, source directory "example", main file "main.f90").
  - **`default_test(self, name)`**: Populates a `test_config_t` with default settings (name based on package with "-test" suffix, source directory "test", main file "main.f90").
- **`package_defaults(package, root, error)`**:
  - This subroutine checks for the existence of default directories/files (e.g., `src/`, `app/main.f90`, `example/main.f90`, `test/main.f90`) relative to the project `root`.
  - If these defaults exist and corresponding configurations are not already present in the `package` object (i.e., not explicitly defined in `fpm.toml`), it allocates and populates the respective `library`, `executable`, `example`, or `test` components of the `package` using the `default_*` subroutines.
  - It ensures that a project has at least one buildable component (library, executable, example, or test), otherwise, it issues a fatal error.
- **`get_package_dependencies(package, main, deps)`**:
  - Collects all declared dependencies from a `package_config_t` object into a single flat array `deps` of `dependency_config_t`.
  - `package`: The input package configuration.
  - `main`: Logical, if true, also includes development dependencies and dependencies from executables, examples, and tests. If false, only direct package dependencies are included.
  - This is useful for passing a consolidated list of dependencies to the dependency resolution mechanism.

# Important Variables/Constants
N/A - The module primarily uses procedures and re-exports types from other modules.

# Usage Examples
The `get_package_data` subroutine is the primary way fpm loads and understands an `fpm.toml` file.

```fortran
use fpm_manifest
use fpm_error, only: error_t, fpm_stop
implicit none

type(package_config_t) :: my_package_config
type(error_t), allocatable :: err
character(len=100) :: manifest_file = "fpm.toml"

! Load package data, applying defaults if not specified in fpm.toml
call get_package_data(my_package_config, manifest_file, err, apply_defaults=.true.)

if (allocated(err)) then
    call fpm_stop(1, "Error loading manifest: " // err%message)
end if

! Now my_package_config is populated and can be used to build the fpm_model_t
! For example, print the package name:
print *, "Package Name: ", my_package_config%name
! Access library settings:
if (allocated(my_package_config%library)) then
    print *, "Library source directory: ", my_package_config%library%source_dir
end if
```

# Dependencies and Interactions
- **`fpm_manifest_example`, `fpm_manifest_executable`, `fpm_manifest_dependency`, `fpm_manifest_library`, `fpm_manifest_preprocess`, `fpm_manifest_package`, `fpm_manifest_test`**: These modules define the specific derived types for different sections of the manifest and provide the low-level parsing logic for those sections. `fpm_manifest` re-exports some of these types and uses `fpm_manifest_package:new_package`.
- **`fpm_error`**: For `error_t` and `fatal_error`.
- **`tomlf`**: For the `toml_table` type.
- **`fpm_toml`**: For `read_package_file` which actually reads and parses the TOML file from disk.
- **`fpm_filesystem`**: For path operations (`join_path`, `dirname`, `is_dir`) and file existence checks (`exists`), used in `package_defaults`.
- **`fpm_environment`**: For `os_is_unix` (though not directly used in the provided snippet, it might influence path handling indirectly).
- **`fpm_strings`**: For `string_t`.
- **fpm Build Process**: The `package_config_t` object populated by this module is a crucial input for constructing the `fpm_model_t`, which in turn drives the entire build process.
This module acts as the primary interface for accessing the contents of an `fpm.toml` file in a structured way.
