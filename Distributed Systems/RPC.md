# RPC

**RPC = Remote Procedure Call**
- Proposed by Birrell and Nelson in 1984
- Important abstraction for processes to call functions in other processes

Firstly, we can discuss ***Local Procedure Call*** (LPC):
- Call from one function to another function within the same process
  - Uses stack to pass arguments and return values
  - Accesses objects via pointers (e.g., C) or by reference (e.g., Java)
- LPC has **exactly-once** semantics
  - If process is alive, called function executed exactly once

**Remote Procedure Call**: Call from one function to another function, where caller and callee function reside in different processes
  - Function call crosses a process boundary
  - Accesses procedures via global references
    - Can't use pointers across processes since a reference address in process 
    - Procedure address = IP + port + procedure number

RPC Call Semantics:
- **Not exactly-once**: Under failures, hard to guarantee exactly-once semantics
- Function may not be executed if
  - Request (call) message is dropped
  - Reply (return) message is dropped
  - Called process fails before executing called function
  - Called process fails after executing called function
  - Hard for caller to distinguish these cases
- Function may be executed multiple times if
  - Request (call) message is duplicated

Possible RPC Call semantics:
- At most once semantics (e.g., Java RMI)
- At least once semantics (e.g., Sun RPC)
- Maybe, i.e., best-effort (e.g., CORBA)

**Idempotent operations** are those that can be repeated multiple times, without any side effects.
- Idempotent operations can be used with at-leastonce semantics.

RPC Implementation:
- Client
  - Client stub: has same function signature as callee()
    - Allows same caller() code to be used for LPC and RPC
  - Communication Module: Forwards requests and replies to appropriate hosts
- Server
  - Dispatcher: Selects which server stub to forward request to
  - Server stub: calls callee(), allows it to return a value
  
![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201203133408.png)

**Code Generation**:
- Programmer usually only writes code for caller function and callee function
- Code for remaining components all generated automatically from function signatures (or object interfaces in Object-based languages)

**Marshalling**:
- Different architectures use different ways of representing data. (e.g. little endian and big endian)
- Caller (and callee) process uses its own *platform-dependent* way of storing data
- Middleware has a common data representation (**CDR**) which is *platform-independent*
- **Marshalling**: Caller process converts arguments into CDR format
- **Unmarshalling**: Callee process extracts arguments from message into its own platform-dependent format
- Return values are marshalled on callee process and unmarshalled at caller process