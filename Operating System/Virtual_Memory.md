# Virtual Memory

The **Memory Management Unit** is part of the CPU, and it converts a virtual memory address into a physical address.
- Virtual memory keeps processes safe because one process cannot directly read or modify another process's memory. 
- Virtual memory keeps processes safe because one process cannot directly read or modify another process's memory. 

The modern process of translating memory is as follows.
1. A process makes a memory request.
2. The circuit first checks the **Translation Lookaside Buffer (TLB)** if the address page is cached into memory. It skips to the reading from/writing to phase if found otherwise the request goes to the MMU.
3. The **Memory Management Unit (MMU)** performs the address translation. If the translation succeeds, the page gets pulled from RAM - conceptually the entire page isn't loaded up. The result is cached in the TLB.
4. The CPU performs the operation by either reading from the physical address or writing to the address.


## Virtual Page, Physical Frame, and Page Table

A **virtual page** is a block of virtual memory. 
- A typical block size on Linux is 4KiB or $2^{12}$ addresses, though one can find examples of larger blocks. 
- One process can have multiple virtual pages. For example, some for heap and some for stack. 

![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201207131502.png)


Each virtual page corresponds to a **physical frame** or sometimes called a '**page frame**', which is a block of physical memory or RAM - Random Access Memory. 
- A frame is the same number of bytes as a virtual page or 4KiB on our machine.
- To access a particular byte in a frame, an MMU goes from the start of the frame and adds the offset.

A **page table** is a map from a **virtual page** to a particular **physical frame**. 
- The assignment of frames are not in order. For example Page 1 might be mapped to frame 45, page 2 mapped to frame 30. 
- Other frames might be currently unused or assigned to other running processes or used internally by the operating system.

![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201207131520.png)

## Multi-level Page Tables

Multi-level pages are one solution to the page table size issue for 64-bit architectures. 

![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201207131537.png)

So what is the intuition for dereferencing an address?
1. the MMU takes the top-level page table and find the Index1'th entry. That will contain a number that will lead the MMU to the appropriate sub-table 
2. go to the Index2'th entry of that table. That will contain a frame number. This is the good old fashioned 4KiB RAM that we were talking about earlier. 
3. the MMU adds the offset and do the read or write.

![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201207131552.png)


## Translation Lookaside Buffer (TLB)

> So what is the intuition for dereferencing an address? First, the MMU takes the top-level page table and find the Index1'th entry. That will contain a number that will lead the MMU to the appropriate sub-table Then go to the Index2'th entry of that table. That will contain a frame number. This is the good old fashioned 4KiB RAM that we were talking about earlier. Then, the MMU adds the offset and do the read or write.

Translation Lookaside Buffer is very fast cache for lookup table. 

TLB makes use of spatial locality and temporal locality.

## Page Fault, Demand Page In, Page Out

A page might not be in our memory. 

## Dirty Bit and other Page Attributes

Dirty bit means a page has been modified.

Page Attributes:
1. Readonly
2. Executable