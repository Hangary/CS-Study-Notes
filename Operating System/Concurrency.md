# Concurrency

Keywords:
- Critical Section
- Condition Variable
- Semaphore
- Mutex

Terminology: 
- ***Parallelism***: When at least two tasks are executing *simultaneously* (at the same time).
- ***Concurrency***: When at least two threads are making progress.
  - More general than parallelism, as it allows for time-slicing.
  - Parallelism is one special condition of concurrency.
- ***Atomics***: CPU instructions that cannot be interrupted or they will fail.
- ***Transaction***: Procedures that, like atomic instructions, either execute or fail.
- ***Serializable***: When a set of operations has the same outcome as running the operations serially (no concurrency).
- ***Linearizable***: When a set of operations each have the same outcome as if they were completed in some order $σ$.
- Strong Serializability:
  - When a transaction is both Serializable and Linearizable.
  - Hardest to obtain.

## Processes

A process is an instance of a computer program that may be running. 
- Processes have many resources at their disposal. 
- At the start of each program, a program gets one process, but each program can make more processes. 
- Each process would get an exclusive virtual memory. The virtual address in the virtual memory would be mapped to a corresponding physical address.

A program consists of the following:
- A binary format: This tells the operating system about the various sections of bits in the binary - which parts are executable, which parts are constants, which libraries to include etc.
- A set of machine instructions
- A number denoting which instruction to start from
- Constants
- Libraries to link and where to fill in the address of those libraries

### Memory Layout

When a process starts, it gets its own address space, containing:

- A **Stack**. The stack is the place where automatically allocated variables and function call return addresses are stored. The stack is statically allocated by default; there is only a certain amount of space to which one can write.
  - Every time a new variable is declared, the program moves the stack pointer <u>down</u> to reserve space for the variable. 
  - This segment of the stack is writable but not executable. 
  - If the stack grows too far - meaning that it either grows beyond a preset boundary or intersects the heap - the program will `stack overflow error`, most likely resulting in a `SEGFAULT`.

- A **Heap**: The heap is a contiguous, expanding region of memory. If a program wants to allocate an object whose lifetime is manually controlled or whose size cannot be determined at compile-time, it would want to create a heap variable.
  - The heap starts at the top of the text segment and grows <u>upward</u>, meaning `malloc` may push the heap boundary - called the program break - upward.
  - This area is also writable but not executable. 
  - One can run out of heap memory if the system is constrained or if a program run out of addresses, a phenomenon that is more common on a 32-bit system.
- A **Data Segment**:

### `fork ` function

#### POSIX `Fork` Details

