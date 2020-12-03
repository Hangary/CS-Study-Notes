# POSIX: Error Handling

POSIX defines a special global integer `errno` that is set when a system call fails. 
- The initial value of `errno` is zero (i.e. no error).
- When a system call fails, it will typically 
  - return -1 to indicate an error
  - set `errno`
- `errno` would not be reset back to 0 automatically. If a system call is successful, it would not change `errno`.
- `strerror(errno)` can return a short description of the error value.

For multiple threads, each thread has its own copy of `errno` to avoid intfering with each others.

## Good practices to use `errno` 

Since every system call might change `errno` even `put`, a good practice is to copy the `errno` when we want to handle it:
```C
// Unsafe - the first fprintf may change the value of errno before we use it!
if (-1 == sem_wait(&s)) {
   fprintf(stderr, "An error occurred!");
   fprintf(stderr, "The error value is %d\n", errno);
}
// Better, copy the value before making more system and library calls
if (-1 == sem_wait(&s)) {
   int errno_saved = errno;
   fprintf(stderr, "An error occurred!");
   fprintf(stderr, "The error value is %d\n", errno_saved);
}
```

Similarly, a signal hanlder would better save the `errno` if it would make system calls:
```C
void handler(int signal) {
   int errno_saved = errno;

   // make system calls that might change errno

   errno = errno_saved;
}
```

## `EINTR`

Some system calls can be interrupted when a signal (e.g SIGCHLD, SIGPIPE,...) is delivered to the process. This interruption can be detected by checking the return value and if `errno` is `EINTR`. 

A common pattern to avoid `EINTR` is:
```C
while ((-1 == (result = systemcall(...))) && (errno == EINTR)) {
    /* repeat! */
}
```

### Which system calls may be interrupted and need to be wrapped?

On Linux, calling `read` and `write` to a local disk will normally not return with `EINTR` (instead the function is automatically restarted for you). However, calling `read` and `write` on a file descriptor that corresponds to a network stream can return with `EINTR`.

We can use the man page to check what kinds of errors a system call might return. The man page includes a list of errors (i.e. errno values) that may be set by the system call. 

==A rule of thumb is 'slow' (blocking) calls (e.g. writing to a socket) may be interrupted but fast non-blocking calls (e.g. `pthread_mutex_lock`) will not.==

If a signal handler is invoked while a system call or library function call is blocked, then either:
- the call is automatically restarted after the signal handler returns; or
- the call fails with the error `EINTR`. Which of these two behaviors occurs depends on the interface and whether or not the signal handler was established using the `SA_RESTART` flag. 

The details vary across UNIX systems; below, the details for Linux:
If a blocked call to one of the following interfaces is interrupted by a signal handler, then the call will be automatically restarted after the signal handler returns if the `SA_RESTART` flag was used; otherwise the call will fail with the error `EINTR`:
- read(2), readv(2), write(2), writev(2), and ioctl(2) calls on "slow" devices.
- A "slow" device is one where the I/O call may block for an indefinite time, for example, a terminal, pipe, or socket. (A disk is not a slow device according to this definition.) 
- If an I/O call on a slow device has already transferred some data by the time it is interrupted by a signal handler, then the call will return a success status (normally, the number of bytes transferred). "

Note, it is easy to believe that setting 'SA_RESTART' flag is sufficient to make this whole problem disappear. Unfortunately that's not true: there are still system calls that may return early and set EINTR! See signal(7) for details.