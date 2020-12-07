# Memory

Types of variables in program memory:
- **Static variables**
  - Persist throughout the lifetime of the program
  - No runtime cost, but memory cannot be used for anything else
- **Local variables, function arguments**
  - Live for the lifetime of the function call
  - Stored on the **stack**
  - Allocation and reclaiming of memory is very **cheap**, but limited by the fixed size
- **Heap allocation**
  - *Slower* than stack allocation because of bookkeeping by memory manager
  - This memory is not automatically reclaimed, so will eventually fill up

## Garbage Collection

Garbage collection is used to manage the heap automatically.

Some common ways of garbage collection:
- Reference Counting
- Tracing
- Tri-Color Marking
- 

### Reference Counting

**Reference Counting**:
- Each object has count of how many objects point to it
- Need to track count whenever object pointers are set or removed/freed

Problems of Reference Counting:
- Overhead of maintaining count
- Lumpy deletions: tend to have clusters of related objects.
- Reference cycles

### Tracing

**Tracing** is the most common form of garbage collection.
- "Trace" connections between references and allocations

Two Step Process:
- **Mark** - Find and label all accessible objects
  - Follow all pointers from the "root set"
  - Perform DFS on pointers
  - Flag field set to marked
- **Sweep** - Remove all inaccessible objects
  - Iterate through memory
  - If marked, unmark; Else, free

Pros:
- Works with cyclic data structures
- Effective and easy to implement
Cons:
- Program must **halt**
- **Fragmentation**: Memory is divided into many small pieces. It can be hard to allocate a new large piece.

We can solve the Fragmentation problem with **Compaction**: Sweep moves memory to the left.
![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201205165854.png)

### Tri-Color Marking

Tri-Color Marking is an incremental garbage collector:
- Gray Set - Need to be scanned blocks
- Black Set - Accessible blocks
- White Set - Inaccessible blocks

Process:
- All blocks accessible from the root start in the gray set.
- All others start in the white set.
- Gray moves to black, all white set objects referenced moved to gray.

### Generational GC

Do we really need to scan everything all the time?
- Split the objects up into generations
- Generations: Eden -> Survival -> Old
- Scan each generation with a different frequency
  - Scan young generation frequently
  - Scan old generation rarely


![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201205170133.png)