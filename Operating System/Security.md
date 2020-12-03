# Security

- Stack buffer overflow
  - Stack smashing, privilege escalation, remote callbacks
  - Canaries
- Address space layout randomization (ASLR)
  - NOP slides
- Executable space protection (NX bit)
  - Return-oriented programming (ROP)

## Stack Smashing

`strcpy` is function that can be used for stack smashing.
- Other similar functions: `strcat`, `sprintf`, `gets`.

When the return address is overwritten, it will call the value in the return address.

This means that we can inject our own code and run it.

![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201202142253.png)

The whole payload will be: `shellcode + padding + code address`.
- In hacking, a ***shellcode*** is a small piece of code used as the payload in the exploitation of a software vulnerability. 
- The `padding` is used to overflow the buffer and let `code address` replace the `return address`.

![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201202152042.png)

Once we can run our own code in other user's program, we can access files with their privileges. For example, send secret data or open a cheat socket.

To avoid this kind of bugs, we can use `strncpy` instead of `strcpy`.

### Stack Canaries

Stack Canary is a simple defense mechanism against stack smashing. We place a *canary* value at the beginning of the stack frame and check memory address at end of function. If value has changed, stack overflow has occurred.
- Two kinds of canaries:
  - **Random Canary**
    - 4 random bytes
    - Hard to guess
    - Stored in a guarded page
  - **Terminator Canary**
    - 0, CR, LF, -1
    - stops strcpy, strcat, gets, etc.
    - Known to the attacker

![](https://raw.githubusercontent.com/Hangary/CS-Study-Notes/main/images/20201202153658.png)

Limitations of stack canaries:
1. Not enabled by default until gcc 4.8.3. We can disable with `gcc -fno-stack-protector`.
1. Not usually enabled for every function, just the ones likely to be
exploited. Because it has minor performance overflow.
3. Can still overflow function pointers

### Address Space Layout Randomization

Buffer overflow relies on knowing the address of some part of our stack so we can jump to it. Add random offsets to stack (and heap) so we can't predict its addresses.
- Enabled by default on the Linux kernel since 2005

Limitation:
- In practice, the amount of randomness (entropy) can be quite low Range 0xff800000 → 0xffff0000 (approx)
  - Around 2^21 possible values — we can probably brute force
- NOP slide

### Executable Space Protection

NX bit:
- Concept: separation of data from code
- Set a special bit in the page table for a memory block
  - If `1`, then we won't let the CPU execute instructions in that block
- If the program counter `eip` enters a data block, `segfault`
- Enabled by default in gcc — disable with `gcc -z execstack`

To hack this protections, we can use **Return-oriented programming** (ROP):
- We can still smash our return address, but we can't run our own code
- Chain together sequences of existing code to do unexpected things
  - Everything uses libc, so we can count on compatibility
  - Gadgets: parts of the ends of functions—chain them together