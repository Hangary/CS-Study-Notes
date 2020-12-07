# Concurrency Control

We have two ways of concurrency control:
- ***Pessimistic concurrency control***
  - assume the worst, prevent transactions from accessing the same object
  - E.g. Locking
- ***Optimistic concurrency control***
  - assume the best, allow transactions to write, but check later
  - E.g. Check at commit time, multi-version approaches

## Mutual Exclusion and Locking

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

### Leader Central Solution

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

### Ring-based Solution

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

### Ricart-Agrawala's Algorithm

Ricart-Agrawala's Algorithm:
- Has No token
- Uses the notion of causality and multicast
- Has lower waiting time to enter CS than RingBased approach

## Transaction

Transaction is a series of operations executed by client.
- Each operation is an RPC to a server.
- Transaction either:
  - completes and *commits* all its operations at server
  - Or *aborts* and has no effect on server

Example Code:
```Java
// open a new transaction
int transaction_id = openTransaction();

// do some operations
x = server.getFlightAvailability(ABC, 123, date);
if (x > 0)
    y = server.bookTicket(ABC, 123, date);
server.putSeat(y, "aisle");

// commit entire transaction or abort
closeTransaction(transaction_id); 
```

ACID Properties for Transactions:
- **Atomicity**: *All or nothing* principle: a transaction should either 
  - i) complete successfully, so its effects are recorded in the server objects; or 
  - ii) the transaction has no effect at all.
- **Consistency**: if the server starts in a consistent state, the transaction ends the server in a consistent state.
- **Isolation**: Each transaction must be performed without interference from other transactions.
  - No access to intermediate results/states of other transactions
  - Free from interference by operations of other transactions
- **Durability**: After a transaction has completed successfully, all its effects are saved in permanent storage.

However, 
- Clients and/or servers might crash
- Transactions could run concurrently, i.e., with multiple clients
- Transactions may be distributed, i.e., across multiple servers

Some possible problems:
- Lost Update Problem
- Inconsistent Retrieval Problem

### Serial Equivalence

To prevent transactions from affecting each
other, we could execute them *one at a time* at server.

But this reduces number of concurrent transactions and therefore has a bad performance.
- *Transactions Per Second* (TPS) directly related to revenue of companies. This metric needs to be maximized.

To solve this problem, we use introduce ***Serial Equivalence***.

An interleaving (say $O$) of transaction operations is serially equivalent iff:
- There is some ordering $O'$ of those transactions, one at a time, which
  - Gives the same end-result (for all objects and transactions) as the original interleaving $O$
  - Where the operations of each transaction occur consecutively (one at a time, in a batch)

Checking for Serial Equivalence:
- An operation has an *effect* on
  - The server object if it is a *write*
  - The client (returned value) if it is a *read*
- Two operations are said to be ***conflicting operations***, if their combined effect depends on the order they are executed
  - RW conflict: read(x) and write(x)
  - WR conflict: write(x) and read(x)
  - WW conflict: write(x) and write(x)
- Following operations are not conflicting:
  - NOT read(x) and read(x): swapping them doesn't change their effects
  - NOT read/write(x) and read/write(y): operations over different objects would not affect each other, swapping them ok
- Two transactions are serially equivalent if and
only if all pairs of conflicting operations (pair
containing one operation from each transaction)
are executed in the same order (transaction
order) for all objects (data) they both access.

### Pessimistic Concurrency Control: Locking Solutions

#### Exclusive Locking Solution

Solution:
- Each object has a lock
  - At most one transaction can be holding a lock
- Before reading or writing object O, transaction T must call lock(O)
  - Blocks if another transaction already inside lock
- After entering lock T can read and write O multiple times
- When done (or at commit point), T calls unlock(O)
  - If other transactions waiting at lock(O), allows one of them in

#### Read-Write Locking Solution

Read-Write Locks Solution have better concurrency than simple Exclusive Locking Solution because non-conflicting operations (multiple read) over the same object is now allowed.

Solution:
- Each object has a lock that can be held in one
of two modes:
  - **Read mode**: multiple transactions allowed in
  - **Write mode**: exclusive lock
