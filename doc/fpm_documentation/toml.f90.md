# Overview
The `fpm_toml` module serves as an interface and utility layer for handling TOML (Tom's Obvious, Minimal Language) and JSON (JavaScript Object Notation) data within fpm. It primarily acts as a proxy to the `toml-f` Fortran library for TOML processing and integrates the `jonquil` library for JSON processing. The module defines an abstract derived type, `serializable_t`, which provides a common interface for fpm data structures that need to be read from or written to TOML or JSON configuration files (like `fpm.toml` or cache files). It also offers several helper subroutines to get and set values within TOML tables, incorporating fpm-specific error handling.

# Key Components
- **Re-exported `toml-f` and `jonquil` types/modules**:
  - From `tomlf`: `toml_table`, `toml_array`, `toml_key`, `toml_stat`, `get_value` (original), `set_value` (original), `toml_parse`, `toml_error`, `new_table`, `add_table` (original), `add_array`, `toml_serialize`, `len`, `toml_load`.
  - From `jonquil`: `json_serialize`, `json_error`, `json_value`, `json_object`, `json_load`, `cast_to_object`.
- **`serializable_t`**: An abstract derived type defining an interface for objects that can be serialized to and deserialized from TOML or JSON.
  - `dump_to_toml(table, error)` (deferred): Abstract procedure to write the object's state to a `toml_table`.
  - `load_from_toml(table, error)` (deferred): Abstract procedure to populate the object from a `toml_table`.
  - `serializable_is_same(this, that)` (deferred, `operator(==)`): Abstract function to compare two serializable objects.
  - `dump(unit/file, error, json)` (non-overridable): Concrete procedures to write the object to a Fortran unit or a file, with an option to format as JSON.
  - `load(unit/file, error, json)` (non-overridable): Concrete procedures to read an object from a Fortran unit or a file, with an option to parse as JSON.
  - `test_serialization(message, error)`: A procedure to test if an object can be written and then read back correctly in both TOML and JSON formats.
- **Interface Wrappers for `tomlf` procedures**:
  - `get_value`: Generic interface for type-specific getters (`get_logical`, `get_integer`, `get_integer_64`, `get_char` for allocatable strings, `get_string` for `string_t`) that retrieve values from a `toml_table` and provide fpm-specific error handling (`fatal_error`).
  - `set_value`: Generic interface for type-specific setters (`set_logical`, `set_integer`, `set_integer_64`).
  - `set_string`: Generic interface for `set_character` (for `character(len=*)`) and `set_string_type` (for `string_t`).
  - `add_table`: Wraps `tomlf:add_table` with fpm error handling.
- **`read_package_file(table, manifest, error)`**: Subroutine to read an fpm manifest file (e.g., `fpm.toml`) into a `toml_table`. It handles file existence checks and TOML parsing errors.
- **`has_list(table, key)`**: Logical function to check if a `toml_table` contains an array associated with `key`.
- **`get_list(table, key, list, error)`**: Subroutine to retrieve an array of strings from a `toml_table` into an array of `string_t`. Handles cases where the TOML value might be a single string or an array of strings.
- **`set_list(table, key, list, error)`**: Subroutine to write an array of `string_t` to a `toml_table` under `key`. If the list has one element, it writes a single string; otherwise, it writes a TOML array.
- **`check_keys(table, valid_keys, error)`**: Subroutine to validate that a `toml_table` only contains keys from a predefined list of `valid_keys`. Reports an error if unknown keys are found or if value types are unexpected.
- **`name_is_json(filename)`**: Logical function to determine if a filename ends with ".json" (case-insensitive), suggesting it should be parsed as JSON.

# Important Variables/Constants
N/A

# Usage Examples
This module is fundamental for how fpm interacts with configuration files.
- **Loading `fpm.toml`**: `fpm_manifest` uses `read_package_file` to load the manifest.
- **Serialization of fpm data types**: Many fpm internal data structures (like `dependency_node_t`, `fpm_model_t`, `git_target_t`) extend `serializable_t` and implement its deferred procedures. This allows them to be, for instance, cached to disk.
  ```fortran
  ! Assuming my_object is of a type that extends serializable_t
  ! type(my_custom_serializable_type) :: my_object
  ! type(error_t), allocatable :: err

  ! Save to a TOML file
  ! call my_object%dump("cache_data.toml", err)

  ! Load from a TOML file
  ! call my_object%load("cache_data.toml", err)
  ```
- **Accessing values from a parsed TOML table**:
  ```fortran
  ! type(toml_table) :: config_table
  ! logical :: enable_feature
  ! character(len=:), allocatable :: project_name
  ! type(error_t), allocatable :: err

  ! call get_value(config_table, "feature_enabled", enable_feature, err, "my_feature_config")
  ! call get_value(config_table, "project_name", project_name, err, "project_settings")
  ```

# Dependencies and Interactions
- **`fpm_error`**: For `error_t`, `fatal_error`, `file_not_found_error`.
- **`fpm_strings`**: For `string_t` type and string utilities like `str_ends_with`, `lower`.
- **`tomlf` (external library)**: This is the primary backend for TOML parsing and serialization. `fpm_toml` re-exports many of its components and builds upon them.
- **`tomlf_de_parser` (part of `tomlf` or related)**: Specifically for the `parse` procedure (though not directly used in the public interface of `fpm_toml`, it's a dependency of `tomlf`).
- **`jonquil` (external library)**: This is the primary backend for JSON parsing and serialization, used when the `json=.true.` option is passed to `dump` or `load` methods of `serializable_t`.
- **`iso_fortran_env`**: For `int64`.
- **All fpm modules defining data structures that need to be persisted or configured via TOML/JSON**: These modules will typically have types that extend `serializable_t` and use the helper functions in `fpm_toml` to interact with `toml_table` objects. Examples include `fpm_manifest_*` modules, `fpm_dependency`, `fpm_model`, etc.
This module provides a crucial abstraction layer over TOML and JSON processing libraries, tailoring their use for fpm's specific needs and error handling.
