# Overview
This file provides utility functions for interacting with the filesystem, such as checking if a path is a directory and reading directory contents. These functions are likely used by other parts of the fpm (Fortran Package Manager) project to handle file and directory operations in a cross-platform manner.

# Key Components
- `c_is_dir(const char *path)`: Checks if the given path corresponds to an existing directory. Returns `1` if it is a directory, `0` otherwise.
- `get_d_name(struct dirent *d)`: Returns the name of a directory entry.
- `c_opendir(const char *dirname)`: Opens a directory specified by `dirname`. Returns a `DIR` pointer.
- `c_readdir(DIR *dirp)`: Reads the next directory entry from a directory stream `dirp`. Returns a pointer to a `dirent` structure.

# Important Variables/Constants
N/A

# Usage Examples
```c
// Example (conceptual, would need to be part of a larger C program)
#include <stdio.h>
// Assuming the functions from filesystem_utilities.c are available

int main() {
    const char *path = "./some_directory";
    if (c_is_dir(path)) {
        printf("%s is a directory.\n", path);
        DIR *dir = c_opendir(path);
        if (dir) {
            struct dirent *entry;
            while ((entry = c_readdir(dir)) != NULL) {
                printf("Found file/directory: %s\n", get_d_name(entry));
            }
            closedir(dir); // Important to close the directory
        }
    } else {
        printf("%s is not a directory.\n", path);
    }
    return 0;
}
```

# Dependencies and Interactions
- This file uses standard C library headers: `sys/stat.h` (for `stat`, `S_ISDIR`) and `dirent.h` (for `opendir`, `readdir`, `DIR`, `struct dirent`).
- It includes specific handling for macOS (`__APPLE__`) on certain architectures (non-ARM64, non-PowerPC, non-i386) to use `opendir$INODE64` and `readdir$INODE64` likely for compatibility with older systems or specific filesystem features.
- These functions are C wrappers and are likely called from Fortran code within the fpm project using Fortran's C interoperability features.
