# Overview
The `fpm_downloader` module provides a simple abstraction for downloading files, uploading data, and unpacking archives, primarily for interacting with an fpm package registry. It defines a `downloader_t` type, which encapsulates procedures that wrap external command-line tools like `curl`, `wget`, and `tar`. This module is essential for fetching package information and tarballs from a registry and for publishing new packages to a registry.

# Key Components
- **`downloader_t`**: A derived type that groups file transfer and unpacking operations.
  - **`get_pkg_data(url, version, tmp_pkg_file, json, error)`**:
    - A `nopass` procedure (meaning it doesn't take `downloader_t` as an argument, effectively a static method).
    - Constructs a URL based on the base `url` and an optional `version`.
    - Calls `get_file` to download package metadata (expected to be JSON) into `tmp_pkg_file`.
    - Parses the downloaded file into a `json_object` using `jonquil:json_load`.
  - **`get_file(url, tmp_pkg_file, error)`**:
    - A `nopass` procedure.
    - Downloads a file from the given `url` to `tmp_pkg_file`.
    - It first checks if `curl` is available using `fpm_filesystem:which`. If so, it uses `curl <url> -s -o <tmp_pkg_file>`.
    - If `curl` is not found, it checks for `wget`. If found, it uses `wget <url> -q -O <tmp_pkg_file>`.
    - If neither tool is found, it returns a fatal error.
    - Returns a fatal error if the download command fails.
  - **`upload_form(endpoint, form_data, verbose, error)`**:
    - A `nopass` procedure.
    - Uploads data using an HTTP POST request with `multipart/form-data`.
    - It requires `curl` to be available.
    - Constructs the `curl` command with `-X POST -H "Content-Type: multipart/form-data"`, adding each item from `form_data` (array of `string_t`) as a form field (`-F 'field=value'`).
    - Calls `fpm_filesystem:run` to execute the command.
  - **`unpack(tmp_pkg_file, destination, error)`**:
    - A `nopass` procedure.
    - Unpacks a gzipped tarball (`.tar.gz`).
    - It requires `tar` to be available.
    - Executes `tar -zxf <tmp_pkg_file> -C <destination>`.
    - Returns a fatal error if `tar` is not found or if the command fails.

# Important Variables/Constants
N/A - The module primarily relies on external tools and their command-line arguments.

# Usage Examples
This module is mainly used by `fpm_dependency` when fetching packages from a registry and by the `fpm publish` command.

**Conceptual example of fetching package data:**
```fortran
use fpm_downloader
use fpm_versioning, only: version_t
use jonquil, only: json_object
use fpm_error, only: error_t
implicit none

character(len=100) :: registry_url = "https://my-fpm-registry.org/packages/mygroup/mypackage"
character(len=100) :: temp_file = "pkg_info.json"
type(version_t), allocatable :: specific_version
type(json_object) :: package_info
type(error_t), allocatable :: err
class(downloader_t), allocatable :: downloader_instance

! allocate(downloader_instance) ! Not strictly needed as methods are nopass

! To get latest version data:
! call downloader_instance%get_pkg_data(registry_url, specific_version, temp_file, package_info, err)
! To get specific version data (assume specific_version is allocated and set):
! call downloader_instance%get_pkg_data(registry_url, specific_version, temp_file, package_info, err)

if (allocated(err)) then
    print *, "Error getting package data: ", err%message
else
    ! Process package_info (json_object)
    print *, "Successfully fetched package data."
end if
```

**Conceptual example of unpacking a downloaded tarball:**
```fortran
! ... (after downloading mypackage.tar.gz to temp_tarball_path)
character(len=100) :: temp_tarball_path = "downloaded_package.tar.gz"
character(len=100) :: extract_to_dir = "build/dependencies/mypackage_src"

! call downloader_instance%unpack(temp_tarball_path, extract_to_dir, err)
if (allocated(err)) then
    print *, "Error unpacking: ", err%message
else
    print *, "Package unpacked successfully."
end if
```

# Dependencies and Interactions
- **`fpm_error`**: For `error_t` and `fatal_error`.
- **`fpm_filesystem`**: For `which` (to find `curl`, `wget`, `tar`) and `run` (used by `upload_form`). The intrinsic `execute_command_line` is used by `get_file` and `unpack`.
- **`fpm_versioning`**: For `version_t` type (used in `get_pkg_data`).
- **`jonquil`**: For JSON parsing (`json_object`, `json_value`, `json_error`, `json_load`, `cast_to_object`).
- **`fpm_strings`**: For `string_t`.
- **External Command-Line Tools**:
  - `curl` or `wget`: Required for downloading files (`get_file`, `get_pkg_data`).
  - `curl`: Required for uploading form data (`upload_form`).
  - `tar`: Required for unpacking gzipped tarballs (`unpack`).
  The module's functionality is entirely dependent on these external tools being installed and accessible in the system's PATH.
- **`fpm_dependency`**: This is a primary consumer of `downloader_t`'s capabilities when resolving dependencies from a remote registry.
- **`fpm publish` command**: Would use `upload_form` to send package data to a registry.
The `downloader_t` type acts as a simple wrapper to consolidate these external tool interactions, making them mockable for testing purposes.
