# Stream Processing

Processing data in real-time a big requirement today: Large amounts of data => Need for real-time views of data => Stream Processing

MapReduce is a ***Batch Processing*** framework:
- Need to wait for entire computation on large dataset to complete.
- Not intended for long-running stream-processing

## Storm

***Storm*** is one of the most famous Stream Processing frameworks.

### Storm Components

Storm Components:
- **Tuples**: An ordered list of elements
- **Streams**: An *sequence* of tuples
- **Spouts**: An entity (process) that produces streams
  - Often reads from a crawler or DB.
- **Bolts**: An entity (process) that 
  - Processes input streams
  - Outputs more streams for other bolts
- **Topologies**: A directed graph of spouts and bolts (and output bolts)
  - Corresponds to a Storm application
  - Can have cycles if the application requires it

**Bolts** are the main entities that process data. They can perform many operations:
- **Filter**: forward only tuples which satisfy a condition
- **Joins**: When receiving two streams A and B, output all pairs (A,B) which satisfy a condition
- **Apply/Transform (Map)**: Modify each tuple according to a function
- And many others

Bolts are *parallelizing*:
- Multiple processes (called **tasks**) constitute a *bolt*.
- Incoming streams split among the tasks.
- Typically each incoming tuple goes to one task in the bolt.
- How streams are distributed among parallelizing bolts are decided by **Grouping strategy**.

### Grouping Strategy

Three types of grouping are popular:
- **Shuffle Grouping**
  - Streams are distributed evenly among the bolt's tasks
  - Round-robin fashion
  - ![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201203193736.png)
- **Fields Grouping**
  - Group a stream by a subset of its fields
  - ![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201203193839.png)
- **All Grouping**
  - All tasks of bolt receive all input tuples
  - Useful for *joins*
  - ![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201203194004.png)

### Storm Cluster

Structure of a storm cluster:
- **Master node**
  - Runs a daemon called *Nimbus*
  - Responsible for 
    - Distributing code around cluster
    - Assigning *tasks* to machines
    - Monitoring for failures of machines
- **Worker node**
  - Runs on a machine (server)
  - Runs a daemon called *Supervisor*
  - Listens for work assigned to its machines
  - Runs "*Executors*" (which contain groups of *tasks*)
- **Zookeeper**
  - Coordinates *Nimbus* and *Supervisors* communication
  - All state of *Supervisor* and *Nimbus* is kept here

![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201203194509.png)

### Failures

A tuple is considered **failed** when its topology (graph) of resulting tuples fails to be fully processed within a specified *timeout*.

**Anchoring**: Anchor an output to one or more input tuples
- Failure of one tuple causes one or more tuples to replayed

API For Fault-Tolerance (OutputCollector):
- **Emit(tuple, output)**
  - Emits an output tuple, perhaps anchored on an input tuple (first argument)
- **Ack(tuple)**
  - Acknowledge that you (bolt) finished processing a tuple
- **Fail(tuple)**
  - Immediately fail the spout tuple at the root of tuple topology if there is an exception from the database, etc.
- Must remember to ack/fail each tuple
  - Each tuple consumes memory. Failure to do so results in memory leaks.