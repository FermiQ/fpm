# Overview
The `fpm_meta_blas` module is a specialized meta-package module within fpm designed to simplify the integration of BLAS (Basic Linear Algebra Subprograms) libraries into a Fortran project. It attempts to automatically determine the necessary compiler and linker flags required to use BLAS. The module employs a multi-pronged approach:
1.  Checks for macOS's Accelerate framework.
2.  Checks for Intel MKL-specific compiler flags (`-qmkl` or `/Qmkl`).
3.  Falls back to using `pkg-config` to find a system-installed BLAS library (e.g., OpenBLAS, MKL via pkg-config, or a generic `blas` package).

The goal is to allow users to simply request "blas" as a meta-package in their `fpm.toml` and have fpm figure out the rest.

# Key Components
- **`init_blas(this, compiler, all_meta, error)`**: The primary public subroutine. It takes an uninitialized `metapackage_t` object (`this`) and attempts to populate it with BLAS-specific configurations.
  - It first calls `destroy(this)` to ensure a clean state.
  - **macOS Accelerate Framework Check**: If the OS is macOS, it tries to use the `-framework Accelerate` flag (both as a compile and link flag). It verifies if the compiler supports this flag using `compiler%check_flags_supported`.
  - **Intel MKL Check**: If the compiler is identified as Intel (`compiler%is_intel()`), it checks for MKL flags (`-qmkl` for Unix-like, `/Qmkl` for Windows) and verifies their support.
  - **pkg-config Lookup**: If the above methods fail, it asserts that `pkg-config` is available. If so, it iterates through a list of candidate package names (`mkl-dynamic-lp64-tbb`, `openblas`, `blas`) and uses `pkgcfg_has_package` to find the first available one. If found, `add_pkg_config_compile_options` (from `fpm_meta_util`) is called to populate the `metapackage_t` with flags obtained from `pkg-config`.
  - If no suitable BLAS configuration is found, it returns a fatal error.
- **`compile_and_link_flags_supported(compiler, flags)`**: A private logical function that checks if the given `flags` are supported by the `compiler` for both compilation and linking. It uses `compiler%check_flags_supported`.
- **`set_compile_and_link_flags(this, compiler, flags)`**: A private subroutine that sets the provided `flags` into `this%flags` (for compilation) and `this%link_flags` (for linking) and sets the corresponding `has_build_flags` and `has_link_flags` to true in the `metapackage_t` object.

# Important Variables/Constants
- **`candidates(*)`**: A private parameter array of character strings: `['mkl-dynamic-lp64-tbb', 'openblas', 'blas']`. This lists the `pkg-config` package names that `init_blas` will search for, in order of preference.

# Usage Examples
This module is intended to be used internally by the `fpm_meta` module when a user requests the "blas" meta-package in their `fpm.toml` file.

**`fpm.toml` example:**
```toml
name = "my_linear_algebra_app"
version = "0.1.0"

[build]
metapackages = ["blas"]
```

**Conceptual fpm workflow:**
1. `fpm_meta` receives the request for the "blas" meta-package.
2. It calls `init_blas(meta_object, compiler_object, all_requests, error_obj)`.
3. `init_blas` then attempts its detection logic:
   - On macOS, it might find `-framework Accelerate` works. `meta_object`'s `flags` and `link_flags` are set to this.
   - Or, with an Intel compiler, it might find `-qmkl` works. `meta_object`'s flags are set.
   - Or, it finds `openblas` via `pkg-config`. `fpm_meta_util:add_pkg_config_compile_options` populates `meta_object` with include paths from `pkg-config --cflags openblas` and library paths/names from `pkg-config --libs openblas`.
4. The populated `meta_object` is then used by `fpm_meta` to apply these flags to the main build model.

# Dependencies and Interactions
- **`fpm_compiler`**: For `compiler_t` type and its methods `get_include_flag`, `is_intel`, and `check_flags_supported`.
- **`fpm_environment`**: For `get_os_type` and OS constants (`OS_MACOS`, `OS_WINDOWS`).
- **`fpm_meta_base`**: For the `metapackage_t` type and its `destroy` procedure.
- **`fpm_meta_util`**: For `add_pkg_config_compile_options`, which is used to translate `pkg-config` output into `metapackage_t` fields.
- **`fpm_pkg_config`**: For `assert_pkg_config` and `pkgcfg_has_package` to interact with the `pkg-config` system utility.
- **`fpm_manifest_metapackages`**: For `metapackage_request_t` (though not heavily used by `init_blas` itself, it's part of the calling signature from `fpm_meta`).
- **`fpm_strings`**: For `string_t`.
- **`fpm_error`**: For `error_t` and `fatal_error`.
- **External `pkg-config` tool**: If neither macOS Accelerate nor Intel MKL direct flags are applicable, this module relies on `pkg-config` being installed and correctly configured for BLAS libraries.
- **System-installed BLAS libraries**: The effectiveness of this module (especially the `pkg-config` part) depends on having a BLAS library (like OpenBLAS, MKL, or a generic system BLAS) installed on the system and discoverable by `pkg-config` or the specific compiler flags.
