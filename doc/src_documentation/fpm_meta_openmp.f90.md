# Overview
The `fpm_meta_openmp` module is a meta-package handler within fpm that provides the necessary compiler and linker flags to enable OpenMP parallelization. It identifies the current compiler being used by fpm and supplies the appropriate OpenMP flags based on the compiler type (e.g., GCC, Intel, NVHPC, NAG, LFortran, LLVM Flang).

# Key Components
- **`init_openmp(this, compiler, all_meta, error)`**: The primary public subroutine.
  - It takes an uninitialized `metapackage_t` object (`this`) and populates it with OpenMP-specific flags.
  - Calls `destroy(this)` to ensure a clean state.
  - Sets `this%name = "openmp"`.
  - Sets `this%has_build_flags = .true.` and `this%has_link_flags = .true.` because OpenMP typically requires flags for both compilation and linking stages.
  - Uses a `select case (compiler%id)` structure to determine the appropriate OpenMP flags:
    - **GCC (`id_gcc`, `id_f95`)**: Uses `flag_gnu_openmp` (typically `-fopenmp`).
    - **Intel (Windows)**: Uses `flag_intel_openmp_win` (typically `/Qopenmp`).
    - **Intel (Unix-like)**: Uses `flag_intel_openmp` (typically `-qopenmp`).
    - **PGI/NVHPC (`id_pgi`, `id_nvhpc`)**: Uses `flag_pgi_openmp` (typically `-mp`).
    - **IBM XL (`id_ibmxl`)**: Uses `"-qsmp=omp"`.
    - **NAG (`id_nag`)**: Uses `flag_nag_openmp` (typically `-openmp`).
    - **LFortran (`id_lfortran`)**: Uses `flag_lfortran_openmp` (typically `--openmp`).
    - **LLVM Flang (`id_flang`, `id_flang_new`)**: Uses `flag_flang_new_openmp` (typically `-fopenmp`).
    - **Default**: If the compiler is not recognized in the list, it issues a fatal error indicating that OpenMP support for that compiler is not yet implemented in this meta-package.
  - The determined flags are set into `this%flags` (for compilation) and `this%link_flags` (for linking).

# Important Variables/Constants
- The module imports various `flag_*_openmp` constants (e.g., `flag_gnu_openmp`, `flag_intel_openmp_win`) from `fpm_compiler`. These constants hold the actual compiler-specific flag strings.
- Compiler ID constants (e.g., `id_gcc`, `id_intel_classic_windows`) from `fpm_compiler` are used in the `select case` statement.

# Usage Examples
This module is used internally by the `fpm_meta` module when a user requests the "openmp" meta-package in their `fpm.toml` file.

**`fpm.toml` example:**
```toml
name = "my_parallel_code"
version = "0.1.0"

[build]
metapackages = ["openmp"]
```

**Conceptual fpm workflow:**
1. `fpm_meta` receives the request for the "openmp" meta-package.
2. It calls `init_openmp(meta_object, compiler_object, all_requests, error_obj)`.
3. `init_openmp` identifies the current compiler (e.g., `compiler_object%id` is `id_gcc`).
4. It sets `meta_object%flags` and `meta_object%link_flags` to `"-fopenmp"`.
5. `fpm_meta` then applies these flags from `meta_object` to the main fpm build model, which will then be used during the compilation and linking of the project's source files.

# Dependencies and Interactions
- **`fpm_compiler`**: Crucial for `compiler_t` type, compiler ID constants (e.g., `id_gcc`), and the actual OpenMP flag string constants (e.g., `flag_gnu_openmp`).
- **`fpm_strings`**: For `string_t` type.
- **`fpm_meta_base`**: For the `metapackage_t` type and its `destroy` procedure.
- **`fpm_error`**: For `error_t` and `fatal_error`.
- **`fpm_manifest_metapackages`**: For `metapackage_request_t`.
- **Compiler Capabilities**: The effectiveness of this module depends on the chosen compiler actually supporting OpenMP and the correctness of the flags defined in `fpm_compiler`.
The module provides a straightforward way to enable OpenMP by abstracting away the compiler-specific flags.
