        +--------------------------+
        |          CS 2043         |
        | PROJECT 2: USER PROGRAMS |
        |    DESIGN DOCUMENT       |
        +--------------------------+

---- PRELIMINARIES ----
>> By: Anusan Krishnathas (230048J)

>> If you have any preliminary comments on your submission, notes for the
>> TAs, or extra credit, please give them here.

>> Please cite any offline or online sources you consulted while
>> preparing your submission, other than the Pintos documentation, course
>> text, lecture notes, and course staff.

ARGUMENT PASSING
================

---- DATA STRUCTURES ----

>> A1: Copy here the declaration of each new or changed `struct` or
>> `struct` member, global or static variable, `typedef`, or
>> enumeration. Identify the purpose of each in 25 words or less.

`#define DEFAULT_ARG_SIZE 3` — initial allocation size for argv pointer array before possible growth.

`static bool setup_stack(void **esp, const char *commandLineArguments)` — builds the initial user stack, tokenising the command line and pushing argv/argc metadata.

---- ALGORITHMS ----

>> A2: Briefly describe how you implemented argument parsing. How do
>> you arrange for the elements of argv[] to be in the right order?
>> How do you avoid overflowing the stack page?

`process_execute` copies the raw command line into a page buffer that is handed to `start_process` and, eventually, to `load`. Inside `setup_stack` we duplicate the string, split it with `strtok_r`, and keep the tokens in a dynamically grown array. We then walk that array backward so each argument string is copied onto the stack from high to low addresses, recording the address of every copy. After enforcing word alignment and pushing a sentinel `NULL`, we push the saved addresses, the `argv` base, `argc`, and a fake return address. The backward walk preserves argv order because the last token we processed ends up deepest on the stack and the saved addresses are written in reverse. Overflow is avoided by building the stack inside a single zero-filled page mapped at `PHYS_BASE - PGSIZE`; every decrement stays within that page, so the function fails if the page cannot be mapped.

---- RATIONALE ----

>> A3: Why does Pintos implement strtok_r() but not strtok()?

`strtok_r` is re-entrant: each caller keeps its own state. That makes it safe for concurrent threads and nested tokenisation, unlike the global-state `strtok`.

>> A4: In Pintos, the kernel separates commands into an executable name
>> and arguments. In Unix-like systems, the shell does this
>> separation. Identify at least two advantages of the Unix approach.

- Shells can validate, expand, or transform arguments before the expensive `exec`, catching user errors early.
- The kernel receives a pre-parsed vector, so the loader can focus on memory management instead of string manipulation.

							 SYSTEM CALLS
							 ============

---- DATA STRUCTURES ----

>> B1: Copy here the declaration of each new or changed `struct` or
>> `struct` member, global or static variable, `typedef`, or
>> enumeration. Identify the purpose of each in 25 words or less.

`struct list child_list;` — per-thread list tracking child processes for `wait` and cleanup.

`struct list_elem child;` — list node that links a child into its parent’s `child_list`.

`struct thread *parent_t;` — pointer back to the parent thread for bookkeeping.

`struct semaphore init_sema;` — lets a parent wait for a child to finish loading.

`struct semaphore pre_exit_sema;` — blocks a dying child until the parent collects its exit status.

`struct semaphore exit_sema;` — blocks the parent after `wait` until the child can finish exiting.

`bool load_success_status;` — records whether `load` completed successfully for reporting to the parent.

`int exit_status;` — child’s exit code, retrieved by the parent during `wait`.

`int next_fd;` — monotonically increasing counter used to hand out process-local file descriptors.

`struct list open_fd_list;` — per-thread list of open descriptors for lookup and cleanup.

`struct file *process_file;` — pointer to the executable file so writes can be denied while running.

`struct lock file_system_lock;` — global lock serialising file system operations across processes.

```
struct file_descriptor {
	struct file *_file;      /* backing file pointer */
	int fd;                  /* process-local descriptor number */
	struct list_elem fd_elem;/* node stored in open_fd_list */
};
```
— wrapper that ties a Pintos `struct file` to a per-process descriptor.

---- ALGORITHMS ----

>> B2: Describe how file descriptors are associated with open files.
>> Are file descriptors unique within the entire OS or just within a
>> single process?

Each process keeps an `open_fd_list` of `struct file_descriptor` objects. A descriptor stores the next available integer `fd` (from `next_fd`) and the backing `struct file *`. Lookup walks the list to find the matching `fd`. Descriptor numbers are unique only within a process; different processes can reuse the same integers without conflict.

>> B3: Describe your code for reading and writing user data from the
>> kernel.

For reads we first validate the user buffer, then branch on the descriptor. `fd == 0` uses `input_getc` in a loop, while other descriptors resolve to a `struct file_descriptor` and call `file_read` under `file_system_lock`. Writes validate the buffer, send `fd == 1` through `putbuf`, and otherwise use `file_write` under the same lock. Both paths surface `-1` for invalid descriptors.

>> B4: Suppose a system call causes a full page (4,096 bytes) of data
>> to be copied from user space into the kernel. What is the least
>> and the greatest possible number of inspections of the page table
>> (e.g. calls to pagedir_get_page()) that might result? What about
>> for a system call that only copies 2 bytes of data? Is there room
>> for improvement in these numbers, and how much?

