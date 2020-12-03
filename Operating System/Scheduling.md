# Scheduling

CPU Scheduling is the problem of efficiently selecting which process to run on a system's CPU cores. In a busy system, there will be more *ready-to-run processes* than there are *CPU cores*, so the system kernel must evaluate which processes should be scheduled to run and which processes should be executed later.

## High Level Overview

Schedulers are pieces of software programs, which can also be implemented by users. If you are given a list of commands to exec, a program can schedule them them with `SIGSTOP` and `SIGCONT`. These are called ***user space schedulers***. Hadoop and python's celery may do some sort of user space scheduling or deal with the operating system.

Scheduling affects the performance of the system, specifically the *latency* and *throughput* of the system.
-  The throughput might be measured by a system value.
   -  for example, the I/O throughput - the number of bits written per second, or the number of small processes that can complete per unit time. 
-  The latency might be measured by the response time - the elapse time before a process can start to send a response, or wait time or turnaround time - the elapsed time to complete a task.

Different schedulers offer different optimization trade-offs that may be appropriate for desired use. There is no optimal scheduler for all possible environments and goals.

### Flowchart

At the operating system level, you generally have this type of flowchart:
1. **New** is the initial state. A process has been requested to schedule. All process requests come from fork or clone. At this point the operating system knows it needs to create a new process.
2. A process moves from the new state to the **ready**. This means any structs in the kernel are allocated. From there, it can go into ready suspended or running.
3. **Running** is the state that we hope most of our processes are in, meaning they are doing useful work. A process could either get preempted, blocked, or terminate. Preemption brings the process back to the ready state. If a process is blocked, that means it could be waiting on a mutex lock, or it could've called sleep - either way, it willingly gave up control.
4. On the **blocked** state the operating system can either turn the process ready or it can go into a deeper state called blocked suspended.
5. There are so-called deep slumber states called **blocked suspended** and **blocked ready**.

## Preemption

Without preemption, processes will run until they are unable to utilize the CPU any further. Once a process is scheduled it will continue, even if another process with a high priority appears on the ready queue.

With preemption, the existing processes may be removed immediately if a more preferred process is added to the ready queue. 

Any scheduler that doesn't use some form of preemption can result in *starvation*, because earlier processes may never be scheduled to run (assigned a CPU). 

## Priority

Processes are scheduled in the order of priority value. For example, a navigation process might be more important to execute than a logging process.

## Ready Qeueu

A process is placed on the ***ready queue*** when it can use a CPU. Some examples include:
- A process was blocked waiting for a read from storage or socket to complete and data is now available.
- A new process has been created and is ready to start.
- A process thread was blocked on a synchronization primitive (condition variable, semaphore, mutex lock) but is now able to continue.
- A process is blocked waiting for a system call to complete but a signal has been delivered and the signal handler needs to run.

## Definition

First some definitions:
- **start_time** is the wall-clock start time of the process (CPU starts working on it)
- **end_time** is the end wall-clock of the process (CPU finishes the process)
- **run_time** is the total amount of CPU time required
- **arrival_time** is the time the process enters the scheduler (CPU may start working on it)

Here are measures of efficiency and their mathematical equations
- **Turnaround Time** is the total time from when the process arrives to when it ends. 
  - = end_time - arrival_time
- **Response Time** is the total latency (time) that it takes from when the process arrives to when the CPU actually starts working on it.
  - = start_time - arrival_time
- **Wait Time** is the total wait time or the total time that a process is on the ready queue.
  - end_time - arrival_time - run_time
  - A common mistake is to believe it is only the initial waiting time in the ready queue. If a CPU intensive process with no I/O takes 7 minutes of CPU time to complete but required 9 minutes of wall-clock time to complete we can conclude that it was placed on the ready-queue for 2 minutes. For those 2 minutes, the process was ready to run but had no CPU assigned. It does not matter when the job was waiting, the wait time is 2 minutes. 


## Convoy Effect

The ***convoy effect*** is when a process takes up a lot of the CPU time, leaving all other processes with potentially smaller resource needs following like a Convoy Behind them.

