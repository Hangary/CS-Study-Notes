# Scheduling

Schedulers are in charge of allocating various resources among jobs.

Resources include:
- processor
- memory
- VMs
- and so on

## Single Processor Scheduling

For a single processor scheduling, we have different scheduling ways:
- First-In First-Out
- Shortest Task First
  - **Optimal** average completion
- Round-Robin
  - Use a *quantum* to run portion of task at queue head
  - Pre-empts processes by saving their state, and resuming later
  - After pre-empting, add to end of queue

Preferance:
- Round-Robin preferable for
  - Interactive applications: User needs quick responses from system
- FIFO/STF preferable for Batch applications
  - User submits jobs, goes away, comes back to get result.

## Cloud Scheduling

### Hadoop Scheduling

A Hadoop job consists of Map tasks and Reduce tasks.

Hadoop YARN has two popular schedulers:
- **Hadoop Capacity Scheduler**
- **Hadoop Fair Scheduler**

#### Hadoop Capacity Scheduler

A Hadoop Capacity Scheduler contains multiple *queues*
- Each queue contains multiple *jobs*
- Each queue guaranteed some portion of the cluster capacity
  - E.g., 
    - Queue 1 is given 80% of cluster, Queue 2 is given 20% of cluster
    - Higher-priority jobs go to Queue 1
- For jobs within same queue, **FIFO** typically used
- Queues can be hierarchical:
  - May contain child sub-queues, which may contain child sub-queues, and so on
  - Child sub-queues can share resources equally

Administrators can configure queues:
- Soft limit: how much % of cluster is the queue guaranteed to occupy
- Hard limit (Optional) : max % of cluster given to the queue

Elasticity:
- A queue allowed to occupy more of cluster if resources free
- But if other queues below their capacity limit, now get full, need to give these other queues resources

**Pre-emption not allowed**:
- Cannot stop a task part-way through
- When reducing % cluster to a queue, wait until some tasks of that queue have finished

#### Hadoop Fair Scheduler

Goal: all jobs get equal share of resources
- When only one job present, occupies entire cluster
- As other jobs arrive, each job given equal % of cluster
- Divides cluster into *pools*
  - Typically one pool per user
- Resources divided equally among pools
  - Gives each user fair share of cluster
- Within each pool, can use either
  - Fair share scheduling, or
  - FIFO/FCFS
  - (Configurable)
- Some pools may have minimum shares
  - When minimum share not met in a pool, for a while
    - Take resources away from other pools
    - By pre-empting jobs in those other pools
    - By **killing** the currently-running tasks of those jobs
      - Tasks can be re-started later
      - Ok since tasks are idempotent!
    - To kill, scheduler picks most-recently-started tasks to Minimizes wasted work

### Dominant-Resource Fair Scheduling

Jobs may have multi-resource requirements instead of a single resource. 
e.g.
- Job 1's tasks: 2 CPUs, 8 GB
- Job 2's tasks: 6 CPUs, 2 GB

How do we schedule these jobs in a fair manner?
Use ***Dominant Resource Fairness***.

DRF is
- Fair for multi-tenant systems
- Strategy-proof: tenant can't benefit by lying
- Envy-free: tenant can't envy another tenant's allocations

DRF is useful in scheduling
- VMs in a cluster
- Hadoop in a cluster
- CPU, RAM, Network, Disk, etc.a

Calculate resource vector, dominant resource.

DRF Fair: For a given job, the % of its dominant resource type that it gets cluster-wide, is the same for all jobs.