POSIX determines the standards of fork ("Fork" [#ref-fork_2018](http://cs241.cs.illinois.edu/coursebook/Processes#ref-fork_2018)). You can read the previous citation, but do note that it can be quite verbose. Here is a summary of what is relevant:

1. Fork will return a *<u>non-negative integer</u>* on success.
2. ==A child will <u>inherit any open file descriptors</u> of the parent.==
   - That means if a parent half of the file and forks, the child will start at that offset. 
   - A read on the child's end will shift the parent's offset by the same amount. 
   - Any other flags are also carried over.
3. <u>Pending signals are not inherited</u>. This means that if a parent has a pending signal and creates a child, the child will not receive that signal unless another process signals the child.
4. The process will be created with <u>one thread</u>.
5. Since we have copy on write (COW), read-only memory addresses are shared between processes.
6. If a program sets up certain regions of memory, they can be shared between processes.
7. Signal handlers are inherited but can be changed.
8. The process' current working directory (often abbreviated to CWD) is inherited but can be changed.
9. <u>Environment variables are inherited but can be changed</u>.

**Key differences** between the parent and the child include:

- The process id returned by `getpid()`. The parent process id returned by `getppid()`.
- The parent is notified via a signal, `SIGCHLD`, when the child process finishes but not vice versa.
- The child does not inherit pending signals or timer alarms.
- The child has <u>its own set of environment variables</u>.

### `exec` function

The `exec` set of functions replaces the process image with that of the specified program. This means that any lines of code after the `exec` call are replaced with those of the executed program.

`exec` family:

1. `e` – An array of pointers to environment variables is explicitly passed to the new process image.
2. `l` – Command-line arguments are passed individually (a list) to the function.
3. `p` – Uses the PATH environment variable to find the file named in the file argument to be executed.
4. `v` – Command-line arguments are passed to the function as an array (vector) of pointers.

#### POSIX `exec` Details

POSIX details all of the semantics that exec needs to cover (“Exec” [#ref-exec_2018](http://cs241.cs.illinois.edu/coursebook/Processes#ref-exec_2018)). Note the following

1. <u>File descriptors are preserved</u> after an exec. That means if a program open a file and doesn't to close it, it remains open in the child. This is a problem because usually the child doesn't know about those file descriptors. Nevertheless, they take up a slot in the file descriptor table and could possibly prevent other processes from accessing the file. The one exception to this is if the file descriptor has the Close-On-Exec flag set (O_CLOEXEC) - we will go over setting flags later.
2. Various signal semantics. The executed processes preserve the signal mask and the pending signal set but does not preserve the signal handlers since it is a different program.
3. <u>Environment variables are preserved</u> unless using an environ version of exec
4. The operating system may open up 0, 1, 2 - stdin, stdout, stderr, if they are closed after exec, most of the time they leave them closed.
5. The executed process <u>runs as the same PID</u> and has <u>the same parent and process group</u> as the previous process.
6. The executed process is run on <u>the same user and group</u> with <u>the same working directory</u>.

### The fork-exec-wait Pattern

A common programming pattern is to call `fork` followed by `exec` and `wait`. The original process calls fork, which creates a child process. The child process then uses exec to start the execution of a new program. Meanwhile, the parent uses `wait` (or `waitpid`) to wait for the child process to finish.

![fork_exec_wait](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201207131349.png)

Why not execute ls directly? The reason is that now we have a monitor program - our parent that can do other things. It can proceed and execute another function, or it can also modify the state of the system or read the output of the function call.

### PROCESS STATE CODES
Here are the different values that the s, stat and state output specifiers (header "STAT" or "S") will display to describe the state of a process.

CODE | Meaning
--- | ---
D | Uninterruptible sleep (usually IO)
R | Running or runnable (on run queue)
S | Interruptible sleep (waiting for an event to complete)
T | Stopped, either by a job control signal or because it is being traced.
W | paging (not valid since the 2.6.xx kernel)
X | dead (should never be seen)
Z | Defunct ("zombie") process, terminated but not reaped by its parent.

For BSD formats and when the stat keyword is used, additional characters may be displayed:

CODE | Meaning
--- | ---
< | high-priority (not nice to other users)
N | low-priority (nice to other users)
L | has pages locked into memory (for real-time and custom IO)
s | is a session leader
l | is multi-threaded (using CLONE_THREAD, like NPTL pthreads do)
+ | is in the foreground process group

## Threads

A thread is short for 'thread-of-execution'. It represents the sequence of instructions that the CPU has and will execute. 

Threads are like lightweight processes except there is **no copying** meaning no copy on write. The overhead for creating threads are a lot less than processes.
- Threads live in a process.
- Threads can be considered special processes which share the same address space but have its own stack with other processes.

Under POSIX, we can use system call `vlonr` to create a thread.

### Processes vs threads

Creating separate processes is useful because
- more security 
    - For example, Chrome browser uses different processes for different tabs.
- When running an existing and complete program then a new process is required
    - for example starting 'gcc'.
- When you are running into synchronization primitives and each process is operating on something in the system.
- too many threads are bad - the kernel tries to schedule all the threads near each other which could cause more harm than good.
- no need to worry about race conditions
- If one thread blocks in a task (say IO) then all threads block. Processes don't have that same restriction.
- When the amount of communication is minimal enough that simple IPC needs to be used.

Creating threads is more useful because:
- leverage the power of a multi-core system to do one task
- less overhead than processes
- simplified communication between the threads than processes
- threads can be part of the same process

Note that: ==threads are just special processes with shared data structures!==
- Both threads and processes are created with system call `clone()` (`fork()` is just an wrapper of `clone()`)

### Thread Internals

The `pthread` library allocates some stack space and uses the `clone` function call to start the thread at that stack address.

In a multi-threaded program, there are **multiple stacks** but only one address space. The threads all live inside the **same virtual memory** because they are part of the same process. Thus they share the heap, the global variables, and the program code.

![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201207131011.png)

Thus, a program can have multiple CPUs working on the same program at the same time and inside the same process. It’s up to the operating system to assign the threads to CPUs. If a program has more active threads than CPUs, the kernel will assign the thread to a CPU for a short duration or until it runs out of things to do and then will automatically switch the CPU to work on another thread.

### pthread usage

If a program need more threads, it can call `pthread_create` to create a new thread using the `pthread` library. You’ll need to pass a pointer to a function so that the thread knows where to start.

```C
int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine) (void *), void *arg);
```
- The first is a pointer to a variable that will hold the id of the newly created thread.
- The second is a pointer to attributes that we can use to tweak and tune some of the advanced features of pthreads.
- The third is a pointer to a function that we want to run
- Fourth is a pointer that will be given to our function

#### Pthread Functions

Here are some common pthread functions.
- `pthread_cancel`: stops a thread. 
    - Note the thread may still continue. For example, it can be terminated when the thread makes an operating system call (e.g. write). 
    - In practice, pthread_cancel is **rarely used** because a thread won't clean up open resources like files. An alternative implementation is to use a boolean (int) variable whose value is used to inform other threads that they should finish and clean up.
- `pthread_exit(void *)`: stops the calling thread meaning the thread never returns after calling `pthread_exit`. 
    - The pthread library will automatically finish the process if no other threads are running. 
    - `pthread_exit(...)` is equivalent to returning from the thread's function; both finish the thread and also set the return value (void *pointer) for the thread. 
    - ==Calling `pthread_exit` in the main thread is a common way for simple programs to ensure that all threads finish.== 
        - On the other hand, `exit()` exits the entire process and sets the process' exit value. This is equivalent to `return ();` in the main method. All threads inside the process are stopped. 
    - Note the `pthread_exit` version creates thread zombies; however, this is not a long-running process, so we don't care.
- `pthread_join()`: waits for a thread to finish and records its return value. 
    - Note the finished threads will continue to consume resources. Eventually, if enough threads are created, `pthread_create` will fail. 
    - In practice, this is only an issue for long-running processes but is not an issue for simple, short-lived processes as all thread resources are automatically freed when the process exits. 
    - This is equivalent to turning your children into zombies, so keep this in mind for long-running processes. In the exit example, we could also wait on all the threads.

### Exit Threads

There are many ways to exit threads:
- Returning from the thread function
- Calling `pthread_exit`
- Canceling the thread with `pthread_cancel`
- Terminating the process through a signal
- calling `exit()` or `abort()`
- Returning from `main`
- Executing another program
- Some undefined behavior can terminate your threads, it is undefined behavior.

Four ways to terminate a thread:
1. pexit
2. return from start function
3. exit from any thread
4. return from main
5. signal terminates process


### Fork in a thread

A program can fork inside a process with multiple threads! However, the child process only has **a single thread**, which is a clone of the thread that called fork.

In practice, creating threads before forking can lead to unexpected errors because the other threads are immediately terminated when forking. Another thread might have locked a mutex like by calling malloc and never unlock it again. 

## Concurrent conceptions

### Race Conditions (竞争条件)

**Race conditions** are whenever the outcome of a program is determined by its sequence of events determined by the processor. This means that the execution of the code is non-deterministic. Meaning that the same program can run multiple times and depending on how the kernel schedules the threads could produce inaccurate results. 

### Critical Section (临界区)

Only one thread should be in the critical section at one time.

Critical sections should be minimized to improve performance.

### Atomic Operation

An operation (or set of operations) is atomic or uninterruptible if it appears to the rest of the system to occur instantaneously. 
- Without locks, only simple CPU instructions ("read this byte from memory") are atomic (indivisible). 
- In practice, atomicity is achieved by using ***synchronization primitives***, typically a mutex lock.

## Synchronization Primitive

### Mutex Locks

Mutex = Mutual Exclusive

Mutex locks are a way to implement critical section.

Mutex functions:
```C
pthread_mutex_init
pthread_mutex_lock
pthread_mutex_unlock
pthread_mutex_destroy
```

Mutex template:

Global variable mutex:
```C
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER; // global variable
pthread_mutex_lock(&m);   // start of Critical Section
// Critical Section
pthread_mutex_unlock(&m); // end of Critical Section
pthread_mutex_destroy(&m);
```

Equivalent way:
```C
pthread_mutex_t *lock = malloc(sizeof(pthread_mutex_t)); 
pthread_mutex_init(lock, NULL);
// later 
// we must have equal number of unlocks and locks in an execution
pthread_mutex_destroy(lock);
free(lock);
```

If a mutex is already locked, the other thread trying to lock this mutex would be blocked until the mutex is unlocked.

==Another important usage of mutext locks: keep cache consistent with the memory==



### Condition Variables

> Condition Variable 相较于 lock 的优势：
> 1. 可以建立在条件判断上，当某一条件成立时，线程进入 CS。
> 2. 可以允许多个线程进入 CS。
> 2. 相较于基于 lock 和 loop 实现的 naive 版本，不需要 busy wait。

> Mutexes are good for allowing or blocking access to a critical region. In addition to mutexes, Pthreads offers a second synchronization mechanism: condition variables.  Almost always the two methods are used together.

Synchronization mechanisms need more than just mutual exclusion; also need a way to ==wait for another thread to do somethin==g (e.g., wait for a character to be added to the buffer).

Condition variables: used to wait for a particular condition to become true (e.g. characters in buffer).
- wait(condition, lock): release lock, put thread to sleep until condition is signaled; when thread wakes up again, re-acquire lock before returning.
- signal(condition, lock): if any threads are waiting on condition, wake up one of them. Caller must hold lock, which must be the same as the lock used in the wait call.
- broadcast(condition, lock): same as signal, except wake up all waiting threads.
- Note: after signal, signaling thread keeps lock, waking thread goes on the queue waiting for the lock.
- Warning: when a thread wakes up after cond_wait there is no guarantee that the desired condition still exists: another thread might have snuck in.

Condition variables are synchronization primitives that enable threads to *wait until a particular condition occurs*.
- Condition variables enable threads to atomically release a lock and enter the sleeping state.
- Condition variables allow threads to block due to some condition not being met.


Condition variables allow a set of threads to sleep until tickled! You can tickle one thread or all threads that are sleeping. If you only wake one thread then the operating system will decide which thread to wake up. You don't wake threads directly instead you 'signal' the condition variable, which then will wake up one (or all) threads that are sleeping inside the condition variable.
- Condition variables are used with a mutex and with a loop (to check a condition).
- Threads sleeping inside a condition variable are woken up by calling `pthread_cond_broadcast` (wake up all) or `pthread_cond_signal` (wake up one). 
    - Note despite the function name, this has nothing to do with POSIX signals!

The call pthread_cond_wait performs three actions:
1. unlock the mutex
2. waits (sleeps until pthread_cond_signal is called on the same condition variable). It does 1 and 2 atomically.
3. Before returning, locks the mutex

```C
pthread_cond_t cv = PTHREAD_COND_INITIALIZER;

pthread_cond_wait(&cv, &mutex) // unlock mutex; sleep; relock mutex after being waken up
pthread_cond_signal() // signal to wake up a random thread
pthread_cond_broadcast()
```

#### spurious wake

Occasionally a waiting thread may appear to wake up for no reason (this is called a spurious wake)! This is not an issue because you always use wait inside a loop that tests a condition that must be true to continue.

**Why do spurious wakes exist?**
For performance. On multi-CPU systems it is possible that a race-condition could cause a wake-up (signal) request to be unnoticed. The kernel may not detect this lost wake-up call but can detect when it might occur. To avoid the potential lost signal the thread is woken up so that the program code can test the condition again.

#### Condition Variable and Mutex

==Condition variables are ***always*** used with a mutex lock.==

Why do Condition Variables also need a mutex?

1. a lock prevents an early wakeup message (signal or broadcast functions) from being 'lost.' 
    - In a case where the signal is sent _before_ the wait, the signal would be lost.
    - If both threads had locked a mutex, the signal can not be sent until after `pthread_cond_wait(cv, m)` is called (which then internally unlocks the mutex)
2. updating the program state (answer variable) typically requires mutual exclusion.
3. to satisfy real-time scheduling concerns which we only outline here: In a time-critical application, the waiting thread with the highest priority should be allowed to continue first. To satisfy this requirement the mutex must also be locked before calling `pthread_cond_signal` or `pthread_cond_broadcast`.

#### Template

```C
pthread_cond_t cv;
pthread_mutex_t m;
int count;

// Initialize
pthread_cond_init(&cv, NULL);
pthread_mutex_init(&m, NULL);
count = 0;

pthread_mutex_lock(&m);
while (count < 10) {
    pthread_cond_wait(&cv, &m); 
    /* Remember that cond_wait unlocks the mutex before blocking (waiting)! */
    /* After unlocking, other threads can claim the mutex. */
    /* When this thread is later woken it will */
    /* re-lock the mutex before returning */
}
pthread_mutex_unlock(&m);

// later clean up with pthread_cond_destroy(&cv); and mutex_destroy 


// In another thread increment count:
while (1) {
    pthread_mutex_lock(&m);
    count++;
    pthread_cond_signal(&cv);
    /* Even though the other thread is woken up it cannot not return */
    /* from pthread_cond_wait until we have unlocked the mutex. This is */
    /* a good thing! In fact, it is usually the best practice to call */
    /* cond_signal or cond_broadcast before unlocking the mutex */
    pthread_mutex_unlock(&m);
}
```

### Counting Semaphore (计数信号量 / 旗语)

With mutex and condition variable, semaphore can be made.

> Semaphore 是一个特化的整数信号量。由于使用了 conditional variable，所以可以控制进入 CS 的线程数量，同时无需 busy waiting。

> Mutex是一把钥匙，一个人拿了就可进入一个房间，出来的时候把钥匙交给队列的第一个。一般的用法是用于串行化对critical section代码的访问，保证这段代码不会被并行的运行。
> 
> Semaphore是一件可以容纳N人的房间，如果人不满就可以进去，如果人满了，就要等待有人出来。对于N=1的情况，称为binary semaphore。一般的用法是，用于限制对于某一资源的同时访问。

A semaphore restricts the number of simultaneous users of a shared resource up to a maximum number. Threads can request access to the resource (decrementing the semaphore), and can signal that they have finished using the resource (incrementing the semaphore).

所以，mutex就是一个binary semaphore （值就是0或者1）。但是他们的区别又在哪里呢？主要有两个方面：
- 初始状态不一样：mutex的初始值是1（表示锁available），而semaphore的初始值是0（表示unsignaled的状态）。随后的操 作基本一样。mutex_lock和sem_post都把值从0变成1，mutex_unlock和sem_wait都把值从1变成0（如果值是零就等 待）。初始值决定了：虽然mutex_lock和sem_wait都是执行V操作，但是sem_wait将立刻将当前线程block住，直到有其他线程 post；mutex_lock在初始状态下是可以进入的。
- 用法不一样（对称 vs. 非对称）：这里说的是"用法"。Semaphore实现了signal，但是mutex也有signal（当一个线程lock后另外一个线程 unlock，lock住的线程将收到这个signal继续运行）。在mutex的使用中，模型是对称的。unlock的线程也要先lock。而 semaphore则是非对称的模型，对于一个semaphore，只有一方post，另外一方只wait。就拿上面的厕所理论来说，mutex是一个钥 匙不断重复的使用，传递在各个线程之间，而semaphore择是一方不断的制造钥匙，而供另外一方使用（另外一方不用归还）。
- Mutex 相比信号量增加了**所有权**的概念，一只锁住的 Mutex 只能由给它上锁的线程解开，只有系铃人才能解铃。Mutex 的功能也就因而限制在了构造临界区上。

A counting semaphore contains a non negative value and supports two operations "wait" and "post". Post increments the semaphore and immediately returns. "wait" will wait if the count is zero. If the count is non-zero the wait call decrements the count and immediately returns.

Semaphore functions:
- `sem_wait()`: wait for the counter is over zero, then decrease the count and immediately return
- `sem_post()`: increase the counter

An analogy is that some people want to eat cookies in a cookie jar. `post` is to add one cookie into the jar, while `wait` is to take one cookie from the jar, if there is none, then wait.

```
task_t *buffer[10];

sem_t numspaces, numitems;

init() {

```

Template:
```C
#include <semaphore.h>

sem_t s;
int main() {
    sem_init(&s, 0, 10); // max cookies are ten // returns -1 (=FAILED) on OS X 
    sem_wait(&s); // Could do this 10 times without blocking
    sem_post(&s); // Announce that we've finished (and one more resource item is available; increment count)
    sem_destroy(&s); // release resources of the semaphore
}
```

A semaphore with a count of one can be used to replace a mutex.
- `sem_wait` is similar to `lock`
- `sem_post` is similar to `unlock`
- however, the overhead of a semaphore is greater than a mutex.

## Producer and Consumer

## Reader Writer Problem

A typical Reader Writer Problem:
- No write and read at the same time

## Deadlock

***Deadlock*** is defined as when a system cannot make any forward progress. 

## Concurrency Freedoms

Freedoms:
- **Wait-Free**: each thread moves forward regardless of whatever is happening elsewhere.
- **Lock-Free**: the system as whole moves forward regardless of any internal state.
- **Obstruction-Free**: threads make progress only if it does not encounter contention from other threads (live lock can happen)

Types of Concurrent Data Structures:
- Blocking Data Structures
- Lock-Free Data Structures
- Bounded-Wait Data Structures
- Wait-Free Data Structures

### Lock-Free Data Structures

Ways to implement Lock-Free Data Structures:
- **Ownership**
- **Transactions**
