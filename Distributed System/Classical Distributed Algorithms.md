# Classical Distributed Algorithms

Questions:
- 1. where the synchronous consensus proof (discussed in class) breaks down when the system is asynchronous
- 4. FLP proof Lemma 3
- 6. Leader Election Algorithm
- 7. FIFO-Total Ordering
- 8. K-leader election problem
- 9. Maekawa algorithm
- 10. Mutual Exclusion

## Snapshots

In a cloud, each application or service is running on
multiple servers. Servers handling concurrent events and interacting with each other. The ability to obtain a "global photograph" (snapshot) of the system is important.

**Global Snapshot** = Global State = the combination of
- Individual state of each process in the distributed
system
- Individual state of each communication channel in the
distributed system

i.e. we want to capture the **instantaneous** states of:
- **each process** $P$ and
- **each (process-to-process) communication channel** $C$

Some uses of having a **snapshotof** the system:
- **Checkpointing**: can restart distributed application on failure
- **Garbage collection of objects**: objects at servers that don't have any other objects (at any servers) with pointers to them
- **Deadlock detection**: useful in database transaction systems
- **Termination of computation**: useful in batch computing systems like Folding@Home, SETI@Home

So what is a **state**? When does it change?
- Whenever an *event* happens anywhere in the system, the global state changes
  - Process receives message
  - Process sends message
  - Process takes a step
- State to state movement obeys **causality**

### Chandy-Lamport Global Snapshot Algorithm

**System Model**:
- $N$ processes in the system
- There are two uni-directional communication channels between
each ordered process pair : $P_j \rightarrow P_i$ and $P_i \rightarrow P_j$
- Communication channels are FIFO-ordered
  - First in First out
- No failure
- All messages arrive intact, and are not duplicated
  - Other papers later relaxed some of these assumptions

**Requirements for snapshot**:
- Snapshot should ***not interfere*** with normal application actions, and it should not require application to stop sending messages.
- Each process is able to record its own state
  - Process state: Application-defined state or, in the worst case, its heap, registers, program counter, code, etc. (essentially the coredump)
- Global state is collected in a distributed manner
- Any process may initiate the snapshot
  - We'll assume just one snapshot run for now

**Chandy-Lamport Global Snapshot Algorithm**:

- Initiator $P_i$:
  - Initiator firstly records its own state.
  - Initiator process creates special messages called "**Marker**" messages.
    - Marker message is not an application message, therefore does not interfere with application messages
  - for $j=1$ to $N$ except $i$, $P_i$ sends out a Marker message on outgoing channel $C_{ij}$
    - $N-1$ channels
  - Starts **recording** the incoming messages on each of the incoming channels at $P_i$: $C_{ji}$ (for j=1 to N except i)
- Whenever a process $P_j$ receives a Marker message on an incoming channel $C_{ij}$
  - if this is the first Marker $P_j$ is seeing
    - $P_j$ records its own state first.
    - Marks the state of channel $C_{ij}$ as "**empty**"
    - for $k=1$ to $N$ except $j$
      - $P_j$ sends out a Marker message on outgoing channel $C_{jk}$
    - Starts **recording** the *incoming* messages on each of the incoming channels at $P_j$: $C_{jk}$ (for $k=1$ to N except $i$ and $j$)
      - does not record *outgoing* channels
  - else if already seen a Marker message
    - Mark the state of channel $C_{ij}$ as all the messages that have arrived on it since recording was turned on for $C_{ij}$
      - ==The snapshoot of channels are only at the **receiving** end of the channel. This can ensure that all messages would only be recorded once.==
- The algorithm terminates when:
  - All processes have received a Marker
    - To record their own state
  - All processes have received a Marker on all the $N-1$ incoming channels at each
    - To record the state of all channels
- Then, (if needed), a central server collects all these
partial state pieces to obtain the full global snapshot.

Any run of the Chandy-Lamport Global Snapshot algorithm creates a **consistent cut**.

### Consistent Cuts

***Cut*** = time frontier at each *process* and at each *channel*.
- *Events* at the process/channel that happen before the cut are "**in the cut**".
- *Events* happening after the cut are "**out of the cut**".

***Consistent Cut***: a cut that obeys *causality*.
A cut $C$ is a consistent cut if and only if: 
- for each pair of events $(e, f)$ in the system, where event $e$ is in the cut $C$
    - if $f \rightarrow e$ ($f$ happens-before $e$), then event $f$ is also in the cut $C$

![20201110152441-GrpDAM](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201110152441-GrpDAM.png)

