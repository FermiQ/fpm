# Overview
The `fpm_meta_minpack` module is a meta-package handler within fpm that facilitates the inclusion of the MINPACK library in a Fortran project. MINPACK is a collection of Fortran subroutines for solving systems of nonlinear equations and nonlinear least squares problems. Unlike meta-packages for system libraries like BLAS or HDF5 (which look for pre-installed versions), `fpm_meta_minpack` treats MINPACK as an external fpm package. It defines a dependency on the official Fortran-lang MINPACK Git repository, specifying a particular tag or version to be fetched by fpm's dependency management system.

# Key Components
- **`init_minpack(this, compiler, all_meta, error)`**: The primary public subroutine.
  - It takes an uninitialized `metapackage_t` object (`this`) and populates it with the configuration for MINPACK.
  - Calls `destroy(this)` to ensure a clean state.
  - Sets `this%name = "minpack"`.
  - Sets `this%has_dependencies = .true.`, indicating that this meta-package introduces an fpm package dependency.
  - Allocates one entry in `this%dependency`.
  - Configures this dependency entry:
    - `this%dependency(1)%name = "minpack"`
    - `this%dependency(1)%git`: This is set using `git_target_tag("https://github.com/fortran-lang/minpack", "v2.0.0-rc.1")`. This means fpm will fetch the MINPACK package from the specified GitHub URL, checking out the tag "v2.0.0-rc.1".
  - If the Git dependency cannot be initialized (e.g., `git_target_tag` fails), it returns a fatal error.

# Important Variables/Constants
N/A - The key information is the hardcoded Git repository URL and tag for MINPACK.

# Usage Examples
This module is used internally by the `fpm_meta` module when a user requests the "minpack" meta-package in their `fpm.toml` file.

**`fpm.toml` example:**
```toml
name = "my_optimization_program"
version = "0.1.0"

[build]
metapackages = ["minpack"]
```

**Conceptual fpm workflow:**
1. `fpm_meta` receives the request for the "minpack" meta-package.
2. It calls `init_minpack(meta_object, compiler_object, all_requests, error_obj)`.
3. `init_minpack` configures `meta_object` to have a dependency on the fortran-lang/minpack GitHub repository at tag `v2.0.0-rc.1`.
4. The `fpm_meta` module then calls `meta_object%resolve(package_config, error_obj)`.
5. This adds the MINPACK Git dependency to the main project's list of dependencies.
6. When fpm resolves dependencies, it will clone the MINPACK repository at the specified tag and build it as part of the overall project build. The user's project can then `use minpack_*` modules.

# Dependencies and Interactions
- **`fpm_compiler`**: For `compiler_t` type (passed in but not directly used by `init_minpack` itself, as MINPACK is treated as a source dependency whose build will be configured by its own manifest or fpm defaults).
- **`fpm_meta_base`**: For the `metapackage_t` type and its `destroy` procedure.
- **`fpm_error`**: For `error_t` and `fatal_error`.
- **`fpm_git`**: For `git_target_tag` function, which creates the Git dependency information.
- **`fpm_manifest_metapackages`**: For `metapackage_request_t`.
- **`fpm_manifest_dependency`**: The `dependency(1)%git` component within `metapackage_t` is of type `git_dependency_t`, which is defined here. (Implicitly, as `metapackage_t` contains `dependency_config_t` which can hold a git target).
- **fpm's dependency management system**: This meta-package relies on fpm's ability to fetch and build Git-based dependencies.
- **Internet connectivity**: Required at build time (at least the first time or when dependencies are updated) to fetch the MINPACK source code from GitHub.
The primary role of this module is to inject a specific version of the fortran-lang/minpack package as a dependency into the user's project.
