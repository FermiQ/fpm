# Overview
The `fpm_git` module provides functionalities for fpm to interact with Git repositories. This is primarily used for managing Git-based dependencies. The module defines a `git_target_t` derived type to describe a specific Git source (URL and a reference like a branch, tag, or commit hash). It includes procedures to perform Git operations such as fetching a repository, checking out a specific reference, and querying the current revision of a local repository. It also has a utility to create an archive from a Git repository.

# Key Components
- **`enum_descriptor` / `git_descriptor`**: Defines an enumeration for different types of Git references:
  - `default`: Typically HEAD or the default branch.
  - `branch`: A specific branch name.
  - `tag`: A specific tag name.
  - `revision`: A specific commit hash (SHA1).
  - `error`: An invalid descriptor.
- **`git_target_t`**: A derived type (extending `serializable_t`) to represent a Git dependency.
  - `descriptor`: Integer, one of the `git_descriptor` values.
  - `url`: Allocatable character string, the URL of the Git repository.
  - `object`: Allocatable character string, the specific branch, tag, or commit hash.
  - Procedures:
    - `checkout(local_path, error)`: Clones/fetches the Git repository to `local_path` and checks out the specified `object` (or HEAD if `object` is not allocated). It uses `git init`, `git fetch --depth=1`, and `git checkout -qf FETCH_HEAD`.
    - `info(unit, verbosity)`: Prints information about the `git_target_t` instance.
    - Serialization procedures: `git_is_same`, `dump_to_toml`, `load_from_toml`.
- **Constructor Functions for `git_target_t`**:
  - `git_target_default(url)`: Creates a target for the default reference.
  - `git_target_branch(url, branch)`: Creates a target for a specific branch.
  - `git_target_tag(url, tag)`: Creates a target for a specific tag.
  - `git_target_revision(url, sha1)`: Creates a target for a specific commit hash.
- **`git_revision(local_path, object, error)`**: A subroutine that gets the current commit hash (SHA1) of a local Git repository located at `local_path`. It uses `git log -n 1`.
- **`git_matches_manifest(cached, manifest, verbosity, iunit)`**: A logical function to compare a cached `git_target_t` with one defined in a manifest to see if they refer to the same Git source and object.
- **`git_archive(source, destination, ref, additional_files, verbose, error)`**: A subroutine to create an archive (e.g., `tar.gz`) from a Git repository. It uses `git archive`.
  - `source`: Directory of the Git repository.
  - `destination`: Path for the output archive file.
  - `ref`: The Git reference (branch, tag, commit) to archive.
  - `additional_files` (optional): Untracked files to include in the archive.
- **Helper functions for serialization**: `parse_descriptor`, `descriptor_name`.

# Important Variables/Constants
- **`compressed_package_name`**: Parameter, `"compressed_package"`. (Its usage is not directly evident in this module but might be related to temporary naming when fetching/archiving).
- **`git_descriptor`**: A parameter of `enum_descriptor` type, providing access to the enumeration constants.

# Usage Examples
This module is primarily used by `fpm_dependency` when a dependency is specified with a `git` key in the `fpm.toml` manifest.

**`fpm.toml` dependency example:**
```toml
[dependencies]
my_lib = { git = "https://github.com/user/my_lib.git", tag = "v1.0.2" }
another_lib = { git = "https://example.com/another_lib.git", branch = "develop" }
```

**Conceptual fpm workflow:**
1.  `fpm_manifest` parses the `fpm.toml` and creates a `dependency_config_t` with a `git_target_t` component (e.g., populated by `git_target_tag(...)`).
2.  `fpm_dependency` processes this dependency.
3.  It calls `checkout(local_checkout_path, error_obj)` on the `git_target_t` object to clone/fetch the repository into a local directory (e.g., `build/dependencies/my_lib`).
4.  It might then call `git_revision(local_checkout_path, actual_commit_hash, error_obj)` to record the exact commit that was checked out.
5.  The `fpm publish` command might use `git_archive` to package the current project.

# Dependencies and Interactions
- **`fpm_error`**: For `error_t` and `fatal_error`.
- **`fpm_filesystem`**: For `get_temp_filename`, `getline`, `join_path`, `execute_and_read_output`, `run`. These are used to execute Git commands and capture their output.
- **`tomlf`, `fpm_toml`**: For `serializable_t` and TOML (de)serialization of `git_target_t` objects (likely used for caching dependency information).
- **External `git` command-line tool**: This module is entirely dependent on the `git` executable being installed and accessible in the system's PATH. All Git operations are performed by constructing and running `git` commands.
- **`fpm_dependency`**: The primary consumer of this module for fetching and managing Git-based dependencies.
- **`fpm_publish` (potentially)**: Could use `git_archive` for packaging a project.
The module abstracts Git operations into a Fortran-friendly interface, enabling fpm to handle remote source code dependencies.