### Chandy-Lamport Global Snapshot algorithm AND consistent cut  

Any run of the Chandy-Lamport Global Snapshot algorithm creates a **consistent cut**.

Proof:
Let's quickly look at the proof:
- Let $e_i$ and $e_j$ be events occurring at $P_i$ and $P_j$ respectively such that
  - $e_i \rightarrow e_j$   $e_i$ happens before $e_j$
- To create a consisten cut, the snapshot algorithm needs to ensure that: *if $e_{j}$ is in the cut, then $e_{i}$ is also in the cut*. 
- That is: *if $e_j \rightarrow < P_j \text{ records its state } >$, then it must be true that $e_i \rightarrow < P_i \text{ records its state } >$*.

We can prove by contradiction:
- Suppose $e_i \rightarrow e_j$, we have $e_{j} \rightarrow < P_j \text{ records its state } >$ and $< P_i \text{ records its state } > \rightarrow e_i$.
  - $<P \text{ records its state } >$ is the cut.
  - This implies that $e_j$ is in the cut while $e_i$ is out the cut.
- Consider the path of app messages (through other processes) that go from $e_i \rightarrow e_j$
- Due to FIFO ordering, markers on each link in above path will precede regular app messages 
- Thus, since $< P_i \text{ records its state } > \rightarrow e_i$, it must be true that $P_j$ received a marker before $e_j$ 
- Thus $e_j$ is not in the cut $\Rightarrow$ contradiction.

### Usage of Chandy-Lamport algorithm

"Correctness" in Distributed Systems can be seen in two ways: ***Liveness*** and ***Safety***.

They are Often confused, but it's important to distinguish from each other:
- ***Liveness***: guarantee that something *good* will *eventually* happen
  - Eventually: does not imply a time bound, but if you let the system run long enough, then it would finally happen
  - Examples:
    - "Completeness" in failure detectors: every failure is eventually detected by some non-faulty process
    - In Consensus: All processes eventually decide on a value
- ***Safety***: guarantee that something *bad* will *never* happen
  - Examples:
    - There is no deadlock in a distributed transaction system
    - No object is orphaned in a distributed object system
    - "Accuracy" in failure detectors
    - In Consensus: No two processes decide on different values

It is difficult to satisfy both *liveness* and *safety*
in an asynchronous distributed system:
- **Failure Detector**: Completeness (Liveness) and Accuracy (Safety) cannot both be guaranteed by a failure detector in an asynchronous distributed system.
- **Consensus**: Decisions (Liveness) and correct decisions (Safety) cannot both be guaranteed by any consensus protocol in an asynchronous distributed system.
- Real-life Example: Very difficult for legal systems (anywhere in the world) to guarantee that all criminals are jailed (Liveness) and no innocents are jailed (Safety)

Recall that a distributed system moves from one global state to another global state, via causal steps
- **Liveness** w.r.t. a property Pr in a given state S means
    - S satisfies Pr, or there is some causal path of global states from S to S' where S' satisfies Pr
- **Safety** w.r.t. a property Pr in a given state S means S satisfies Pr, and all global states S' reachable from S also satisfy Pr

**Chandy-Lamport algorithm** can be used to detect global properties that are **stable**.
- **Stable** = once true, stays true forever afterwards
  - Stable Liveness examples: Computation has terminated
  - Stable Non-Safety examples: 
    - There is a deadlock
    - An object is orphaned (no pointers point to it)
- ==**All** stable global properties can be detected using the Chandy-Lamport algorithm, due to its causal correctness.==

## Multicast

### Multicast Ordering

Communication Forms:
- **Multicast**: message sent to a group of processes
- **Broadcast**: message sent to all processes (anywhere)
- **Unicast**: message sent from one sender process to one receiver process

*Mutlicast* is a widely-used abstraction by almost all cloud systems:
- Replica servers for a key: Writes/reads to the key are multicast within the replica group.
- All servers: membership information (e.g., heartbeats) is multicast across all servers in cluster.

***Multicast Ordering*** determines the meaning of "same order" of multicast delivery at different processes in the group.

Three popular flavors implemented by several multicast protocols:
1. **FIFO ordering**
2. **Causal ordering**
3. **Total ordering**

#### FIFO Ordering Definition

In *FIFO Ordering*, multicasts from each sender are received in the order they are sent, at all receivers.
- Don't worry about the order of multicasts from different senders.