Suppose the CPU is currently assigned to a CPU intensive task and there is a set of I/O intensive processes that are in the ready queue. These processes require a tiny amount of CPU time but they are unable to proceed because they are waiting for the CPU-intensive task to be removed from the processor. These processes are starved until the CPU bound process releases the CPU. But, the CPU will rarely be released. For example, in the case of an FCFS scheduler, we must wait until the process is blocked due to an I/O request. The I/O intensive process can now finally satisfy their CPU needs, which they can do quickly because their CPU needs are small and the CPU is assigned back to the CPU-intensive process again. Thus the I/O performance of the whole system suffers through an indirect effect of starvation of CPU needs of all processes.

This effect is usually discussed in the context of FCFS scheduler; however, a Round Robin scheduler can also exhibit the Convoy Effect for long time-quanta.

## Scheduling Algorithms

Suppose that:
- Process 1: Runtime 1000ms
- Process 2: Runtime 2000ms
- Process 3: Runtime 3000ms
- Process 4: Runtime 4000ms
- Process 5: Runtime 5000ms

### Shortest Job First (SJF)

![Shortest Job First Graph](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201127213241.png)

Arival Time:
- P1 Arrival: 0ms
- P2 Arrival: 0ms
- P3 Arrival: 0ms
- P4 Arrival: 0ms
- P5 Arrival: 0ms

The scheduler schedules the job with the **shortest total CPU time**. The critical problem is that this scheduler needs to know how long this program will run over time before it ran the program. 

**Advantages**
1. Shorter jobs tend to get run first
2. On average wait times and response times are down

**Disadvantages**
1. Needs algorithm to be omniscient
2. Need to estimate the burstiness of a process which is harder than a computer network
3. Longer jobs might suffer from starvation if there are a lot of small jobs

### Preemptive Shortest Job First (PSJF)

![Preemptive Shortest Job First](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201127213746.png)

Arrival Time:
- P2 at 0ms
- P1 at 1000ms
- P5 at 3000ms
- P4 at 4000ms
- P3 at 5000ms

Preemptive shortest job first is like shortest job first but if a new job comes in with **a shorter runtime** than the total runtime of the current job, it is run instead. 

Note: The scheduler uses the total runtime of the process. If the scheduler wants to compare the shortest remaining time left, that is a variant of PSJF called Shortest Remaining Time First (SRTF).

**Advantages**
1. Ensures shorter jobs get run first

**Disadvantages**
1. Need to know the runtime again
2. Context switching and jobs can get interrupted
3. Longer jobs might suffer from starvation if there are a lot of small jobs. This condition is even worse than SJF because in PSJF, new jobs are added continuously.

### First Come First Served (FCFS)

![First Come First Served](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201127213920.png)

Arrival Time:
- P2 at 0ms
- P1 at 1000ms
- P5 at 3000ms
- P4 at 4000ms
- P3 at 5000ms

Processes are scheduled in the order of arrival. 

**Advantages**
- Simple algorithm and implementation
- Context switches infrequent when there are long-running processes
- No starvation if all processes are guaranteed to terminate

**Disadvantages**
- Suffer from **Convoy effect**
- Context switches infrequent when there are long-running processes

### Round Robin (RR)

![Round Robin](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201127214247.png)

Arival Time:
- P1 Arrival: 0ms
- P2 Arrival: 0ms
- P3 Arrival: 0ms
- P4 Arrival: 0ms
- P5 Arrival: 0ms

Processes are scheduled in order of their arrival in the ready queue. However, after a small time step, a running process will be forcibly removed from the running state and placed back on the ready queue. 
- This ensures long-running processes refrain from starving all other processes from running. 
- The maximum amount of time that a process can execute before being returned to the ready queue is called the ***time quanta***. 
- As the time quanta approaches to infinity, Round Robin will be equivalent to FCFS.

**Advantages**:
- Ensures some notion of fairness
**Disadvantages**:
- Large number of processes = Lots of switching