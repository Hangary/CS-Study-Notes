# Kernal

What is kernal:
- Operates in '*kernel space*' (Where kernel code runs in ring 0 access to full CPU instruction set).
  - Kernel space is lower level - you don't get the nice abstractions like `sbrk(2)` or `write(2)`. The kernel has to route each of the requests to the appropriate drivers or handlers.
- The kernel can do anything and touch all memory (e.g. accessing devices and IO). Need a layer of security preventing unwated changes in the state of the system, especially in undened ways. 
- Exposed kernel calls are called **system calls**.

Some kernal calls:
- `void * kmalloc(size_t size, int ags)`
- `mmap(void *addr, size_t length, int prot, int ags, int fd, off_t offset)`

Booting process: BIOS loads up -> MBR loads up -> GRUB loads up -> Then starts the Kernel
- The first thing the kernel does is start `init` (the main process for your operating system).
  - `init` does a lot of things. One important thing it does is initializing `fork()`, the magical library call that starts the entire process.
  - Then `init` checks run level and runs the appropriate startup scripts.

## System Calls

A **system call** is a call a program makes in user space that gets executed by the kernel.

Process of a system call:
1. When calling system calls, a software *interrupt* is used to give control back to the kernal.
2. Kernel traps the signal, routes to the appropriate system call and executes the system call in Kernel space.
3. Kernal would read parameters and store return value in the registers.

![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201202165914.png)

## Linking and Loading


There are two types of libraries, those *compiled* with your programs and those that are *linked dynamically* at runtime. 

There are many benefits to use programs that get compiled with your program, but some drawbacks.

Benefits:
- You have the source code/debug checking
- All code is in your code segment
- You can modify the library

Drawbacks:
- Updating is often tedious
- Your executable is bigger
- Your library cannot be reused by other application

What if we have one library that a bunch of programs can use (make it read only) and have it dynamically link the function calls in the program?
The solution is using **Dynamic Libraries**.

How Dynamic Libraries are implemented?
- Have the functions in your code point to pointer where the functions are going to be.
- Have your code jump to the pointer and the pointer jump to the actual function.
- This is also called a **Dynamic Lookup Table**.

Process:
1. Exec generates lookup-table (`PT_INTERP` segment)
  - During exec, the function checks what libraries you need.
  - If that library is already loaded into memory, increase the reference count and link the functions.
  - Else, `mmap` the library into memory. Set the executable bit (memory can either be executable or writeable) and then link the function. Cache this library's location.
  - When a process is done, reduce the reference count and return the page back to the system if need be.
2. Calls to a library will cause a jump to the lookup table
3. Lookup table entry will redirect the program to the library function
4. After execution, returns back to your program

![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201202170913.png)

## Process

Process creation:
![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201202171338.png)

## Thread

Two types of threads - user space threads and kernel threads:
- User-space threads are POSIX Portable Threads `pthreads`
- Kernel threads are used for things like system calls and monitoring. No need to context switch to user space!
  - To the kernel, there are no things as threads, and *everything is a process*.
  - A thread is just a process that happens to share resources (such as memory, signal disposition, open les etc.) with the original process.

Connections between threads and process:
![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201202171805.png)

`pthread_create`:
1. After pthread initialization, set the attributes like detached state, stack address, and scheduling preferences.
2. Then, allocate some stack space in the current program's stack. Then the thread gets added to a table and a lightweight process is created using clone() [Think fork but no copying]
3. Then the pthread is added to the pthread table, and returns out of the pthread function.
4. Scheduling the process is left up to the completely fair scheduler.

![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201202172251.png)

## Scheduling

Two kinds of processes:
- IO-intensive process
  - Important, like a web browser
  - if starved, computer would be very slow
- Computing-intensive process

We can assign a percentage of the CPU to a
process.
- This way when an IO-bound process sleeps, it has a high priority when it wakes up.
- The CPU creates a Red-Black tree with the processes virtual runtime (runtime / nice_value) and sleeper fairness (if the process is waiting on something give it the CPU when it is done waiting). 
- The pop operation is guaranteed $O(\log(n))$

**Nice values** are the kernel's way of yielding resources. A higher nice value means that you are "nicer" and give up priority.

Since threads are light-weight processes then they get scheduled like any other process and usually they get scheduled at the same time as your first process which usually results in race conditions for improper code.
- The CFS tends to schedule groups of processes together - taking advantage of cache coherency, open les, open sockets etc.
- The CFS handles higher priority and long running processes fairly so no process fades away into the scheduling abyss.