`validate_buffer` checks every byte: the initial `validate_ptr` plus one check per byte in the loop. Copying 4,096 bytes therefore issues 4,097 page-table lookups; copying 2 bytes issues 3. We could improve this dramatically by validating page-by-page instead of byte-by-byte, reducing the 4,096-byte case to at most two lookups (start and end) per page span.

>> B5: Briefly describe your implementation of the "wait" system call
>> and how it interacts with process termination.

`wait` simply delegates to `process_wait`. The parent searches its `child_list`; on success it removes the child node, waits on the child’s `pre_exit_sema`, reads `exit_status`, signals `exit_sema`, and returns the status. The child’s `process_exit` posts `pre_exit_sema` once it has printed and released resources, then blocks on `exit_sema` until the parent observes the exit code.

>> B6: Any access to user program memory at a user-specified address
>> can fail due to a bad pointer value. Such accesses must cause the
>> process to be terminated. System calls are fraught with such
>> accesses, e.g. a "write" system call requires reading the system
>> call number from the user stack, then each of the call's three
>> arguments, then an arbitrary amount of user memory, and any of
>> these can fail at any point. This poses a design and
>> error-handling problem: how do you best avoid obscuring the primary
>> function of code in a morass of error-handling? Furthermore, when
>> an error is detected, how do you ensure that all temporarily
>> allocated resources (locks, buffers, etc.) are freed? In a few
>> paragraphs, describe the strategy or strategies you adopted for
>> managing these issues. Give an example.

We centralised validation into helpers. `validate_ptr` performs the null, range, and `pagedir_get_page` checks in one place; `validate_string` and `validate_buffer` reuse it for strings and buffers. The syscall handlers call these helpers before touching user memory, so the main logic remains focused on the operation itself. If validation fails we call `syscall_exit(-1)`, which sets the status and tears down the thread, ensuring any held locks are released via structured control flow. For example, `SYS_WRITE` validates both the stack arguments and the data buffer before acquiring `file_system_lock`; if validation fails the process exits before the lock is taken, so no additional cleanup is required.

---- SYNCHRONIZATION ----

>> B7: The "exec" system call returns -1 if loading the new executable
>> fails, so it cannot return before the new executable has completed
>> loading. How does your code ensure this? How is the load
>> success/failure status passed back to the thread that calls "exec"?

`syscall_exec` calls `process_execute` and then waits on the child’s `init_sema`. The child posts that semaphore only after `load` finishes and it has recorded the result in `load_success_status`. When the parent wakes up it reads that flag; if loading failed it returns `-1`, otherwise it returns the child’s tid.

>> B8: Consider parent process P with child process C. How do you
>> ensure proper synchronization and avoid race conditions when P
>> calls wait(C) before C exits? After C exits? How do you ensure
>> that all resources are freed in each case? How about when P
>> terminates without waiting, before C exits? After C exits? Are
>> there any special cases?

If P calls `wait(C)` while C is running, P blocks on `pre_exit_sema` until C reaches `process_exit`; C can then finish once P signals `exit_sema`. If C finishes first, it still blocks on `exit_sema` until P arrives, so the exit status is delivered exactly once. Regardless of ordering, the child descriptor is removed from `child_list`, and `process_exit` closes every open descriptor and releases the executable file. If P terminates without waiting, its `process_exit` runs without touching the child; C still cleans up its own resources and will eventually unblock when the scheduler runs it, because the semaphores are per child and do not require the parent to remain alive beyond releasing `pre_exit_sema`.

---- RATIONALE ----

>> B9: Why did you choose to implement access to user memory from the
>> kernel in the way that you did?

The dedicated validation helpers keep syscall handlers readable while enforcing a single policy for rejecting bad pointers. Exiting through `syscall_exit` guarantees we do not return to user mode with inconsistent state.

>> B10: What advantages or disadvantages can you see to your design
>> for file descriptors?

Per-process lists make lookup simple and ensure descriptors vanish automatically on exit, but walking the list on every lookup is linear in the number of open files. An indexed structure could improve performance for workloads that open many files.

>> B11: The default tid_t to pid_t mapping is the identity mapping.
>> If you changed it, what advantages are there to your approach?

We kept the identity mapping.

SURVEY QUESTIONS
================

Answering these questions is optional, but it will help us improve the
course in future quarters. Feel free to tell us anything you want—
these questions are just to spur your thoughts. You may also choose to
respond anonymously in the course evaluations at the end of the
quarter.

>> In your opinion, was this assignment, or any one of the three problems
>> in it, too easy or too hard? Did it take too long or too little time?

It felt challenging and required significant time, but working through the hurdles was valuable practice.

>> Did you find that working on a particular part of the assignment gave
>> you greater insight into some aspect of OS design?

Understanding how argument stacks are laid out helped solidify the interaction between user space conventions and kernel loaders.

>> Is there some particular fact or hint we should give students in
>> future quarters to help them solve the problems? Conversely, did you
>> find any of our guidance to be misleading?

More incremental checkpoints or reminders about validating user memory early would be helpful.

>> Do you have any suggestions for the TAs to more effectively assist
>> students, either for future quarters or the remaining projects?

Targeted office hours focused on debugging tricky race conditions would be appreciated.

>> Any other comments?

No additional comments.


