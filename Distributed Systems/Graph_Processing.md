# Distributed Graph Processing

Google's Pregel system is the inspiration for many newer graph processing systems: Piccolo, Giraph, GraphLab, PowerGraph, LFGraph, X-Stream, etc.

Large graphs are hard to be stored and processed entirely on one single server. Instead, we use distributed cluster to hanlde graphs.

## Basic Ideas

Typical Graph Processing Application:
- Works in **iterations**
  - Each vertex assigned a value
  - In each iteration, each vertex:
    1. **Gather**: Gathers values from its immediate neighbors (vertices who join it directly with an edge). 
    2. **Apply**: Does some computation using its own value and its neighbors values.
    3. **Scatter**: Updates its new value and sends it out to its neighboring vertices.
- Graph processing terminates after: 
  - i) fixed iterations, or 
  - ii) vertices stop changing values

Basic Distributed Graph Processing:
- Assign each *vertex* to one server
- Each server thus gets a subset of vertices
- In each iteration, each server performs **Gather-Apply-Scatter** for all its assigned vertices
  - **Gather**: get all neighboring vertices' values
  - **Apply**: compute own new value from own old value and gathered neighbors' values
  - **Scatter**: send own new value to neighboring vertices
- A *barrier* is used at the end of each iteration to ensure all vertices are handled.

How to decide which server a given vertex is
assigned to? Two options of assigning vertices:
- **Hash-based**: Hash(vertex id) modulo number of servers
- **Locality-based**: Assign vertices with more neighbors to the same server as its neighbors
  - Reduces server to server communication volume after each iteration
  - Need to be careful: some "intelligent" locality-based schemes may take up a lot of upfront time and may not give sufficient benefits!

## Pregel System By Google

Pregel uses the master/worker model:
- Master (one server)
  - Maintains list of worker servers
  - Monitors workers; restarts them on failure
  - Provides Web-UI monitoring tool of job progress
- Worker (rest of the servers)
  - Processes its vertices
  - Communicates with the other workers

Execution Process:
1. Many copies of the program begin executing on a cluster
2. The master assigns a partition of input (vertices) to each worker
   - Each worker loads the vertices and marks them as active
3. The master instructs each worker to perform a iteration
   - Each worker loops through its active vertices & computes for each vertex
   - Messages can be sent whenever, but need to be delivered before the end of the iteration (i.e., the barrier)
   - When all workers reach iteration barrier, master starts next iteration
4. Computation halts when, in some iteration: no vertices are active and when no messages
are in transit
5. Master instructs each worker to save its portion of the graph

Fault-Tolerance:
- **Checkpointing**
  - Periodically, master instructs the workers to save state of their partitions to persistent storage
    - e.g., Vertex values, edge values, incoming messages
- Failure detection
  - Using periodic *ping* messages from master to worker
- Recovery
  - The master reassigns graph partitions to the currently available workers
  - The workers all reload their partition state from most recent available checkpoint

