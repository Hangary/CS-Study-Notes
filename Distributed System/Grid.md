#



## Grid

==TODO 3.2==

Grid is developed for High Performance Computing (HPC).

- Computation Intensive, so Massively Parallel

A Grid application consist of **job**s.

4 stages of a job: 

- Init 
- Stage in 
- Execute 
- Stage out 
- Publish

### Scheduling Problem

How to schedule and parellel these jobs?

![image-20201013000208361](/Users/Hangar/Desktop/Grid.assets/image-20201013000208361.png)

Two levels of protocols:

- Intra-site protocol, like HTCondor Protocol
  - Internal Allocation & Scheduling
  - Monitoring 
  - Distribution and Publishing of Files
- Inter-site protocol, like Globus Protocol
  - External Allocation & Scheduling
  - Stage in & Stage out of Files 

#### Condor (now HTCondor)

- High-throughput computing system from U. Wisconsin Madison 
- Belongs to a class of “Cycle-scavenging” systems –
  - SETI@Home and Folding@Home are other systems in this category 

Such systems

  - Run on a lot of workstations
  - When workstation is free, ask site’s central server (or Globus) for tasks
  - If user hits a keystroke or mouse click, stop task
      - Either kill task or ask server to reschedule task 
  - Can also run on dedicated machines

#### Globus

- Globus Alliance involves universities, national US research labs, and some companies
- Standardized several things, especially software tools 
- Separately, but related: Open Grid Forum
- Globus Alliance has developed the Globus Toolkit