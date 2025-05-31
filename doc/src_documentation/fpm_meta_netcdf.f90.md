# Overview
The `fpm_meta_netcdf` module is a specialized meta-package handler within fpm designed to assist with linking against the NetCDF (Network Common Data Form) library. NetCDF is widely used for storing and sharing array-oriented scientific data. This module uses the `pkg-config` utility to find the necessary compiler and linker flags for both the base NetCDF C library and the NetCDF Fortran interface library.

# Key Components
- **`init_netcdf(this, compiler, all_meta, error)`**: The primary public subroutine that configures NetCDF support.
  - It takes an uninitialized `metapackage_t` object (`this`) and populates it.
  - Clears any pre-existing configuration in `this` by calling `destroy(this)`.
  - Sets `this%name = "netcdf"`.
  - **`pkg-config` Prerequisite**: It first checks if `pkg-config` is available using `assert_pkg_config()`. If not, it issues a fatal error.
  - **NetCDF C Library Check**: It then uses `pkgcfg_has_package('netcdf')` to verify that the base NetCDF C library is known to `pkg-config`. If not found, it issues a fatal error. If found, it calls `add_pkg_config_compile_options` (from `fpm_meta_util`) to add the Cflags and Libs for 'netcdf' to `this`.
  - **NetCDF Fortran Library Check**: Subsequently, it uses `pkgcfg_has_package('netcdf-fortran')` to verify that the NetCDF Fortran interface library is known to `pkg-config`. If not found, it issues a fatal error. If found, it calls `add_pkg_config_compile_options` again to append the Cflags and Libs for 'netcdf-fortran' to `this`.
  - **External Modules**: It explicitly defines a list of common NetCDF Fortran module names (e.g., `netcdf`, `netcdf_f03`, `netcdf4_f03`) and assigns them to `this%external_modules`. This informs fpm that these modules are provided by an external library.

# Important Variables/Constants
N/A - The module primarily uses hardcoded `pkg-config` names ('netcdf', 'netcdf-fortran') and a list of standard NetCDF module names.

# Usage Examples
This module is intended for internal use by `fpm_meta` when a user includes "netcdf" in the `metapackages` array of their `fpm.toml` file.

**`fpm.toml` example:**
```toml
name = "my_climate_model_output_processor"
version = "0.1.0"

[build]
metapackages = ["netcdf"]
```

**Conceptual fpm workflow:**
1. `fpm_meta` identifies the "netcdf" meta-package request.
2. It calls `init_netcdf(meta_object, compiler_object, all_requests, error_obj)`.
3. `init_netcdf` verifies `pkg-config` availability.
4. It queries `pkg-config` for "netcdf" and adds its flags to `meta_object`.
5. It then queries `pkg-config` for "netcdf-fortran" and appends its flags to `meta_object`.
6. It populates `meta_object%external_modules` with the standard NetCDF module names.
7. `fpm_meta` applies the combined configurations from `meta_object` to the main fpm build model, enabling the project to compile and link against the NetCDF Fortran library.

# Dependencies and Interactions
- **`fpm_compiler`**: For `compiler_t` type and `get_include_flag`.
- **`fpm_meta_base`**: For the `metapackage_t` type and its `destroy` procedure.
- **`fpm_meta_util`**: For `add_pkg_config_compile_options`, which processes the output of `pkg-config` calls.
- **`fpm_pkg_config`**: Essential for `assert_pkg_config` and `pkgcfg_has_package` to interface with the `pkg-config` tool.
- **`fpm_strings`**: For `string_t`.
- **`fpm_error`**: For `error_t` and `fatal_error`.
- **`fpm_manifest_metapackages`**: For `metapackage_request_t`.
- **External `pkg-config` tool**: The module is critically dependent on `pkg-config` being installed and correctly configured to find both 'netcdf' and 'netcdf-fortran' packages.
- **System-installed NetCDF libraries**: A correctly installed NetCDF C library and NetCDF Fortran library (with corresponding `.pc` files for `pkg-config`) must be present on the system.
The module ensures that both the C and Fortran components of NetCDF are configured for the build.
