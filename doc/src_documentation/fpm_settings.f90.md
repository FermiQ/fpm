# Overview
The `fpm_settings` module handles the management of fpm's global configuration settings. These settings are typically read from a global `config.toml` file. The module defines data types to store these settings, with a particular focus on configurations related to the fpm package registry (e.g., local registry path, remote registry URL, cache path for downloaded dependencies). It provides routines to load settings from the `config.toml` file, apply default values if the file or specific settings are missing, and make these settings available to other parts of fpm.

# Key Components
- **`official_registry_base_url`**: A public parameter defining the URL for the official fpm package registry (`https://fpm-registry.vercel.app`).
- **`default_config_file_name`**: A public parameter for the default name of the global configuration file (`config.toml`).
- **`fpm_registry_settings`**: A derived type to store settings related to the package registry.
  - `path`: Allocatable string, path to a local registry. If set, this is used instead of a remote registry.
  - `url`: Allocatable string, URL for a remote package registry.
  - `cache_path`: Allocatable string, path to the cache folder for downloaded dependencies.
- **`fpm_global_settings`**: A derived type to hold all global settings.
  - `path_to_config_folder`: Allocatable string, path to the directory containing the global config file.
  - `config_file_name`: Allocatable string, name of the config file (defaults to `config.toml`).
  - `registry_settings`: Allocatable `fpm_registry_settings`, holds the registry configurations.
  - Procedures:
    - `has_custom_location()`: Logical function, true if a custom path/name for the config file is set.
    - `full_path()`: Function, returns the full path to the config file.
    - `path_to_config_folder_or_empty()`: Pure function, returns the config folder path or an empty string if not set.
- **`get_global_settings(global_settings, error)`**: The main subroutine to load global settings.
  - It checks if a custom config file location is provided in the `global_settings` input.
  - If custom, it validates the existence of the path and file.
  - If not custom, it determines the default config path (e.g., `~/.local/share/fpm/config.toml` on Unix).
  - If the config file exists, it loads it using `tomlf:toml_load`.
  - It then calls `get_registry_settings` to parse the `[registry]` table from the loaded TOML data.
  - If the config file doesn't exist or the `[registry]` table is missing, it calls `use_default_registry_settings`.
- **`get_registry_settings(table, global_settings, error)`**: A subroutine that parses the `[registry]` TOML table and populates the `global_settings%registry_settings` component. It handles logic for absolute/relative paths for `path` and `cache_path`, and ensures that conflicting settings (e.g., both `path` and `url`, or `path` and `cache_path`) are handled with errors.
- **`use_default_registry_settings(global_settings)`**: A subroutine that applies default registry settings: uses the `official_registry_base_url` and sets a default `cache_path` (e.g., `~/.local/share/fpm/dependencies`).

# Important Variables/Constants
- `official_registry_base_url`: Defines the default remote registry.
- `default_config_file_name`: Defines the standard name for the global config file.
- `valid_keys` (in `get_registry_settings`): A parameter array defining valid keys within the `[registry]` table of `config.toml`, used for validation.

# Usage Examples
This module is used early in fpm's startup process to load global configurations that might affect various operations, especially dependency resolution and fetching.

Conceptual workflow:
1. fpm initializes an `fpm_global_settings` object.
2. If a command-line option like `--config <path>` is provided, `global_settings%path_to_config_folder` and `global_settings%config_file_name` are set accordingly.
3. `call get_global_settings(settings_object, error_obj)` is invoked.
4. `get_global_settings` attempts to load the specified or default `config.toml`.
5. Based on the file content or defaults, `settings_object%registry_settings%url` and `settings_object%registry_settings%cache_path` (or `path`) are populated.
6. Other fpm modules (e.g., `fpm_dependency_tree`) can then access these settings from the `settings_object` to know where to fetch packages from and where to cache them.

```toml
# Example ~/.config/fpm/config.toml or ~/.local/share/fpm/config.toml
[registry]
# url = "https://my-custom-registry.org" # Uncomment to use a custom remote registry
# path = "/path/to/my/local/registry"  # Uncomment to use a local file-based registry
cache_path = "/tmp/fpm_cache"          # Custom cache path
```

# Dependencies and Interactions
- **`fpm_filesystem`**: For `exists`, `join_path`, `get_local_prefix`, `is_absolute_path`, `mkdir`.
- **`fpm_environment`**: For `os_is_unix` to determine default paths.
- **`fpm_error`**: For error handling (`error_t`, `fatal_error`).
- **`tomlf`**: For the core TOML parsing capabilities (`toml_table`, `toml_error`, `toml_stat`, `toml_load`).
- **`fpm_toml`**: For TOML utility functions like `get_value` and `check_keys`.
- **`fpm_os`**: For `get_current_directory`, `change_directory`, `get_absolute_path`, `convert_to_absolute_path` (used for path resolution).
- **`fpm_command_line`**: The command-line parsing module might set a custom config path in `fpm_global_settings` before `get_global_settings` is called.
- **Dependency Management (`fpm_dependency_tree`, etc.)**: These modules would consume the registry URL and cache path settings to manage external packages.