- Before first reading O, transaction T calls
read_lock(O)
  - T allowed in only if all transactions inside lock for O all entered via read mode
  - Not allowed if any transaction inside lock for O entered via write mode
- Before first writing O, call write_lock(O)
  - Allowed in only if no other transaction inside lock
- If T already holds read_lock(O), and wants to write, call write_lock(O) to promote lock from read to write mode
  - Succeeds only if no other transactions in write mode or read mode
  - Otherwise, T blocks
- Unlock(O) called by transaction T releases any lock on O by 

#### Two-phase Locking (2PL)

**Two-phase locking**:
- A transaction cannot acquire (or promote) any locks after it has started releasing locks
- Transaction has two phases
  1. Growing phase: only acquires or promotes locks
  2. Shrinking phase: only releases locks gradually
   -  **Strict two phase locking**: releases locks only at commit point

----------------

With Locking and 2PL, we can ensure *Serial Equivalence*.

#### Handling DeadLocks

However, 2PL also has its problem: ***deadlocks***.

Three *necessary* conditions for a deadlock to occur: ("Necessary" = if there's a deadlock, these conditions are all definitely true)
1. Some objects are accessed in exclusive lock modes
2. Transactions holding locks cannot be preempted
3. There is a circular wait (cycle) in the Wait-for graph

![Wait-for Graph](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201203164921.png)

Hanlding Deadlock:
1. **Lock timeout**: abort transaction if lock cannot be acquired within timeout $L$
   - Problem: Expensive; leads to wasted work
2. **Deadlock Detection**:
   - keep track of Wait-for graph (e.g., via Global Snapshot algorithm), and
   - find cycles in it (e.g., periodically)
   - If find cycle, there's a deadlock => Abort one or more transactions to break cycle $L$ 
   - Problem: Still allows deadlocks to occur
3.  **Deadlock Prevention**: let the system *violate* one of the necessary deadlock conditions
    1. Some objects are accessed in exclusive lock modes
       - Fix: Allow read-only access to objects
    2. Transactions holding locks cannot be preempted
       - Fix: Allow preemption of some transactions
    3. There is a circular wait (cycle) in the Wait-for graph
       - Fix: Lock all objects in the beginning; if fail any, abort transaction => No cycles in Wait-for graph

### Optimistic Concurrency Control

Optimistic Concurrency Control:
- Increases concurrency more than pessimistic concurrency control
- Increases transactions per second in most of times
- Preferable than pessimistic when conflicts are
expected to be rare

#### Commit and Rollback Solution

Solution:
- At commit point of a transaction T, check for serial equivalence with all other transactions.
- If not serially equivalent,
  - Abort T
  - Roll back (undo) any writes that T did to server objects
    - Any transactions that read from those transactions also now need to be aborted

However, aborting and rolling back would lead to wasted work.

#### Timestamp Ordering Solution

Solution:
- Assign each transaction an id. Transaction id determines its position in **serialization order**
- Ensure that for a transaction T, both are true:
  1. T's write to object O allowed only if transactions that have read or written O had lower ids than T.
  2. T's read to object O is allowed only if O was last written by a transaction with a lower id than T.
- Implemented by maintaining read and write timestamps for the object.
- If rule violated, abort!

#### Multi-version Concurrency Control

Solution:
- For each object, two versions are maintained:
  - **Tentative version**s: A per-transaction version of the object
    - Each tentative version has a *timestamp*
    - Some systems maintain both a *read timestamp* and a *write timestamp*
  - A **committed version**
- On a read or write, find the "correct" tentative version to read or write from
  - "Correct" based on transaction id, and tries to make transactions only read from "immediately previous" transactions

### Eventual Consistency

Eventual Consistency is a form of optimistic concurrency control without transactions. It is widely used in Key-value stores.

#### Last-Write-Wins

**Last-write-wins (LWW)**
- Timestamp, typically based on physical time, used to determine whether to overwrite 
  - if(new write's timestamp > current object's timestamp) 
    - overwrite;
  - else
    - do nothing;
- With unsynchronized clocks:
  - If two writes are close by in time, older write might have a newer timestamp, and might win
  - **Vector Clocks** might be used and **causal ordering** is used decide who is newer.
    - If new write conflicts with existing value, a *sibling value* is created. 
      - User resolve or automatically by app