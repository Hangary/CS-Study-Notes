# Containerization


## Virtualization

Virtualization is a technique that worked well in the past.

Virtualization add a layer of abstraction on top of hardware and make each OS think it's the only one.

![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201206125633.png)

Pros:
- Can have *isolation* of applications (security)
- Can run things with very *different configurations* next to each other
- Easy to add more systems
Cons:
- Relies on efficient implementation of a ***hypervisor*** - a kind of meta-kernel for the system
- Virtualizing all resources is very *expensive* to enable isolation
- Configuration is more *complex*

Hypervisor is the layer between the hardware and all operating systems:
- Type 1: Translates CPU instructions from the operating system to hardware
  - older, requires less resources
- Type 2: Translates blocks of CPU instructions into kernel calls for a host OS (this is what virtualbox, qemu, etc do)
  - newer, easier configuration, performance depends on workload and hypervisor efficiency
  - can emulater
  - illustrate an important idea for understanding containerization: ***reuse***
  - Reusing existing resources provided by a host OS (via kernal calls) leads to extremely cheap, scalable environments.

## Containerization

![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201206151536.png)

Pros:
- More efficient use of resources since system services are shared and reused
- Can still choose to isolate processes
- Much easier to configure
- Lightweight

Cons:
- No more abstraction around hardware - hardware is shared
- Not all applications can be containerized easily
- Can't run two applications designed for different kernels

Containerization is built on top of Linux ***namespaces***.
- Isolates resources and decides what each process' view of the world is going to look like

A ***namespace*** wraps a global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the global resource. 
- Changes to the global resource are visible to other processes that are members of the namespace, but are invisible to other processes. 
- Namespaces can allow you to isolate resources.
- One important use of namespaces is to implement containers.