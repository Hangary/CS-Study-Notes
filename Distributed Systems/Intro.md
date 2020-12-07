# Distributed System:

Important topics in distributed systems:
- Time
- Asynchrony
- Ordering
  - Events Ordering: Causation
- Failure Handling
- Shared Memory
- Multicast
- P2P

**FLP Impossiblity**: In an asynchronous system with crash-failure, it's literally impossible to achieve consensus.

## Time

Clock:
- Clock Skew
- Clock Drift

## Asynchrony

Difference between Synchronous System and Asynchronous System:
- Synchronous System = Delay is bounded
  - Note: Synchronous $\neq$ Non-Concurrent 
- Asynchronous System = Delay is unbounded. 
  - Network so slow that it will possible never arrive. 

In theory, all distributed systems are asynchronous. In practice, we impose a timeout to make it "synchronous". 
  
