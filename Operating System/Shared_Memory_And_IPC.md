# Shared Memory and IPC

IPC: InterProcess Communication
- pipes
- networking

Shared memory is the easiest way to implement an interprocess communication. 

## `mmap`

> In computing, `mmap` is a POSIX-compliant Unix system call that maps files or devices into memory. 
>  It is a method of memory-mapped file I/O. It implements demand paging, because file contents are not read from disk directly and initially do not use physical RAM at all. The actual reads from disk are performed in a "**lazy**" manner, after a specific location is accessed. 

## Pipe

Pipe a stream for communicating between two processes.

A simple example:
```C
// create pipes
int fds[2];
pipe(fds);

// fork
pid_t child = fork();
if (child == 0) {
    write(fds[1], "Hello", 5);
} else {
    char *buffer[100];
    ssize_t num_read = read(fds[0], buffer, sizeof(buffer)-1); // can be blocked
}
    return 0;
```


We cab wrap the pipe into a file handler:
```C
// create pipes
int fds[2]; // two file descriptors are needed 
pipe(fds);

// we can wrap the pipe into a file handler
// so that we can write more interesting data into it
FILE *easy = fdopen(fds[1], "w");

/ pre-work
fflush(easy);
fflush(stdin);
fflush(stdout);

// fork
pid_t child = fork();
if (child == 0) {
    fprintf(easy, "%g\n", 3.14159);
    fflush(easy);
} else {
    char *buffer[100];
    ssize_t num_read = read(fds[0], buffer, sizeof(buffer)-1); // can be blocked
}
    return 0;
```

return values of `read` and `write` a pipe:
- read: `0` if the write end of the pipe is closed
- write: if the read end of the pipe is closed
    - `-1` is returned
    - A `SIGPIPE` signal would also arise!
- Remember to close unused ends, otherwise, it might block the pipe

### `fseek` and `ftell`

`fseek`: read a file from an offse t
`ftell`: current position

```C
FILE *file = fopen("data", "r");
fseek(file, SEEK_END);
long position = ftell(file);
fseek(file, 0, SEEK_SET);
```