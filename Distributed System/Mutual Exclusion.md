# Mutual Exclusion

The Mutual Exclusion Problem is also called **Critical Section** Problem: 
- Critical Section is a piece of code (at all processes) for which we need to ensure there is *at most one process* executing it at any point of time.


We want to implement the three functions for mutual exclusion:
- `enter()` to enter the critical section (CS)
- `AccessResource()` to run the critical section code
- `exit()` to exit the critical section

The properties a mutual exclusion:
- **Safety** (essential): At most one process executes in CS (Critical Section) at any time.
- **Liveness** (essential): Every request for a CS is granted eventually.
- **Ordering** (desirable): Requests are granted in the order they were made.

System Model:
- Each pair of processes is connected by reliable channels (such as TCP).
- Messages are eventually delivered to recipient, and in FIFO (First In First Out) order.
- Processes do not fail.
  - Fault-tolerant variants exist in literature. 

## Leader Central Solution

Central Solution:
- Elect a central leader
  - Use one of our election algorithms!
- Master keeps
  - A queue of waiting requests from processes who wish to access the CS
  - A special token which allows its holder to access CS
- Actions of any process in group:
  - `enter()`
    - Send a request to master
    - Wait for token from master
  - `exit()`
    - Send back token to master
- Master Actions:
  - On receiving a request from process Pi
    - if (master has token)
      - Send token to Pi
    - else
      - Add Pi to queue
  - On receiving a token from process Pi
    - if (queue is not empty)
      - Dequeue head of queue (say Pj), send that process the token
    - else
      - Retain token

Performance analysis:
- **Bandwidth**: the total number of messages sent in each enter and exit operation.
  - 2 messages for enter
  - 1 message for exit
- **Client delay**: the delay incurred by a process at each enter and exit operation (when no other process is in, or waiting)
  - 2 message latencies (request + grant)
- **Synchronization delay**: the time interval between one
process exiting the critical section and the next process
entering it (when there is only one process waiting)
  - 2 message latencies (release + grant) 

However, the drawback of this algorithm is too central:
- The leader is the performance bottleneck
- Single point of failure cannot be handled.

## Ring-based Solution

![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201203123855.png)

Ring-based Solution:
- N Processes organized in a virtual ring
- Each process can send message to its successor in ring.
- There would be exactly 1 token in the ring.
- enter()
  - Wait until you get token
- exit() // already have token
  - Pass on token to ring successor
- If receive token, and not currently in enter(), just pass on token to ring successor

Performance analysis:
- **Bandwidth**:
  - Per `enter()`, 1 message by requesting process but up to N messages throughout system
  - 1 message sent per `exit()`
- **Client delay**: 0 to N message transmissions after entering `enter()` - $O(N)$
  - Best case: already have token
  - Worst case: just sent token to neighbor
- **Synchronization delay**: between one process `exit()` from the CS and the next process `enter()`: $O(N)$
  - Between 1 and (N-1) message transmissions.
  - Best case: process in enter() is successor of process in exit()
  - Worst case: process in enter() is predecessor of process in exit()

The drawback of ring-based solution: client/synchronization delay to access CS are $O(N)$, too slow!

## Ricart-Agrawala's Algorithm

Ricart-Agrawala's Algorithm:
- Has No token
- Uses the notion of causality and multicast
- Has lower waiting time to enter CS than RingBased approach