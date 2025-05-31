# Overview
The `fpm_backend_output` module defines the `build_progress_t` derived type, which is used to display build status and progress messages to the console during the fpm package build process. It offers two output modes: a "normal" mode with "pretty" formatted output (colors, line updates) for interactive terminal use, and a "plain" mode for non-interactive scenarios like verbose logging or when `stdout` is redirected. The plain mode avoids ANSI escape codes. This module relies on `fpm_backend_console` for the low-level console manipulations.

# Key Components
- **`build_progress_t`**: A derived type to manage and display build progress.
  - `console`: A `console_t` object (from `fpm_backend_console`) for console interactions.
  - `n_complete`: Integer, count of completed build targets.
  - `n_target`: Integer, total number of scheduled build targets.
  - `plain_mode`: Logical, if true, use simple text output; otherwise, use formatted output with colors and line updates.
  - `output_lines`: Allocatable integer array, stores line numbers for updating console lines in pretty mode.
  - `target_queue`: Pointer to an array of `build_target_ptr`, representing the scheduled build targets.
  - `compile_commands`: A `compile_command_table_t` object to store compilation commands, which can be written to `compile_commands.json`.
- **`new_build_progress(target_queue, plain_mode)`**: Constructor function for `build_progress_t`. Initializes the progress object with the target queue and output mode.
- **`output_status_compiling(progress, queue_index)`**: Procedure bound to `build_progress_t` (as `compiling_status`). Outputs a message indicating that a specific target is currently being compiled, along with overall progress percentage.
- **`output_status_complete(progress, queue_index, build_stat)`**: Procedure bound to `build_progress_t` (as `completed_status`). Outputs a message indicating that a target has finished compiling (either successfully or with failure), and updates the overall progress.
- **`output_progress_success(progress)`**: Procedure bound to `build_progress_t` (as `success`). Outputs a final message indicating the successful compilation of the entire project.
- **`output_write_compile_commands(progress, error)`**: Procedure bound to `build_progress_t` (as `dump_commands`). Writes the accumulated compilation commands to a `compile_commands.json` file in the `build/` directory.

# Important Variables/Constants
- `plain_mode` (in `build_progress_t`): Controls the output style. Crucial for determining whether to use ANSI escape codes (pretty output) or simple text (plain output).
- `LINE_RESET`, `COLOR_RED`, `COLOR_GREEN`, `COLOR_YELLOW`, `COLOR_RESET`: Constants imported from `fpm_backend_console` used for styling the "pretty" output.

# Usage Examples
This module is primarily used internally by the `fpm_backend` module during the `build_package` subroutine.

```fortran
! Conceptual example within fpm_backend
use fpm_backend_output
use fpm_targets
! ... other necessary modules ...

type(build_target_ptr), allocatable :: queue(:)
logical :: be_verbose, is_tty
type(build_progress_t) :: progress
integer :: i, stat

! ... (queue is populated and sorted) ...

is_tty = ... ! (determine if output is a TTY)
be_verbose = ... ! (check for verbose flag)

progress = new_build_progress(queue, plain_mode=(be_verbose .or. .not. is_tty))

do i = 1, size(queue)
    call progress%compiling_status(i)
    ! ... (call actual compile/link for queue(i)%ptr, get status in stat) ...
    call progress%completed_status(i, stat)
    if (stat /= 0) then
        ! ... (handle error) ...
    end if
end do

if (all_successful) then
    call progress%success()
end if
call progress%dump_commands(error_obj)
! ... (handle potential error from dump_commands) ...
```

# Dependencies and Interactions
- **`iso_fortran_env`**: Used for `stdout`.
- **`fpm_error`**: For `error_t` type used in `dump_commands`.
- **`fpm_filesystem`**: For `basename` and `join_path` (used for constructing file paths and names for output).
- **`fpm_targets`**: For `build_target_ptr` type, which represents a build target.
- **`fpm_backend_console`**: This is a critical dependency. `fpm_backend_output` uses `console_t` and its associated procedures (and constants like `LINE_RESET`, color codes) from `fpm_backend_console` to perform the actual writing and updating of console lines for the "pretty" output mode.
- **`fpm_compile_commands`**: For `compile_command_t` and `compile_command_table_t` to manage and write the `compile_commands.json` file.
- **`fpm_backend`**: The `fpm_backend` module is the primary user of `fpm_backend_output`. It creates and updates the `build_progress_t` object throughout the build lifecycle.
- **OpenMP**: Uses `!$omp critical` directives around some console output operations in `plain_mode` to ensure thread safety, though the primary mechanism for complex console updates (`console_t`) is expected to be managed for thread safety by `fpm_backend_console`.