Formal Definition: If a correct process sends $\operatorname{multicast}(g,m)$ to group $g$ and then $\operatorname{multicast}(g, m')$, then every correct process that delivers $m'$ would already have delivered $m$.

#### Causal Ordering

Causal Ordering is an extension of FIFO Ordering, which is stricter. Causal Ordering => (implies) FIFO Ordering.

In *Causal Ordering*, multicasts whose send events are causally related, must be received in the same causality-obeying order at all receivers.

Formal Definition: If $\operatorname{multicast}(g, m) \rightarrow \operatorname{multicast}(g, m')$, then any correct process that delivers $m'$ would already have delivered $m$.
- $\operatorname{multicast}(g, m)$ and $\operatorname{multicast}(g, m')$ do not require to be sent by the same process, which is different from the FIFO Ordering.

#### Total Ordering

Total Ordering is also known as "Atomic Broadcast". Unlike FIFO and causal, this does not pay attention to order of multicast sending. Instead, it focus on the receiving ordering of multicast.

**Total Ordering** ensures all receivers receive all multicasts in the same order.

Formal Definition: If a correct process $P$ delivers message $m$ before $m'$ (independent of the senders), then any other correct process $P'$ that delivers $m'$ would already have delivered $m$.

#### Hybrid Variants

Since FIFO/Causal are orthogonal to Total, we can have hybrid ordering protocols:
- ***FIFO-total hybrid*** protocol satisfies both FIFO and total orders
- ***Causal-total hybrid*** protocol satisfies both Causal and total orders

### Multicast Ordering Implemenation

#### Implementing FIFO Multicast

General Idea: Each process maintains a counter matrix, and multicasts are sent with a sequence number. Processes only deliver multicasts with next expecting sequence number, and buffer later messages.

**Data Structures**: Each receiver maintains a per-sender sequence number (integers):
- Processes $P_1$ through $P_N$
- $P_i$ maintains a vector of sequence numbers $P_i [1 \ldots N]$ (initially all zeroes)
- $P_i [j] $ is the latest sequence number $P_i$ has received from $P_j$

**Updating Rules**:
- **Send multicast** at process $P_j$:
  - Set $P_j [j] = P_j [j] + 1$
  - Include new $P_j [j]$ in multicast message as its sequence number $S$
- **Receive multicast**: If $P_i$ receives a multicast from $P_j$ with sequence number $S$ in message
  - if $(S == P_i [j]+1)$ then
    - deliver message to application 
    - Set $P_i [j]= P_i [j]+1$
  - else **buffer** this multicast until above condition is true

#### Implementing Total Ordering Multicast: Sequencer-based Approach 

A special process elected as ***sequencer*** (usually the *leader*).
- ***Sequencer***:
  - Maintains a global sequence number $S$ (initially $0$) 
  - When it receives a multicast message $M$, it sets $S=S+1$, and multicasts $< M, S >$
    - Therefore, the message $M$ would actually be sent twice: once by sender, and once by the sequencer.
- **Send multicast** at process $P_i$
  - Send multicast message $M$ to group and sequencer
- **Receive multicast** at process $P_i$
  - $P_i$ maintains a local received global sequence number $S_i$ (initially $0$)
  - If $P_i$ receives a multicast $M$ from $P_j$, it buffers it until it both
    1. $P_i$ receives $<M, S(M)>$ from sequencer, and
    2. $S_i + 1 = S(M)$
  - Then deliver it message to application and set $S_i = S_i +1$

#### Implementing Causal Ordering Mulicast

The data structure for Causal Ordering Mulicast is very similar to FIFO. The difference is the update rules:
- **Send multicast** at process $P_j$:
  - Set $P_{j[j]} = P_{j[j]}+1$
  - Include new **entire vector** $P_j[1 \ldots N]$ in multicast message as its sequence number
- **Receive multicast**: If $P_i$ receives a multicast from $P_j$ with vector $M [1 \ldots N ](= P_j[1 \ldots N])$ in message, buffer it until both:
  1. This message is the next one $P_i$ is expecting from $P_j$, i.e. $$ M[j]= P_i[j]+1 $$
  2. All multicasts, anywhere in the group, which happened-before M have been received at $P_i$, i.e., 
    - For all $k \neq j: M [k] \leq P i[k]$
    - i.e., *Receiver satisfies causality*
  3. When above two conditions satisfied, deliver $M$ to application and set $P_i[j]= M[j]$.

### Reliable Multicast

***Reliable Multicast*** needs all *correct* (i.e., nonfaulty) processes to receive the same set of multicasts as all other correct processes.
