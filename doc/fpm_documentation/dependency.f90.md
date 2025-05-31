# Overview
The `fpm_dependency` module is a critical part of the Fortran Package Manager (fpm), responsible for managing project dependencies. It defines data structures to represent the dependency graph of a project, including information about each dependency such as its name, version, source (Git repository, path, or registry), and its own transitive dependencies. The module handles the process of fetching remote dependencies, resolving the complete dependency tree, and potentially caching this information to speed up subsequent builds. It interacts with other fpm components like the manifest reader, Git utilities, and registry downloaders.

# Key Components
- **`dependency_node_t`**: A derived type (extending `dependency_config_t` from `fpm_manifest`) representing a single node in the dependency tree.
  - `version`: `version_t`, the actual resolved version of the dependency.
  - `proj_dir`: Character string, the local path to the dependency's source code.
  - `revision`: Character string, the specific Git revision if it's a Git dependency.
  - `done`: Logical, true if this dependency has been processed/resolved.
  - `update`: Logical, true if this dependency needs to be updated (e.g., re-fetched).
  - `cached`: Logical, true if this dependency's information was loaded from a cache.
  - `package_dep`: Array of `string_t`, names of packages this node depends on (its direct dependencies).
  - Procedures:
    - `register()`: Updates the node with information from its fetched manifest.
    - `get_from_registry()`: Fetches the dependency from a package registry.
    - `info()`: Prints information about the node.
    - Serialization procedures: `dependency_node_is_same`, `node_dump_to_toml`, `node_load_from_toml`.
- **`dependency_tree_t`**: A derived type (extending `serializable_t`) representing the entire collection of dependencies for a project.
  - `unit`, `verbosity`: For controlling output.
  - `dep_dir`: Character string, the directory where fetched dependencies are stored (usually `build/dependencies`).
  - `ndep`: Integer, the number of currently registered dependencies.
  - `dep`: Allocatable array of `dependency_node_t`, storing all unique dependencies in a flattened list.
  - `cache`: Character string, path to the cache file for the dependency tree.
  - `path_to_config`: Character string, path to a custom global fpm configuration file.
  - Procedures:
    - `add` (generic): Adds dependencies to the tree (from project config, individual configs, or other nodes).
    - `resolve` (generic): Resolves dependencies by fetching them and reading their manifests.
    - `has(dependency)`: Checks if a dependency (by name or node) is already in the tree.
    - `find(name)`: Returns the index of a dependency by name.
    - `local_link_order(root_id, order, error)`: Determines the correct topological link order for a given node's dependencies.
    - `finished()`: Checks if all dependencies in the tree have been resolved.
    - `load_cache` (generic): Loads dependency tree information from a cache file or TOML table.
    - `dump_cache` (generic): Saves the dependency tree information to a cache file or TOML table.
    - `update` (generic): Updates specified dependencies or the entire tree.
    - Serialization procedures: `dependency_tree_is_same`, `tree_dump_to_toml`, `tree_load_from_toml`.
- **`new_dependency_tree(...)`**: Subroutine to initialize a `dependency_tree_t` instance.
- **`new_dependency_node(...)`**: Subroutine to initialize a `dependency_node_t` instance from a `dependency_config_t`.
- **`add_project(self, package, error)`**: The main entry point for building the dependency tree for a root `package`. It initializes the tree, adds the root package, and then iteratively resolves its dependencies.
- **`resolve_dependency_graph(self, main, error)`**: After fetching, this resolves the internal `package_dep` lists for each node to reflect the full transitive dependency graph.

# Important Variables/Constants
N/A - Functionality is primarily through types and procedures.

# Usage Examples
The `dependency_tree_t` is typically created and populated by the main fpm executable or the build setup process.

Conceptual workflow:
1.  An `fpm_global_settings` object is initialized (potentially with a custom config path).
2.  A `dependency_tree_t` instance is created using `new_dependency_tree`.
3.  The root project's manifest (`package_config_t`) is parsed.
4.  `call dep_tree%add_project(root_package_config, error_obj)` is invoked.
    *   This adds the root project as the first node.
    *   It then iteratively calls `resolve` for each unprocessed dependency.
    *   `resolve` fetches the dependency (if Git or registry), reads its `fpm.toml`, updates the `dependency_node_t` with version/path/revision, and adds its own dependencies to the tree to be processed in subsequent iterations.
    *   Caching mechanisms (`load_cache`, `dump_cache`) are used to avoid re-fetching unchanged dependencies.
5.  Once the tree is built (`dep_tree%finished()` is true), the `dep_tree%dep(:)` array contains all resolved dependencies, and their `proj_dir` can be used to find their source code for compilation.
6.  `dep_tree%local_link_order(...)` can be used to determine correct linking order for libraries.

# Dependencies and Interactions
- **`iso_fortran_env`**: For standard output unit.
- **`fpm_environment`**: For OS type checking.
- **`fpm_error`**: For error handling.
- **`fpm_filesystem`**: For path operations, file/directory existence checks, creating directories.
- **`fpm_git`**: For Git-specific dependency handling (`git_target_revision`, `git_target_default`, `git_revision`).
- **`fpm_manifest`**: For `package_config_t`, `dependency_config_t`, and procedures to read manifest files (`get_package_data`, `get_package_dependencies`).
- **`fpm_manifest_dependency`**: For `manifest_has_changed` and `dependency_destroy`.
- **`fpm_manifest_preprocess`**: For comparing preprocessor configurations.
- **`fpm_strings`**: For `string_t` and string operations.
- **`tomlf`, `fpm_toml`**: For TOML parsing and serialization (used for caching the dependency tree).
- **`fpm_versioning`**: For `version_t` and version handling.
- **`fpm_settings`**: For `fpm_global_settings` to get registry URL, cache paths, etc.
- **`fpm_downloader`**: For fetching packages from a registry.
- **`jonquil`**: For JSON parsing (used when interacting with registry responses).
- **`fpm_model`**: The resolved dependencies (paths to their source, versions) are used to populate the `packages(:)` array in `fpm_model_t`.
This module orchestrates a complex part of fpm's functionality, bridging the gap between declared dependencies in manifests and a resolved set of source code ready for building.
