# Chapter 3: Address Spaces { #Chapter3 }

-------------------------------------------------------------------------------
**Summary**
In this chapter, we will look at a process - a progam with life. The program on
file is a collection of sections. We will see how it looks like in memory, when
it is under execution. We will explore a process's memory layout and address
spaces.
-------------------------------------------------------------------------------

## 3.1 Init { #Section3.1 }

<p>
Let us start with a question. What is a process? A process is a program under execution. A program is a dead file sitting in the secondary storage. But a process is full of life, present in main memory and is running.

Let us start with a very simple operating system. An operating system where a user can run only one program at a time - a single-tasking operating system. There can be multiple program(files) stored in the secondary storage. But only one program can be executed at once ie., only one process can exist in memory at once. Figure-1 shows us how the RAM would look like when a process is running.
</p>

```
-------------------------------------------------------------------------------
|-------------------------|<-------- RAMSIZE-1
|                         |
|     OS text, data       |
|                         |
|-------------------------|<-------- Boundary
|                         |
|                         |
|   Program text, data    |
|                         |
|                         |
|-------------------------|<-------- 0x0
--------------------------------------------------------------------------Fig-1
```

<p>
One portion of main memory will always have operating system text and data. The operating system is always present in main memory. It is supposed to be an efficient resource manager. When a program is run, parts of the program file essential for it to run are copied to the main memory. In the [previous chapter](#Chapter2), we discussed about Physical Address Space. If the processor has to access something on the main memory, it will use the physical address of that content and fetch it. Early versions of [MS-DOS](https://en.wikipedia.org/wiki/MS-DOS) were single-user single-tasking operating system. Let us talk in terms of address spaces. A part of the processor's physical address space is used to address the entire main memory. Let us call it the main memory space. In the above scenario, the main memory space is divided into 2 parts: operating system space(**os-space**) and user program space(**user-space**). The operating system being the supervisor, its text should be protected from any process run on this machine. A program run by some user should not be able to change the operating system's text, data or anything. The operating system should enforce a strict mechanism defining the boundaries. In current day operating systems, if a user process tried to access some OS-space memory in the os-space, the process will be terminated for doing it.

Let us move a step forward. Consider a multi-user operating system. Assume there are currently 5 users on this system and each of them run a program. With the single-tasking nature, the operating system should generate an order to execute the 5 programs. The following is one order of execution.
</p>


```
------------------------------------------------------Straight-forward ordering
|-----|-----|-----|-----|-----|
|  P1 | P2  |  P3 | P4  |  P5 |
|-----|-----|-----|-----|-----|
   ^
   |
   |
  Head 
--------------------------------------------------------------------------Fig-2
```

<p>
Let us take a closer look at how the 5 programs get executed. Initially, P1 is copied onto main memory. It is executed. Once it is done, the user-space is cleaned up, P2 is copied and executed. In the same way, P3, P4 and P5 are executed. This is one way of ordering(or **scheduling**) the programs for execution. The operating system can choose different orders of execution. It can order based on program-priority, if the user executing the program is root or not etc., The straight-forward ordering gets the job done but there are problems. When P1 is executing, only U1 will be busy interacting with his program - input, output, computation while the other 4 users wait for their opportunity. U5 waits for the longest amount of time. It would be best if all the 5 processes run simultaneously. But that is not possible because there is only 1 CPU and only 1 process can run on the CPU at any given moment. Can we do something to give the users an **illusion** of these 5 processes running simultaneously? How can this be implemented? Instead of the above straight-forward order of execution, consider a **round-robin** ordering. Total execution time is divided into time-intervals of equal lengths T units of time. P1 runs for T units of time. Once T units are over, P1 stops running and P2 starts running. P2 runs for T units. Once T units are done, P2 stops and P3 starts. Continuing, once P5 runs for T units, P5 is stopped and P1 is brought back. This keeps happening till all processes come to an end. Take a look at the following diagram.
</p>

```
-----------------------------------------------------------Round Robin ordering 
|<-T->|<-T->|<-T->|<-T->|<-T->|<-T->|<-T->|<-T->|<-T->|<-T->|<-T->|<-T->|..

|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|
|  P1 |  P2 |  P3 |  P4 |  P5 |  P1 |  P2 |  P3 |  P4 |  P5 |  P1 |  P3 |..
|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|
                                                                  ^
                                                                  |
                                                    Assuming P2 is done running 
--------------------------------------------------------------------------Fig-3
```

<p>
You might ask even here U5 waits till P1, P2, P3 and P4 finishes its T units of time? That is true. The time interval T is generally a  small quantity. Say T is 5ms. Excluding the first round, each process waits for (4 * T) units of time. This means each user will also have to wait for (4 * T) = 20ms. From a process's point of view, 5ms is a lot of time. A lot of instructions get executed in 5ms. From a user's point of view, 20ms is a really small amount of time. Waiting for 20ms is almost equivalent to not waiting. This way, the users **feel** that their programs are run simultaneously but even now only one process is run at any given point in time. This idea of providing an **illusion** of simultaneous execution of programs is called **Time-Sharing**. When the concept of time-sharing is used to to run programs of various users, it is known as a multi-tasking operating system. The [original UNIX operating system](https://people.eecs.berkeley.edu/~brewer/cs262/unix.pdf) was a time-sharing operating system.

That is the concept. Now let us see how it can be implemented in the operating system. A fresh copy of text, data etc., are loaded into memory the first time a program(say P1) comes under execution. When it stops running after T units of time, it will come back some time later. When it comes back, it needs to start exactly at the point where execution had stopped the last time. Because it was executed for T units of time, the processor's register values might be different, the variables' values might be different, execution probably had stopped in the middle of some function, the runtime stack had some contents etc., basically, the **process state** would have changed. The operating system should take a **snapshot** of P1 before P2 is put under execution. When its P1's turn again, the operating system must load the snapshot and not a fresh copy. How can this snapshot mechanism be implemented? The snapshot is nothing but the user-space after P1 runs for T units of time. This snapshot can be copied onto the secondary storage. Later when its P1's turn, this snapshot will be copied  back onto the main memory. This technique is called **swapping**. You swap out P1 into secondary storage when its time-interval expires and swap in P2 into the main memory when its time-interval starts. The part of secondary storage used for swapping is called **swap-space**. The swap-space here should be a multiple of size of user-space. Because the swap-space is used by the operating system, it should be protected from users. The RAMSIZE is generally very small compared to the size of secondary storage. That is why we can make use of a piece of secondary storage for swapping. 

Note that the memory manager part of the operating system was very simple when it implemented the straight-forward ordering. Time-sharing increased the complexity. Let us calculate the total execution times of the 2 types of ordering. Let T1, T2, T3, T4 and T5 be the execution times of P1, P2, P3, P4 and P5 respectively. Each program needs to be loaded into memory and once execution is over, memory needs cleanup(Why?). Amount of time needed for loading and cleanup depends on the size of the program. Loading is nothing but copying the program from secondary storage to main memory. Bigger the program, more time it takes to load. Similarly, bigger the program, more time is needed to cleanup. Let it take T_Load units of time to load X bytes from secondary storage to main memory and T_Clean units of time to clean X bytes of main memory. With that, we can calculate total runtimes for both orderings. Total runtime = Total Program Execution time + Total Load time + Total Cleanup time

1. Straight-forward Ordering: 
    * Total runtime = (T1 + T2 + T3 + T4 + T5) + [Ceil(sizeof(Pi) / X ) * T_Load][i=1 to 5] + [Ceil(sizeof(Pi) / X) * T_Clean][i = 1 to 5]

2. Round-Robin ordering
    * Total runtime = (T1 + T2 + T3 + T4 + T5) + [Ceil(Ti/T) * Ceil(sizeof(Pi)/X) * T_Load][i = 1 to 5] + [Ceil(Ti/T) * Ceil(sizeof(Pi)/X) * T_Clean][i = 1 to 5].

In a practical scenario, there won't be a fixed number of programs. Programs will be coming in for execution and leaving after execution. It is highly dynamic scenario. In the straight-forward ordering system, the new program is put at the end of the execution queue and it has to wait till every program before it in the queue is executed. But in a time-sharing system, it is a different scene. It is given its T units of time as soon as possible. Then it becomes part of the round-robin ordering.

Observe that the total runtime of the time-sharing system is significantly higher than the straight-forward ordering system. The operating system also has to do a lot more work in the time-sharing scenario. That is the cost paid in return for fairness and a time-sharing system.

That was one thread of exploration to which we will come back later. Let us take up another interesting thread. 

The linker is the one which gives absolute addresses to text and data. When the program is run, these sections are copied to the memory at the addresses alloted by the linker. With almost all the examples we have seen, the text starts at 0x0 and data starts right after text ends. The linker started with the first unused address(which is 0x00) and gave it to text. Once all bytes in text are given addresses, it took the next unused address(which is 0x00 + sizeof(text)-1) and gave it to data. Consider the following assembly program. Assume that we are working on a 64-bit system and addresses are 64-bits in size.
</p>

```asm
------------------------------------------------------------------------hello.s
section .data

str: db "Hello World!", 0x00
ptr: dq 0x0000000000000000              ; pointer

section .text
        global _start

_start:
        xor r15, r15            ; Clean up the register
        lea r14, [str]          ; Get the starting address of the string
        mov qword[ptr], r14     ; ptr is a pointer    
loop:   
        mov r14, qword[ptr]     ; Load the pointer into a register
        mov r15b, byte[r14]     ; Get the byte, r15 = *(char *)r14;
        cmp r15b, 0x00          ; Compare it with NULL
        je loop_out             ; If it is NULL, then we have hit the end. Get out!

        mov r8, r15             ; Just load the character into register r8

        inc qword[ptr]          ; Increment the pointer
        jmp loop                ; Jump back to the beginning

loop_out:
        mov rax, 0x01   
-------------------------------------------------------------------------------
```

<p>
Don't worry about what the program does. It is assembled and linked with 0x00 as text's starting address and data coming right after text. This is how the program would look like.
</p>

```
-------------------------------------------------------------------------------
Disassembly of section .text:

0000000000000000 <_start>:
  000000:	4d 31 ff             	xor    r15,r15
  000003:	4c 8d 34 25 36 00 00 	lea    r14,0x36
  00000a:	00 
  00000b:	4c 89 34 25 43 00 00 	mov    QWORD PTR 0x43,r14
  000012:	00 

0000000000000013 <loop>:
  000013:	4c 8b 34 25 43 00 00 	mov    r14,QWORD PTR 0x43
  00001a:	00 
  00001b:	45 8a 3e             	mov    r15b,BYTE PTR [r14]
  00001e:	41 80 ff 00          	cmp    r15b,0x0
  000022:	74 0d                	je     0x31 <loop_out>
  000024:	4d 89 f8             	mov    r8,r15
  000027:	48 ff 04 25 43 00 00 	inc    QWORD PTR 0x43
  00002e:	00 
  00002f:	eb e2                	jmp    0x13 <loop>

0000000000000031 <loop_out>:
  000031:	b8 01 00 00 00       	mov    eax,0x1

Disassembly of section .data:

0000000000000036 <str>:
  000036:	48 65 6c 6c 6f 20 57 6f 72 6c 64 21 00

0000000000000043 <ptr>:
	000043: 00 00 00 00 00 00 00 00
-------------------------------------------------------------------------------
```

<p>
Figure-4 shows how the program looks like in memory.
</p>

```
-------------------------------------------------------------------------------
|-------------------| <-- 0x00000000bfffffff // Highest available user-space address
|                   | 
|                   |
|     unused        |
|                   |
|-------------------| <-- 0x000000000000004a // data ends
|                   |
|      data         |
|                   |
|-------------------| <-- 0x0000000000000036 // Starting of data
|                   |
|                   |
|      text         |
|                   |
|-------------------| <-- 0x0000000000000000 // Starting of text
--------------------------------------------------------------------------Fig-4
```

<p>
And the program runs properly. If the linker had the information about the amount of user-space physical memory available, it could have alloted different addresses to text and data other than 0x00 and (0x00 + sizeof(text)-1). Assume that the linker now knows that there is 3GiB of user-space physical memory available. It can allot some other addresses for both text and data. Assume that it gave the addresses 0x4000b0 to the text and 0x6000e8 to data. The program would look like this.
</p>

```
-------------------------------------------------------------------------------
chapter3$ objdump -Mintel -D hello

hello:     file format elf64-x86-64


Disassembly of section .text:

00000000004000b0 <_start>:
  4000b0:	4d 31 ff             	xor    r15,r15
  4000b3:	4c 8d 34 25 e8 00 60 	lea    r14,0x6000e8
  4000ba:	00 
  4000bb:	4c 89 34 25 f5 00 60 	mov    QWORD PTR 0x6000f5,r14
  4000c2:	00 

00000000004000c3 <loop>:
  4000c3:	4c 8b 34 25 f5 00 60 	mov    r14,QWORD PTR 0x6000f5
  4000ca:	00 
  4000cb:	45 8a 3e             	mov    r15b,BYTE PTR [r14]
  4000ce:	41 80 ff 00          	cmp    r15b,0x0
  4000d2:	74 0d                	je     4000e1 <loop_out>
  4000d4:	4d 89 f8             	mov    r8,r15
  4000d7:	48 ff 04 25 f5 00 60 	inc    QWORD PTR 0x6000f5
  4000de:	00 
  4000df:	eb e2                	jmp    4000c3 <loop>

00000000004000e1 <loop_out>:
  4000e1:	b8 01 00 00 00       	mov    eax,0x1

Disassembly of section .data:

00000000006000e8 <str>:
  6000e8:	48 65 6c 6c 6f 20 57 6f	72 6c 64 21 00

00000000006000f5 <ptr>:
	6000f5: 00 00 00 00 00 00 00 00
-------------------------------------------------------------------------------
```

<p>
The process would look like the following in memory.
</p>

```
-------------------------------------------------------------------------------
|-------------------| <-- 0x00000000bfffffff // Highest available user-space address
|                   | 
|      unused       |
|-------------------| <-- 0x00000000006000fc // data ends
|      data         |
|-------------------| <-- 0x00000000006000e8 // data starts
|                   |
|      unused       |
|-------------------| <-- 0x00000000004000e1 // text ends 
|                   | 
|      text         |
|-------------------| <-- 0x00000000004000b0 // text starts
|      unused       |
|                   |
|-------------------| <-- 0x0000000000000000 // Lowest available user-space address
--------------------------------------------------------------------------Fig-5
```

<p>
Even with this, the program should run properly. Important point to note is that the number of user-space physical addresses are limited. The linker cannot allot addresses greater than 0x00000000bfffffff because there are only 3GiB of user-space memory available.

Now, let us discuss something interesting. Consider a simple single-tasking operating system which uses the straight-forward ordering. Maximum process size is RAMSIZE - OSSIZE. We assumed that the process size will be atmost RAMSIZE - OSSIZE. Assume you have a 4GiB RAM, 64-bit CPU. Assuming OSSIZE to be 1GiB, user-space is 3GiB. But I have a program which requires 5GB of main memory to run. Is it somehow possible to run this big process on our system? Obviously, the current memory manager cannot handle such cases. Can you come up with a mechanism to handle this? Consider a hypothetical system with a 64-bit CPU along with just the amount of user-space needed to store 1 instruction and 1 data element. In x64 architecture, instruction size can range from 1 byte to 15 bytes. Maximum data size is 8 bytes. Note that the OS is still in the RAM, just the user-space is equal to 23 bytes - 15 bytes for instruction, 8 bytes for data. 
<p>

```
-------------------------------------------------------------------------------
|-------------------------|<-------- RAMSIZE-1
|                         |
|     OS text, data       |
|                         |
|-------------------------|<-------- Boundary
|                         |<-- 0x16 //  Data space ends here
|                         |.
|                         |.
|                         |<-- 0x10
|     User-space          |<-- 0x0f // Data space starts here
|                         |<-- 0x0e // Instruction space ends here
|                         |.
|                         |.
|                         |.
|                         |<-- 0x01
|-------------------------|<-- 0x00 // Instruction space starts here
--------------------------------------------------------------------------Fig-6
```

<p>
With the above system, you need to execute *hello*, the executable generated from *hello.s*. This program traverses through the hello-world string. It is exactly how we do it in C. We use a pointer, dereference it to get the character, we are simply loading that character into register r8 and then incrementing the pointer. When the character is NULL, you exit the loop. 

If the linker knew that there is only 23 bytes of user-space available, it would probably give an error. But let us ignore it and generate the executable *hello*. An address is a unique identifier of a memory location. An address has meaning only when it actually points to a memory location right? With a 3GiB user-space system, the highest address available is 0x00000000bfffffff. If I am talking about 0xd1234567 or any address above 0xbfffffff, it is meaningless. We have the exact same situation now. We have 23 bytes of user-space physical memory which means the only valid addresses are 0x00 to 0x16. If the process's size is greater than 23 bytes, then with the current system, we can't run it. But challenge is to come up with a mechanism to run it. For fun, let us use the executable with different addresses for text and data. The following is the executable.

```
-----------------------------------------------------------Disassembly of hello
Disassembly of section .text:

00000000004000b0 <_start>:
  4000b0:	4d 31 ff             	xor    r15,r15
  4000b3:	4c 8d 34 25 e8 00 60 	lea    r14,0x6000e8
  4000ba:	00 
  4000bb:	4c 89 34 25 f5 00 60 	mov    QWORD PTR 0x6000f5,r14
  4000c2:	00 

00000000004000c3 <loop>:
  4000c3:	4c 8b 34 25 f5 00 60 	mov    r14,QWORD PTR 0x6000f5
  4000ca:	00 
  4000cb:	45 8a 3e             	mov    r15b,BYTE PTR [r14]
  4000ce:	41 80 ff 00          	cmp    r15b,0x0
  4000d2:	74 0d                	je     4000e1 <loop_out>
  4000d4:	4d 89 f8             	mov    r8,r15
  4000d7:	48 ff 04 25 f5 00 60 	inc    QWORD PTR 0x6000f5
  4000de:	00 
  4000df:	eb e2                	jmp    4000c3 <loop>

00000000004000e1 <loop_out>:
  4000e1:	b8 01 00 00 00       	mov    eax,0x1

Disassembly of section .data:

00000000006000e8 <str>:
  6000e8:	48 65 6c 6c 6f 20 57 6f	72 6c 64 21 00

00000000006000f5 <ptr>:
	6000f5: 00 00 00 00 00 00 00 00
-------------------------------------------------------------------------------
```

<p>
The above disassembly tells us what byte will be present at what address IF the complete program is loaded into main memory - that is not happening in our case. Observe all the information it gives about the program. It gives a relative positioning of every byte, every instruction and data element. IF text's starting address is 0x4000b0, then the first instruction is situated at 0x4000b0, second instruction at (0x4000b0 + 3) = 0x4000b3, third at (0x4000b3 + 8) = 0x4000bb and so on. IF data's starting address is 0x6000e8, then str is situated at 0x6000e8, ptr is situated at (0x6000e8 + 13) = 0x6000f5. Even though these are **fake addresses** and don't have real meaning because there is no physical memory to back it up, it provides complete structure to the program.

At this point, we have 2 types of addresses. One is the real, physical addresses which actually point to memory locations. We have 23 of them - 0x00 to 0x16, but there are only 2 relevant physical addresses - 0x00 where a single instruction can be placed and 0x0f where a single data element can be placed. Secondly, we have the fake addresses which gives complete structure of the program but addresses don't refer to any memory locations. 

In our discussions so far, processors used physical addresses and executed the program - quite natural. The processor understood the program structure from the physical addresses. If physical address 0x1234 is the first instruction's address and the first instruction is 5 bytes long, this meant that the second instruction was stored at physical address 0x1238. But now, there are just 2 relevant physical addresses. With that, the processor cannot understand anything about the program. With physical address 0x00, the program does not know what instruction comes after what or with 0x0f, processor cannot figure out where str or ptr is present. The fake addresses has all this information. Infact, it has all the information about the program. Qn: Instead of making the processor work on physical addresses, can we make it work on these fake addresses and execute programs?

We have memory for 1 instruction(max size = 15 bytes) and 1 data element(max size = 8 bytes). Before program execution, the complete program is in secondary storage. We can execute the program instruction by instruction. We load the first instruction from secondary storage into physical memory at address 0x00. Execute it. If some data element is needed, fetch it from secondary storage and put it into the physical memory alloted data element. That is the plan.

On file, the program looks like this.
<p>

```
-------------------------------------------------File offsets of the 2 sections
   text(0x00)-->|--------------------------|
                |                          |
                |                          |
                |          text            |
                |                          |
                |                          |
   data(0x36)-> |--------------------------|
                |          data            |
                |--------------------------|
--------------------------------------------------------------------------Fig-7
```

<p>
On file, the program's size is still sizeof(text) + sizeof(data) = 54 + 21 = 75 bytes. The file-offset of text is 0x00 ie., text is at the beginning of the file. data's File-offset is 0x36.

On the fake address range, the program looks like this. Because we are dealing with a 64-bit CPU, highest fake address is 0xffffffffffffffff.
</p>

```
-------------------------------------------------------------Fake address space
|-------------------| <-- 0xffffffffffffffff // Highest available fake address
|                   | 
|      unused       |
|-------------------| <-- 0x00000000006000fc // data ends
|      data         |
|-------------------| <-- 0x00000000006000e8 // data starts
|                   |
|      unused       |
|-------------------| <-- 0x00000000004000e1 // text ends 
|                   | 
|      text         |
|-------------------| <-- 0x00000000004000b0 // text starts
|      unused       |
|                   |
|-------------------| <-- 0x0000000000000000 // Lowest available fake address
--------------------------------------------------------------------------Fig-8
```

<p>
The operating system loads the fake address of the first instruction(which is 0x00000000004000b0) into the instruction-pointer register(**rip**) and leaves. The processor looks at the physical address 0x00 where an instruction should be present. How does it know if the bytes present at physical address 0x00 are the instruction bytes or some garbage? It may actually pick some garbage value present at physical address 0x00 assuming them to be instruction bytes and continue execution - which is incorrect. What can be do to make sure that does not happen? We can store the fake address whose content(an instruction in this case) is present at physical address 0x00. The processor can use that to verify what instruction is present at physical address 0x00. We can a similar entry for the data element. Lets form a small 2-entry table for this. Initially, this is how the table looks like.

| Physical Address | Fake Address    |
|------------------|-----------------|
|       0x00       |    --------     |
|       0x0f       |    --------     |

Call this table address-mapping table. Because it maps the two physical addresses to Fake addresses. The processor refers to the mapping table and sees that the physical address 0x00(the location where instruction is present) has an empty fake address entry. This means no instruction is present at physical address 0x00. It looks at the **rip** register and requests the operating system to fetch the instruction at the fake address 0x00000000004000b0. Can you tell what instruction situated at the fake address 0x00000000004000b0? How will you fetch it? You know that the text starts with fake address 0x00000000004000b0. The first instruction would be situated at a file-offset of (0x00000000004000b0 - 0x00000000004000b0) = 0 into the file. With the file-offset, you can fetch the instruction bytes without any problem. The operating system fetches and loads **4d-31-ff**(xor r15, r15) at the physical address 0x00. The operating system also updates the corresponding table entry. The processor fetches the instruction from the physical address and executes it. The table and user-space memory looks like this.

| Physical Address | Fake Address           |
|------------------|------------------------|
|       0x00       | 0x00000000004000b0     |
|       0x0f       |    --------            |
</p>

```
-------------------------------------------------------------------------------
0x00: 4d 31 ff 00 00 00 00 00 00 00 00 00 00 00 00  ; xor r15, r15
0x0f: 00 00 00 00 00 00 00 00                       ; No data element
-------------------------------------------------------------------------------
```

<p>
The instruction is 3-byte long and is situated at the fake address 0x00000000004000b0. This means that the next instruction is at the fake address (0x00000000004000b0 + 3) = 0x00000000004000b3. The processor looks at the table, makes sure that the instruction at fake address 0x00000000004000b3 is not present in physical memory and requests the operating system to fetch the instruction. I want you to calculate the file-offset of the instruction with fake address 0x00000000004000b3. It is present at a file-offset of 3. The OS fetches the second instruction **4c-8d-34-25-e8-00-60-00**(lea r14, 0x6000e8) and loads it into physical memory. The operating system updates the table entry. Note that the second instruction replaces the first instruction in the physical memory. The processor executes the second instruction. The table and physical memory looks like this.

| Physical Address | Fake Address           |
|------------------|------------------------|
|       0x00       | 0x00000000004000b3     |
|       0x0f       |    --------            |
</p>

```
-------------------------------------------------------------------------------
0x00: 4c 8d 34 25 e8 00 60 00 00 00 00 00 00 00 00  ; lea r14, 0x6000e8
0x0f: 00 00 00 00 00 00 00 00                       ; No data element
-------------------------------------------------------------------------------
```

<p>

The processor knows that the second instruction is 8 bytes long. It calculates the fake address of the next instruction: (0x00000000004000b3 + 8) = 0x00000000004000bb. 0x00000000004000bb is loaded into **rip**. This instruction is interesting because it has a data access. As usual, the instruction is placed in the main memory by the operating system and the table is updated. The processor tries to execute it. The instruction is **4c-89-34-25-f5-00-60-00**(mov qword[0x6000f5], r14) - contents of register r14 is written into the data element at fake address 0x6000e5. This is how the table looks and physical memory looks like now.

| Physical Address | Fake Address           |
|------------------|------------------------|
|       0x00       | 0x00000000004000bb     |
|       0x0f       |    --------            |
</p>

```
-------------------------------------------------------------------------------
0x00: 4c 89 34 25 f5 00 60 00 00 00 00 00 00 00 ; mov qword[0x6000f5], r14
0x0f: 00 00 00 00 00 00 00 00                   ; No data element
-------------------------------------------------------------------------------
```

<p>
The processor goes ahead and executes the instruction. It tries to access the data element and checks the mapping table to see if if the data element at fake address 0x6000f5 is present in physical memory. It comes to know that the data element is **not** present in physical memory. It stops the execution temporarily and requests the operating system to fetch it from secondary storage and put it into physical memory. How will you fetch a data element at fake address 0x6000f5? You know that the data's starting fake address is 0x6000e8. This means the data element we want is present at an offset of (0x6000f5-0x6000e8) = 13. You also know that data is present at a file-offset of 0x36. With that, you can calculate the file-offset of the required data element as (0x36 + 13) = 0x43. The operating system fetches 8 bytes present at the file-offset 0x43 and loads it into physical memory at 0x0f and it updates the mapping table entry. The table and memory looks like this.

| Physical Address | Fake Address           |
|------------------|------------------------|
|       0x00       | 0x00000000004000bb     |
|       0x0f       | 0x00000000006000f5     |
</p>

```
-------------------------------------------------------------------------------
0x00: 4c 89 34 25 f5 00 60 00 00 00 00 00 00 00 ; mov qword[0x6000f5], r14
0x0f: 00 00 00 00 00 00 00 00                   ; 8 bytes at 0x6000f5 are 0s
-------------------------------------------------------------------------------
```

<p>
Now that the data is present in main memory, the processor goes ahead and executes the instruction. This instruction copies contents of register r14(which is 0x00000000006000e8) into the data element. After the instruction is executed, this is how the user-space memory looks like.
</p>

```
-------------------------------------------------------------------------------
0x00: 4c 89 34 25 f5 00 60 00 00 00 00 00 00 00 ; mov qword[0x6000f5], r14
0x0f: 00 00 00 00 00 60 00 e8                   ; After execution
-------------------------------------------------------------------------------
```

<p>
That was a data-write. Later when another data element is accessed by the processor, this data is overwritten. But when the processor changes the value of a data element, this change in value should be reflected in the secondary storage right? Whenever data-write happens, it needs to be **written back** to the secondary storage. How can this write-back mechanism be implemented? It is quite simple. We can have a single bit of information which tells us if the data has changed by the processor. If it is changed(or written by the processor), let us set that bit to 1, else it will remain 0. The table has a new column now. Before the processor wrote into the data element, the table looked like this.

| Physical Address | Fake Address           |  Changed bit      |
|------------------|------------------------|-------------------|
|       0x00       | 0x00000000004000bb     |     FALSE         |
|       0x0f       | 0x00000000006000f5     |     FALSE         |

The processor wrote the value 0x00000000006000e8 into the data element. The processor must update the data element's Changed-bit. After updating,

| Physical Address | Fake Address           |  Changed bit      |
|------------------|------------------------|-------------------|
|       0x00       | 0x00000000004000bb     |     FALSE         |
|       0x0f       | 0x00000000006000f5     |     TRUE          |

The processor computes the fake address of the next instruction: (0x00000000004000bb + 8) = 0x00000000004000c3. It updates **rip** and checks if that fake address present in the table. It is not present, so it requests the operating system to fetch the instruction at fake address 0x00000000004000c3. The instruction is **4c-8b-34-25-f5-00-60-00**(mov r14,qword[0x6000f5]). It is loaded into physical memory and processor tries to execute it. This instruction has a data access. It needs 8 bytes present at the fake address 0x00000000006000f5. It checks the table to see if physical memory has the data element at fake address 0x00000000006000f5 and yes, it does. The processor goes ahead and reads the value and executes the instruction. The table and physical memory looks like this,

| Physical Address | Fake Address           |  Changed bit      |
|------------------|------------------------|-------------------|
|       0x00       | 0x00000000004000c3     |     FALSE         |
|       0x0f       | 0x00000000006000f5     |     TRUE          |
</p>

```
-------------------------------------------------------------------------------
0x00: 4c 8b 34 25 f5 00 60 00 00 00 00 00 00 00 ; mov r14, qword[0x6000f5]
0x0f: 00 00 00 00 00 60 00 e8                   ; Updated by third instruction
-------------------------------------------------------------------------------
```

<p>
Processor computes the fake address of the next instruction: (0x00000000004000c3 + 8) = 0x00000000004000cb. The instruction is **45-8a-3e**(mov r15b, byte[r14]). Processor updates **rip** and checks if the instruction is present in main memory. It sees that the instruction is not present. It requests the operating system to fetch the instruction and place it in physical memory. Once fetching is done, this is how the table and main-memory looks like.

| Physical Address | Fake Address           |  Changed bit      |
|------------------|------------------------|-------------------|
|       0x00       | 0x00000000004000cb     |     FALSE         |
|       0x0f       | 0x00000000006000f5     |     TRUE          |
</p>

```
-------------------------------------------------------------------------------
0x00: 45 8a 3e 00 00 00 00 00 00 00 00 00 00 00 ; mov r15b, byte[r14]
0x0f: 00 00 00 00 00 60 00 e8                   ; Updated by third instruction
-------------------------------------------------------------------------------
```

<p>
The processor goes ahead and executes the instruction. But note that the instruction requests access to a byte of data at r14(ie., at fake address 0x6000e8). It checks the table and sees that data at fake address 0x6000e8 is not present in the RAM. It stops the execution and requests the operating system to fetch the data element at fake address 0x6000e8. The operating system first takes a look at the table. It sees that the data has been changed by some old instruction. This means that the operating system needs to write back the data element back to its respective fake address(here 0x6000f5) and only then load the new data element. The operating system does the write back and brings in the new data element. Now, the table and memory looks like the following.

| Physical Address | Fake Address           |  Changed bit      |
|------------------|------------------------|-------------------|
|       0x00       | 0x00000000004000cb     |     FALSE         |
|       0x0f       | 0x00000000006000e8     |     FALSE         |
</p>

```
-------------------------------------------------------------------------------
0x00: 45 8a 3e 00 00 00 00 00 00 00 00 00 00 00 ; mov r15b, byte[r14]
0x0f: 48 65 6c 6c 6f 20 57 6f                   ; 8 bytes at fake Address 0x6000e8
-------------------------------------------------------------------------------
```

<p>
The operating system by default fetches 8 bytes. Depending on the instruction, the processor decides how many bytes it should access. In this case, the instruction is ```mov r15b, byte[r14]``` => processor reads only the first byte and copies it into r15b.

With that, I will stop here. I request you to continue this simulation till the complete program is executed.

Look at how the write-back mechanism works. The operating system does not write back the changed data element the moment it was changed by the processor. The operating system waits for as long as possible - till another data element needs to be fetched. Only when it is high time, it writes back the changed data element. The operating system is being **lazy** and non-proactive here. It does work only when it needs to. You will find this lazy concept in a lot of places.

A few important observations. 

1. Before the above exercise, the processor was a simple one. Text and data both were present in physical memory. The processor used the **physical address** and execute the program - straight-forward. But in the above exercise, the processor did **not** work on physical addresses, instead it used the **fake addresses**. The CPU used the fake addresses to request instructions and data. 
2. The CPU is still aware of physical memory and physical addresses. Even now, it has to fetch instruction and data from physical memory itself. It uses the mapping table to translate the fake address into the physical address, put the physical address in the address bus and fetch the instruction/data. Just that from a program execution perspective, physical addresses are not relevant.
3. Note that there was absolutely no relation between the fake addresses and physical addresses.
4. Observe that the only constraint on the fake address range is the CPU type. All the fake addresses are 64-bits(8 bytes) because a 64-bit CPU was used. This means the largest usable fake address is 0xffffffffffffffff(2^64-1). 
5. The text and data can be placed at any fake address and the program would work without any problems. We placed text at 0x4000b0 and 0x6000e8 and we were able to simulate the program properly. Similarly, with any other addresses, it can be run.
6. What can you tell about the maximum size of a process which can be executed? The program we ran above was 75 bytes big - 54 bytes of text and 21 bytes of data. 23 bytes of physical memory was given for program execution. This means that we were able to execute a program whose size is greater than the size of available physical memory for program execution. Earlier, we had a system with 64-bit CPU and 3GiB of user-space memory. The process size was constrained by the amount of physical memory available. On that system, the size of the largest process is 3GiB and nothing more could be handled. But look at it now. We have a 64-bit CPU and 23 bytes in user-space memory. With the old methods, we could not have run a process whose size is more than 23 bytes. But with a few (significant) changes to the CPU, a couple of data structures and changes in the memory manager, we were able to run a 75-byte process on it. What is the size of the largest process that can be run on our new system? You can see that the program size is now constrained by the **size of the fake address range**. Because it is a 64-bit CPU, the size of fake address range is 2^64 or 18,446,744,073,709,551,616 bytes which is unbelievably big! Virtually, there is no limit on the process size.

Our original problem to solve was this: Can we run a program whose size is larger than the available user-space physical memory? We now have a definitive answer for that. Yes, we can run large processes on systems with small physical memories, the size of the processes is constrained by the size of the fake address range.

Before we move forward, let us put whatever we discussed so far in a formal manner.

There are 2 address spaces now. **Physical Address Space** and **Fake Address Space**.

**1**. **Physical Address Space**: As discussed on the previous chapter, the address bus length decides the size of the physical address space. Devices like ROM, RAM, Keyboard etc., are part of the physical address space. All the data elements in each of these devices is given a unique physical address. A physical address can either be unused or it can be used to access some data element of some device. 

* The current-day Intel 64-bit CPUs have a 64-bit address bus - the physical address space has 2^64 addresses. The following is the physical address space of my system.
</p>

```
-------------------------------------------------------------------------------
chapter3$ sudo cat /proc/iomem
00000000-00000fff : Reserved
00001000-00057fff : System RAM
00058000-00058fff : Reserved
00059000-0005efff : System RAM
0005f000-0005ffff : Reserved
00060000-0009efff : System RAM
0009f000-0009ffff : Reserved
000a0000-000bffff : PCI Bus 0000:00
000c0000-000cfdff : Video ROM
000d0000-000d0fff : Adapter ROM
000f0000-000fffff : System ROM
00100000-bb96b017 : System RAM
bb96b018-bb978857 : System RAM
bb978858-bb979017 : System RAM
bb979018-bb989057 : System RAM
bb989058-bd336fff : System RAM
bd337000-bd7b2fff : Reserved
bd7b3000-cd1f3fff : System RAM
bda00000-be800e80 : Kernel code
be800e81-bf2507bf : Kernel data
bf50b000-bf9fffff : Kernel bss
cd1f4000-cd238fff : Reserved
cd239000-d9dbafff : System RAM
d9dbb000-d9ef2fff : Reserved
d9ef3000-d9f15fff : ACPI Tables
d9f16000-da686fff : System RAM
da687000-dade2fff : ACPI Non-volatile Storage
dade3000-dbafefff : Reserved
dbaff000-dbafffff : System RAM
dbb00000-dbffffff : RAM buffer
dd000000-df7fffff : Reserved
dd800000-df7fffff : Graphics Stolen Memory
df800000-feafffff : PCI Bus 0000:00
df800000-df9fffff : PCI Bus 0000:01
dfa00000-dfbfffff : PCI Bus 0000:01
dfc00000-dfdfffff : PCI Bus 0000:02
dfe00000-dfffffff : PCI Bus 0000:03
e0000000-efffffff : 0000:00:02.0
f0000000-f01fffff : PCI Bus 0000:04
f0200000-f03fffff : PCI Bus 0000:04
f6000000-f6ffffff : 0000:00:02.0
f7000000-f70fffff : PCI Bus 0000:03
f7000000-f7001fff : 0000:03:00.0
f7000000-f7001fff : iwlwifi
f7100000-f71fffff : PCI Bus 0000:02
f7100000-f7103fff : 0000:02:00.0
f7104000-f7104fff : 0000:02:00.0
f7104000-f7104fff : r8169
f7200000-f720ffff : 0000:00:14.0
f7200000-f720ffff : xhci-hcd
f7210000-f7213fff : 0000:00:1b.0
f7210000-f7213fff : ICH HD audio
f7214000-f7217fff : 0000:00:03.0
f7214000-f7217fff : ICH HD audio
f7218000-f72180ff : 0000:00:1f.3
f7219000-f72197ff : 0000:00:1f.2
f7219000-f72197ff : ahci
f721a000-f721a3ff : 0000:00:1d.0
f721a000-f721a3ff : ehci_hcd
f721c000-f721c01f : 0000:00:16.0
f721c000-f721c01f : mei_me
f7fe0000-f7feffff : pnp 00:05
f7ff0000-f7ffffff : pnp 00:05
f8000000-fbffffff : PCI MMCONFIG 0000 [bus 00-3f]
f8000000-fbffffff : Reserved
f8000000-fbffffff : pnp 00:05
fec00000-fec00fff : Reserved
fec00000-fec003ff : IOAPIC 0
fed00000-fed03fff : Reserved
fed00000-fed003ff : HPET 0
fed00000-fed003ff : PNP0103:00
fed10000-fed17fff : pnp 00:05
fed18000-fed18fff : pnp 00:05
fed19000-fed19fff : pnp 00:05
fed1c000-fed1ffff : Reserved
fed1c000-fed1ffff : pnp 00:05
fed1f410-fed1f414 : iTCO_wdt.0.auto
fed1f800-fed1f9ff : intel-spi
fed20000-fed3ffff : pnp 00:05
fed45000-fed8ffff : pnp 00:05
fed90000-fed90fff : dmar0
fed91000-fed91fff : dmar1
fee00000-fee00fff : Local APIC
fee00000-fee00fff : Reserved
ff000000-ffffffff : Reserved
ff000000-ffffffff : INT0800:00
ff000000-ffffffff : pnp 00:05
100000000-11f7fffff : System RAM
11f800000-11fffffff : RAM buffer
-------------------------------------------------------------------------------
```

<p>
* It starts with physical address 0x0000000000000000 and goes till 0x000000011fffffff. 4,831,838,208 (0x120000000) physical addresses have been used. But the highest address is 0xffffffffffffffff(2^64-1). It can still accomodate a lot of physical devices.

* Consider a 32-bit Intel System. It has a 32-bit address bus. The physical address space is 4GiB(2^32) in size. Address range is 0x00000000 to 0xffffffff. I am running a 32-bit ubuntu virtual machine. The following is its physical address space.
</p>

```
-------------------------------------------------------------------------------
vm/chapter3$ sudo cat /proc/iomem
00000000-00000fff : reserved
00001000-0009fbff : System RAM
0009fc00-0009ffff : reserved
000a0000-000bffff : PCI Bus 0000:00
    000a0000-000bffff : Video RAM area
000c0000-000c7fff : Video ROM
000e2000-000ef3ff : Adapter ROM
000f0000-000fffff : reserved
    000f0000-000fffff : System ROM
00100000-3ffeffff : System  RAM
    01000000-017e7e53 : Kernel code
    017e7e53-01bcad3f : Kernel data
    01cbe000-01d85fff : Kernel bss
3fff0000-3fffffff : ACPI Tables
40000000-fdffffff : PCI Bus 0000:00
    e0000000-e0ffffff : 0000:00:02.0
    f0000000-f001ffff : 0000:00:03.0
        f0000000-f001ffff : e1000
    f0400000-f07fffff : 0000:00:04.0
        f0400000-f07fffff : vboxguest
    f0800000-f0803fff : 0000:00:04.0
    f0804000-f0804fff : 0000:00:06.0
        f0804000-f0804fff : ohci_hcd
    f0806000-f0807fff : 0000:00:0d.0
        f0806000-f0807fff : ahci
fec00000-fec00fff : reserved
fee00000-fee00fff : Local APIC
    fee00000-fee00fff : reserved
fffc0000-ffffffff : reserved
-------------------------------------------------------------------------------
```

If you need more details about physical address space, about how the processor can read/write into a particular physical address and more, please refer to [previous chapter's init](#Init2.1).

**2**. **Fake Address Space**: This is the concept which helped us run processes with size greater than available usable main memory. The various sections of a program are laid down by the linker on this fake address space. This address space is not real aka it is **virtual**. It is also known as **Virtual Address Space** and the fake addresses are known as **Virtual Addresses**. The current day 64-bit Intel CPU supports 64-bit virtual address space. This means that a process can be 2^64 bytes big. This was an short introduction to Virtual Address Space. The programmer(and tools like linker) are given an illusion that virtually infinite memory is present in the system because a huge address space is given for the program. This is called **Virtual Memory**. Its called virtual memory because so much memory does not exist in the system, but the operating system presents itself as if so much memory is available. Because of Virtual Address Space, the linker does not have to care about the addresses given to the program. It can give any address to it irrespective of the amount of physical memory present - this is one of biggest advantages of Virtual Memory. Virtual Address Space, Virtual Memory are all unbelievably awesome concepts. We will slowly discuss all of them as we move forward.

We solved the problem of running programs with size greater than the available user-space memory. But the solution came at some cost and complexity. The fundamental nature of CPU was changed - it was working with real, physical addresses but now it is working with virtual addresses. For every instruction, every data element, the processor had to stop and request the operating system for the instruction(or data) and the operating system accesses the secondary storage and fetches the instruction(or data). For every instruction(or data), the processor had to refer to the new data structure, the mapping table. We currently have a costly solution. Either we need to reduce the cost or discard the solution and come up with something else.

The hand-simulation we did was hypothetical. A practical system will never have memory as small as 23 bytes. Let us consider a practical system and see if the cost can be reduced. Consider a system with 4GiB RAM and a 64-bit CPU which supports virtual addresses. Assume that 3GiB of RAM is given to run processes. Assume that the physical addresses 0x00-00-00-00-00-00-00-00 to 0x00-00-00-00-bf-ff-ff-ff can be used to refer to the 3GiB of user-space RAM. Because it is a 64-bit CPU which can understand virtual addresses, the size of virtual address space is 2^64(0x0000000000000000-0xffffffffffffffff). Task: How do we run a 5GiB(2GiB of text, 3GiB of data) large process? Let us assume that text starts from virtual address 0x400000. Because its size is 2GiB, virtual address range for text = 0x0000000000400000 - 0x00000000803fffff. Data starts at 0x0000000080400000 and goes till 0x00000001403fffff. Note that text and data don't have to be contiguous. We have kept them contiguous for the sake of simplicity.  

With 3GiB of RAM in hand, we don't have to execute the bring in one instruction at a time from secondary storage to RAM and then execute it. Because there is lot of RAM available, parts of text and data can be fetched in before hand in bulk so that we can save time on fetching instructions/data from secondary storage again and again. But note that the fetch/write-back will not stop because the process is larger than the available user-space memory. To start with, let us place first 1GiB of text and 2GiB of data in physical memory. Once this is placed, we need to have a mapping table for all the valid virtual addresses. At any point, the table should know whether a virtual address is present in main memory or in secondary storage - just like the table we used in the previous example. But there, it was only a matter of 1 instruction and 1 data element. Here, there are way more physical addresses. So we can have a table like this. Note that first 1GiB of text and first 2GiB of data are in RAM.

| Virtual Address    | Present in RAM | Physical Address   |
|--------------------|----------------|--------------------|
| 0x0000000000400000 |     YES        | 0x0000000000000000 |
| 0x0000000000400001 |     YES        | 0x0000000000000001 |
| .                  |     .          |      .             |
| .                  |     .          |      .             |             
| .                  |     .          |      .             |
| 0x00000000403fffff |     YES        | 0x000000003fffffff |
| 0x0000000040400000 |     NO         |      --------      |
| 0x0000000040400001 |     NO         |      --------      |
| .                  |                |                    |
| .                  |                |                    |
| .                  |                |                    |
| 0x00000000803fffff |     NO         |      --------      |
| 0x0000000080400000 |     YES        | 0x0000000040000000 |
| 0x0000000080400001 |     YES        | 0x0000000040000001 |
| .                  |     .          |      .             |
| .                  |     .          |      .             |
| .                  |     .          |      .             |
| 0x00000001003fffff |     YES        | 0x00000000bfffffff |
| 0x0000000100400000 |     NO         |      --------      |
| 0x0000000100400001 |     NO         |      --------      |
| .                  |     .          |      .             |
| .                  |     .          |      .             |
| .                  |     .          |      .             |
| 0x00000001403fffff |     NO         |      --------      |

Slowly go through the table. It has entries for every single valid virtual address. Note that only valid virtual addresses are 0x0000000000400000 - 0x00000001403fffff because text and data fall into that range of virtual addresses. Other virtual addresses are invalid.

Let us start the execution. The first instruction is at the virtual address 0x400000. The processor checks the mapping table to see if it is present. The byte at 0x0000000000400000 is present at physical address 0x0000000000000000. Assuming that the instruction is 8 bytes, the processor checks if all 8 bytes are present in main memory. It confirms that it is present. It executes the instruction.

Assume that a large function is about to be executed. Assume that its first instruction is 2 bytes long and is present at virtual address 0x00000000403ffffe. The processor checks and executes the instruction. It calculates the virtual address of the next instruction: (0x00000000403ffffe + 2) = 0x0000000100400000. The processor refers the table and comes to know that the next instruction's bytes are not present in the physical memory. Now, the processor requests the operating system to fetch the instruction at virtual address 0x0000000100400000 and put it in physical memory. Assume that the next instruction is placed at the physical address 0x0000000000000000. Assuming that the next instruction is 4 bytes in size, this is how the table would look like now. Only the updated entries are put below.

| Virtual Address    | Present in RAM | Physical Address   |
|--------------------|----------------|--------------------|
| 0x0000000000400000 |     NO         |      --------      |
| 0x0000000000400001 |     NO         |      --------      |
| 0x0000000000400002 |     NO         |      --------      |
| 0x0000000000400003 |     NO         |      --------      |
| 0x0000000000400004 |     YES        | 0x0000000000000004 |
| 0x0000000000400005 |     YES        | 0x0000000000000005 |
| 0x0000000000400006 |     YES        | 0x0000000000000006 |
| 0x0000000000400007 |     YES        | 0x0000000000000007 |
| .                  |     .          |      .             |
| .                  |     .          |      .             |             
| .                  |     .          |      .             |
| 0x00000000403fffff |     YES        | 0x000000003fffffff |
| 0x0000000040400000 |     YES        | 0x0000000000000000 |
| 0x0000000040400001 |     YES        | 0x0000000000000001 |
| 0x0000000040400002 |     YES        | 0x0000000000000002 |
| 0x0000000040400003 |     YES        | 0x0000000000000003 |
| 0x0000000040400004 |     NO         |      --------      |
| .                  |                |                    |

The next instruction is in physical memory, the processor executes it. Question is, on what basis did we overwrite the bytes at physical address 0x0000000000000000? We could have overwritten some other bytes too?

The processor gets the virtual address of the next instruction: (0x0000000040400000 + 4) = 0x0000000040400004. It checks the table and sees that those bytes are not present in the memory. Request the operating system and wait till the bytes are loaded at some physical address. This is the case with rest of the function's instructions. Even though we have enough physical memory to load (say) all the instructions of the function in bulk, we are not doing it. We are doing it the old-fashioned way - fetching one instruction at a time which is very costly. Similarly, assume that a large array needs to be traversed and only the first byte is present in the physical RAM. For the rest of the array, we being in element by element. We changed the initial load mechanism by loading lots of text and data. But once the process is running, we are still using the one element/request mechanism - which is inefficient. We need to be able to bring in lots of bytes at a time so that the cost of fetching reduces. Over the years, a lot of programs have been written and their behavior has been studied in great detail. One of the characteristics which will help us is the memory access patterns of a program. Two observations about memory accesses of programs are

1. If a program accesses(reads from/writes into) a memory location, it is highly likely that the program will access the same memory location again.

2. If a program accesses a memory location, it is highly likely that the program will access its neighboring memory locations.

Read these observations again. They reinforce our examples in the above paragraph. When the processor accesses an instruction, it is highly likely that the instruction next to it will be accessed next. If an array element A[i] is accessed, it is highly likely that A[i-1] or A[i+1] gets accessed next. These are not rules, but just observations and are generally true. There are definitely exceptions to it. If the instruction is a **jmp** or **call**, then our observation mostly fails because it is highly unlikely that the function or label is the next instruction(there is an exception to this too :P). Consider the iterator used to access that array. It will be accessed again and again either to read some array element or for increment/decrement etc.,

Let us talk about the mapping table. Currently, every valid virtual address has an entry. This requires a significant part of the operating system's memory.

Based on the above arguments, we need to modify our memory manager. Instead of fetching instruction(or data) one by one, let us swap in or swap out a large number of bytes. Break the 3GiB of user-space memory into pieces of X bytes each. It can be viewed as a huge array where each array element is X bytes in size.
</p>

```
-----------------------------------------------------------------------------------------
|--------------------------------------------------------------------------------|
|   X    |   X    |   X    |  ...   |  ...   |  ...   |   X    |   X    |   X    |
| bytes  | bytes  | bytes  |        |        |        | bytes  | bytes  | bytes  |
|--------------------------------------------------------------------------------|
^                                                                                ^
|                                                                                |
0x0                                                                             3GiB-1
------------------------------------------------------------------------------------Fig-9
```

<p>
Each of these pieces is called a **frame**. How many such frames are present in 3GiB user-space memory? We will have (3GiB / X) number of frames. Assuming X to be 4KiB(4096 bytes) , there are 786432(or 0xc0000) frames. We can give a unique identifier to each frame - 0x00000 to 0xbffff. This is how it looks now.
</p>

```
-----------------------------------------------------------------------------------------
|--------------------------------------------------------------------------------|
| 4096   | 4096   | 4096   |  ...   |  ...   |  ...   | 4096   | 4096   | 4096   |
| bytes  | bytes  | bytes  |        |        |        | bytes  | bytes  | bytes  |
|--------------------------------------------------------------------------------|
^        ^        ^                                   ^        ^        ^                  
|        |        |                                   |        |        |          
0x00000  0x00001  0x00002   . . . . . . . . . . . . . 0xbfffd  0xbfffe  0xbffff                  
-----------------------------------------------------------------------------------Fig-10
```

<p>
In the exact same way, let us break the virtual address space into pieces. Call these **virtual pages**. Idea is that each virtual page will get copied onto a physical frame. Size of the virtual address space is 2^64. It can be divided into 4,503,599,627,370,496(or 0x10000000000000) pages. But the virtual pages which containing parts of our program are relevant to us. Given a virtual address, how do you calculate the page it is in? You simply divide the virtual address by size of the virtual page. Take 0x0000000000400000 for example. It belongs to virtual page number 1024(or 0x0000000000400). It is as simple as that. A virtual address / physical address here is 64-bits long. With a page/frame size of 4KiB(4096 bytes), what is the size of the page/frame ID? Its 52-bits in size.

Let us list all the valid virtual pages now. First valid virtual address is 0x0000000000400000 => page ID is 0x0000000000400. Last valid virtual address is 0x00000001403fffff => page ID is 0x000000001403ff. The virtual page IDs of our program 0x0000000000400 to 0x00000001403ff.

With that, let us discuss how transactions happen. In the simulation, individual instructions/data elements were fetched. Later, byte was the unit of transaction. After some discussion, now we a page. A page is the unit of transaction. A page gets fetched from secondary storage into physical memory and is copied onto some frame, a page gets swapped out of physical memory making space for some other page. Here, the page size is 4096 bytes. Just to make it clear, we have 3 things here: **Physical frame** is part of the physical memory, **Virtual page** is part of the virtual address space and **page** is a unit of measurement, the unit of transaction. Before the program executes, we put a few text virtual pages and data virtual pages onto physical memory. We put 1GiB of text and 2GiB of data just like before. But remember that everything will be in terms of pages, even the mapping table. We will know which virtual page is mapped to which physical frame if it is present. This is how the mapping table looks like.

| Virtual Page number | Present in RAM | Physical Frame number |
|---------------------|----------------|-----------------------|
| 0x0000000000400     |      YES       |   0x0000000000001     |
| 0x0000000000401     |      YES       |   0x0000000000002     |
| .                   |      .         |   .                   |
| .                   |      .         |   .                   |
| .                   |      .         |   .                   |
| 0x00000000403ff     |      YES       |   0x000000003ffff     |
| 0x0000000040400     |      NO        |         -----         |
| 0x0000000040401     |      NO        |         -----         |
| .                   |      .         |         .             |
| .                   |      .         |         .             |
| .                   |      .         |         .             |
| 0x00000000803ff     |      NO        |         -----         |
| 0x0000000080400     |      YES       |   0x0000000040000     |
| 0x0000000080401     |      YES       |   0x0000000040001     |
| .                   |      .         |   .                   |   
| .                   |      .         |   .                   |
| .                   |      .         |   .                   |
| 0x00000001003ff     |      YES       |   0x00000000bffff     |
| 0x0000000100400     |      NO        |         -----         |
| 0x0000000100400     |      NO        |         -----         |
| .                   |      .         |   .                   |
| .                   |      .         |   .                   |
| .                   |      .         |   .                   |
| 0x00000001403ff     |      NO        |   .                   |

You can observe that the number of entries in the table have reduced and number of bytes needed to store a single entry has also reduced.

With that, we are ready to go. If the processor wants to check if a virtual address is present in memory, then what should it do? It needs to find the virtual page that virtual address is in and then check the mapping table. You know how to get the virtual page from a virtual number. From here, program execution will be a bit faster and amount of metadata(like tha table) takes lesser space than before.

There is one question left: When a virtual page needs to be swapped in, some other page needs to get swapped out. Of all the physical frames, which one should we swap out? Think about it.

With that, let us end this thread.

We had 2 discussion threads: The first thread introduced the concept of time sharing and multi-tasking, swapping etc., The above thread introduced the concept of Virtual Address Space, frames, pages, mapping table etc., Each of the threads tried to solve 2 different problems. 

I want to combine these two threads now.

We ended the first thread by calculating the cost of time-sharing vs straight-forward ordering. The following is what we found when there were 5 programs to run. Note that in the first thread, we were using simple processors which worked on physical addresses and programs' size was not greater than user-space memory.

1. Straight-forward Ordering: 
    * Total runtime = (T1 + T2 + T3 + T4 + T5) + [Ceil(sizeof(Pi) / X ) * T_Load][i=1 to 5] + [Ceil(sizeof(Pi) / X) * T_Clean][i = 1 to 5]

2. Round-Robin ordering
    * Total runtime = (T1 + T2 + T3 + T4 + T5) + [Ceil(Ti/T) * Ceil(sizeof(Pi)/X) * T_Load][i = 1 to 5] + [Ceil(Ti/T) * Ceil(sizeof(Pi)/X) * T_Clean][i = 1 to 5].

Before we start any discussion, let us assume a couple of things. Consider a 3GiB user-space memory and a 64-bit CPU. The following describes each of the 5 processes.

1. P1: 512MiB, 384MiB text, 128MiB of data
2. P2: 1GiB, 256MiB text, 768MiB of data
3. P3: 1.5GiB, 1GiB of text, 512MiB of data
4. P4: 2GiB, 512MiB of text, 1.5GiB of data
5. P5: 2.5GiB, 1.5GiB of text, 1GiB of data

Text will always start at physical address 0x0000000000000000 and data will follow the text. With that, we are ready to go.

Total runtime of round-robin ordering is significantly higher because of the sheer loads and cleanups done. Time-sharing is a very useful concept, but at the moment a costly one. We need to reduce its cost. Cost is direct related to the number of loads and cleanups. We reduce the number of loads and cleanups, we reduce the cost. How do we do it? Try taking hints from the second thread.

The operating system starts with executing P1. Currently, complete P1 will be copied to main memory. Later, complete P1 is removed once its T units of time is over and again when its turn comes, complete P1 is copied back. When P1 starts its execution, the first instruction is accessed. It is highly likely that it accesses the first few instructions when the execution has just begun. For data, it is quite unpredictable. One idea is to break the complete program into pages and load them on a **demand** basis. When the program is about to run, let us load the first few pages from text because it is highly likely that they will be accessed. Data access is unpredictable. We have 2 choices - We can be conservative about loading data pages and not load any data pages in the beginning. Once the CPU accesses a data element, let us load the page containing that data element. Or load a couple of data pages in the beginning. Assuming page size to be 4KiB(4096 bytes), exactly how many text pages should we load? Let us calculate a rough number. Assuming the processor can execute 1 billion instructions per second(1,000,000,000 instructions/second), T to be 5ms, number of instructions executed in 5ms = (1,000,000,000 instructions/second * 5ms) = 5,000,000 instructions. Size of a x64 instruction can be anywhere from 1 byte to 15 bytes. We should ideally take the weighted average to get the average instruction size, but to calculate the weighted average, we need the actual program and instructions. We don't have the program. Let us simply take the average. Average instruction size is (1 + 15)/2 = 8 bytes/instruction. Number of bytes to load initially = (5,000,000 instructions * 8 bytes/instruction) = 40,000,000 bytes. With page-size = 4096 bytes/page, 40,000,000 bytes is equivalent to (40,000,000 bytes/(4096 bytes/page) = 9765.625 pages. Let us round off and load the first 10,000 text pages. Along with that, let us load 10,000 data pages too. There is no need for a mapping table here. We are not mapping addresses. We just need to know if a page is present or not. Let us have set of page numbers of the pages present in main memory. That should suffice. Size of P1 is 512MiB which amounts to 131072(or 0x20000) pages. Out of those pages, text falls under pages 0x0000000000001 to 0x0000000017fff and data under 0x0000000018000 to 0x000000001ffff. The following is the list.

| List of page numbers |
|----------------------|
| 0x0000000000000      |
| 0x0000000000001      |
| 0x0000000000002      |
| .                    |
| .                    |
| .                    |
| 0x000000000270f      |
| 0x0000000018000      |
| 0x0000000018001      |
| .                    |
| .                    |
| .                    |
| 0x000000001a70f      |

At the end of P1's 5ms, let there be 20,500 pages. Now, it is P2's turn. Same analysis can be done on P2 and 20,000(10,000 text and 10,000 data) pages can be loaded. Before loading, all of P1's pages should be swapped back to secondary storage and the operating system should preserve P1's set of page numbers so that when its P1's turn again, the operating system knows what pages to bring back. All the physical frames where P1's pages were present needs to be cleaned up. Similar to P1, P2 also gets executed. Assume that in P2's 5ms, there were 21,000 pages in memory. Similar number of pages were loaded during P3, P4 and P5's execution. When its P1's turn again, the operating system takes a look at at P1's set of page numbers and loads them. Obviously, the operating system will have to load newer pages as the process runs. As time progresses, number of pages of a process in memory steadily increases. It is highly likely that many of these pages are never used - because the new pages are used and not the old ones. This means that the number of pages increases steadily. This unnecessarily increases the load and cleanup time. This means, the operating system should keep track of what pages are being accessed, how frequently these pages are being accessed. The unused ones, seldom used pages can be swapped out of the main memory which will contribute in reducing the load and cleanup time. Note that the operating system should not carelessly swap out frequenty used, **hot** pages - this will prove to be very costly. It should strategically swap out unused, **cold** pages. 

Assume that the above concept is implemented in the operating system. Say an average of 20,000 pages were always present in the main memory. You can easily calculate the load-cleanup cost. 20,000 pages amounts to 81,920,000 bytes = 78.125MiB. Compare loading and cleaning 78.1215MiB after every process with loading and cleaning complete processes with sizes 0.5GiB, 1GiB, 1.5GiB, 2GiB and 2.5GiB. Cost is definitely reduced. This is good. But observe the average amount of memory used. There is 3GiB of user-space memory available and about 80MiB of it is being used. This is an case of extreme underutilization of resources. What can be done about this?

Why is the cleanup done? It is done so that a process's data remains private and no other processes can get hold of its data. Currently, relevant pages of a process is loaded, the process runs for T units of time, it is cleaned up and the relevant pages of the next process is loaded. These relevant pages are placed in the same physical memory. The same physical memory is cleaned up when its the next process's turn. Instead of that, we can place relevant pages of different processes in different parts of physical memory. 3GiB user-space memory means we have 786432(0xc0000) frames in hand. If we allocate 20,000 frames for each process, a total of 100,000 frames will be used. Instead of cleanup, the operating system should protect a process's frames against other processes. No one other than that process should be access its frames. This way, we won't even need cleaning at the end of T units of time. All the relevant pages of all the processes will always be in memory - obviously once a particular process is done running, all its frames are cleaned up. In fact, more than 20,000 pages per process can be allocated now.

If this can be done, we can strike a balance between underutilization and the cost of loads and cleans. But with the current setup, do you think the above memory allocation can be done?

At the moment, our linker assumes that the complete user-space memory(3GiB: 0x0000000000000000 - 0x00000000bfffffff) is given to one single process and allots addresses. P1 is given addresses (0 to 512MiB-1). P2 is given addresses (0 to 1GiB-1) and so on. Unless each of these processes are copied at the linker-alloted addresses, they won't run properly. Eg: The linker would have given a data element the address 0x12345678 and instructions would be using this address to refer to that data element. When the program is about to run, that data element must be loaded at the physical address 0x12345678. If it is loaded at some other address, the programs won't run properly. This is exactly what will happen for P2, P3, P4 and P5. Each of these processes want physical memory from (0x00 to 512MiB-1). But with the above memory allocation, everything goes haywire. It is impractical to generate executables based on the unused physical memory left at that moment. The amount of unused physical memory obviously keeps changing - it is dynamic in nature. Programs should run irrespective of how much physical memory is allocated for it. Programs should run irrespective of where it is placed in the physical memory. The linker allotment of addresses should be independent of the availability of physical addresses. Do you know of any concept which will help us solve this problem?

Virtual Address Space and Virtual Memory can help us solve it. We used the concept of Virtual Memory to solve the problem of running large programs on systems with smaller main memory. It solved the problem are given 5 different physical memory allocations.and it also brought in a lot of other features.

We have 5 processes in hand, with the above mentioned physical memory allocation. Instead of thinking this as 1 single problem, let us break it into 5 problems - we need to run 5 individual processes.

Let us take up Process P1. Its size is 512MiB(384MiB text and 128MiB data). The linker is now aware of the huge 64-bit virtual address space it is given. It does not have to worry about any aspect of physical memory now. Assume that the linker starts the text at virtual address 0x400000 and data following it. Virtual address range of text is 0x0000000000400000 - 0x00000000183fffff(384MiB), range of data is 0x0000000018400000 - 0x00000000203fffff(128MiB). text falls under the virtual pages 0x0000000000400 - 0x00000000183ff(98304 virtual pages), data falls under the virtual pages 0x0000000018400 - 0x00000000203ff(32768 virtual pages). As planned, let us place 20,000 virtual pages into the main memory. Note that the allocation need not be contiguous ie., a process's frames don't have to adjacent to each other. They can be scattered across the physical memory. There are 786432 physical frames with us and we can choose any 20,000 frames and copy P1's 20,000 virtual pages onto them. Create a mapping table for P1. Once all this is done, P1 is ready for execution.

P1's T units of time is done. The operating system takes it out of execution and its P2's turn. Let us repeat whatever we did for P1. P2 has its own 64-bit virtual address space in which its text and data resides. The linker may have given P2's text and data the same virtual addresses given to P1's text and data. Virtual address range of P2's text is 0x0000000000400000 - 0x00000000103fffff(256MiB) and data range is 0x0000000010400000 - 0x00000000403fffff(768MiB). P2's text falls under the virtual pages 0x0000000000400 - 0x00000000103ff(65536 virtual pages), data falls under virtual pages 0x0000000010400 - 0x00000000403ff(196608 virtual pages). There are (786,432 - 20,000) = 726,432 physical frames available. Choose any 20,000 frames and copy P2's 20,000 virtual pages onto them. Create a new, different mapping table for P2. Once all this is done, P2 is ready for execution.

Similarly, P3, P4 and P5 can be run.

Let us summarize whatever we discussed so far.

The idea of pages and Virtual Memory helped in accomodating processes with size greater than the available physical memory. In the first thread, we discussed about time-sharing and the cost paid. To reduce the the cost, we started by cutting the program and physical memory into pages. It reduced the cost but ended in memory underutilization. We wanted to increase memory utilization. Using Virtual Address Space did the job. Every process has its own Virtual Address Space, its own mapping table.
</p>

## 3.2 Virtual Memory and Security { #VMandSecurity3.2}

<p>
All the programs we considered so far was divided into two parts - text and data. The text has pure machine code, data has variables, data structures. Because the text has instructions, the processor should be able to **read** it and then **execute** it. Basically the complete text should be readable and executable. Take data now. It has variables and data structures. The processor might have to read them, update(write into) them. There is no meaning in executing a data element. This means data should be readable and writable. These are basically security permissions - basically what can you do with a given section. 

When the linker generates the executable, it should also generate security permissions for each of these sections. Let us take *hello.s*, the program we used in the previous section. Let us assemble and link it and inspect the permissions.
</p>

```
-------------------------------------------------------------------------------
chapter3$ nasm hello.s -f elf64
chapter3$ ld hello.o -o hello
-------------------------------------------------------------------------------
```

<p>
As discussed in the [first chapter](#Chapter1), the executable's segments help in program execution. Let us check them out using readelf.
</p>

```
-------------------------------------------------------------------------------
chapter3$ readelf --segments hello

Elf file type is EXEC (Executable file)
Entry point 0x4000b0
There are 2 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x00000000000000e6 0x00000000000000e6  R E    0x200000
  LOAD           0x00000000000000e8 0x00000000006000e8 0x00000000006000e8
                 0x0000000000000015 0x0000000000000015  RW     0x200000

 Section to Segment mapping:
  Segment Sections...
   00     .text 
   01     .data 
-------------------------------------------------------------------------------
```

<p>
Look at the first segment. Its starts at the virtual address 0x00000000000400000. There is also physical address given which is the same as the virtual address. As discussed, the linker does not have any information about physical addresses. So it fills in the virtual address itself in the place of physical address. Take a look at the **Flags** entry. Its **RE** which stands for **Readable** and **Executable**. THe first segment is nothing but the .text section. The second is the .data section which has flags **RW** which stands for **Readable** and **Writable**.

How do we implement this now? The executable has information about the segments' permissions. When the program is run, the main memory containing these segments should have the corresponding permissions. The frames containing the text section should be Readable and Executable and the data section frames should be readable and writable. If you think about it, the permissions about a particular page can be stored in the mapping table itself. The following is the modified mapping table.

| Virtual Page number | Present in RAM | Physical Frame number |  Permissions |
|---------------------|----------------|-----------------------|--------------|
| 0x0000000000400     |      YES       |   0x0000000000001     |     R-X      |
| 0x0000000000401     |      YES       |   0x0000000000002     |     R-X      |
| .                   |      .         |   .                   |     .        |
| .                   |      .         |   .                   |     .        |
| .                   |      .         |   .                   |     .        |
| 0x00000000403ff     |      YES       |   0x000000003ffff     |     R-X      |
| 0x0000000040400     |      NO        |         -----         |     R-X      |
| 0x0000000040401     |      NO        |         -----         |     R-X      |
| .                   |      .         |         .             |     .        |
| .                   |      .         |         .             |     .        |
| .                   |      .         |         .             |     .        |
| 0x00000000803ff     |      NO        |         -----         |     R-X      |
| 0x0000000080400     |      YES       |   0x0000000040000     |     RW-      |
| 0x0000000080401     |      YES       |   0x0000000040001     |     RW-      | 
| .                   |      .         |   .                   |     .        |   
| .                   |      .         |   .                   |     .        |
| .                   |      .         |   .                   |     .        |
| 0x00000001003ff     |      YES       |   0x00000000bffff     |     RW-      |     
| 0x0000000100400     |      NO        |         -----         |     RW-      |
| 0x0000000100400     |      NO        |         -----         |     RW-      |
| .                   |      .         |   .                   |     .        |
| .                   |      .         |   .                   |     .        |
| .                   |      .         |   .                   |     .        |
| 0x00000001403ff     |      NO        |   .                   |     RW-      |

Any violation of the security permissions ends up in process termination. You cannot write into text frames or execute data elements. It might actually seem crazy to execute data elements or write into text frames, but there lies the fun part! A lot of future chapters are about the same and you will see how important these permissions are. I have observed a strict rule in computer systems. Do/Give whatever is just needed, when needed. Anything more is unnecessary and can end up in screwing/helping attackers screw the system.
</p>

## 3.3 Shared Libraries and Position Independent Code { #SLandPIC3.3 }

<p>
The processes we considered so far had 2 segments: text and data. These segments are private to a process. No other process should be able to access my text and data. In the first chapter, we discussed about code which can be compiled into a **shared library** / **shared object** which can be shared among various processes as the name suggests. Every shared object also has text and data - code to run and variables, data structures. With whatever we know, how can we implement this sharing mechanism?

In the [first chapter](#Chapter1), to understand shared objects, we used the following.
</p>

```c
-------------------------------------------------------------------------------
chapter3$ cat lib.c
#include <stdio.h>

void lib_func()
{
	printf("Inside lib_func()\n");
}

chapter3$ cat lib.h
#ifndef _LIB_H
#define _LIB_H 1

void lib_func();

#endif	/* _LIB_H */
-------------------------------------------------------------------------------
```

<p>
*lib.c* is the simple shared library. It has one function ```lib_func()``` which has a simple print statement which when executed will prove that the shared library's code is executed.

Generate the shared library.
</p>

```
-------------------------------------------------------------------------------
chapter3$ gcc-4.8 lib.c -c -fPIC
chapter3$ gcc-4.8 lib.o -o libdummy.so --shared

-------------------------------------------------------------------------------
```

<p>
This generates a shared library with name *libdummy.so*.

Now, how do we actually share the shared object among various processes? Let us start by making the shared object part of one, single process. The executable just has the libraries' names it depends on. The shared libraries are separate files. When the executable is run, this is how its virtual address space looks like. The following is just an example.
</p>

```
-------------------------------------------------------------------------------
|-------------------| <-- 0xffffffffffffffff // Highest virtual address
|                   | 
|                   |
|                   |
|                   |
|                   | 
|                   |
|      unused       |
|                   |
|                   | 
|                   |
|                   |  
|                   |
|-------------------| <-- program's data ends
|       data        |
|-------------------| <-- 0x0000000000600000 // program's data starts
|       unused      |
|-------------------| <-- program's text ends
|       text        |
|-------------------| <-- 0x0000000000400000 // program's text starts
|       unused      |
|-------------------| <-- 0x0000000000000000 // Lowest virtual address
-------------------------------------------------------------------------------
```

<p>
Idea of a library is that it has functions, data structures which can be used by the program(and ideally shared among multiple programs). For the program to be able to access something, it must be part of the process's virtual address space. For the program to access the library, it should become part of the process's virtual address space. Basically, we should **map** the shared library's text and data onto the process's virtual address space. It should look something like this.
</p>

```
-------------------------------------------------------------------------------
|-------------------| <-- 0xffffffffffffffff // Highest virtual address
|                   | 
|                   |
|       unused      |
|                   |
|                   | 
|-------------------|
|   dummy's data    |
|-------------------| <-- 0x0000123456900000 // libdummy.so's data starts
|   dummy's text    | 
|-------------------| <-- 0x0000123456789012 // libdummy.so's text starts 
|                   |  
|       unused      |
|-------------------| <-- program's data ends
|       data        |
|-------------------| <-- 0x0000000000600000 // program's data starts
|       unused      |
|-------------------| <-- program's text ends
|       text        |
|-------------------| <-- 0x0000000000400000 // program's text starts
|       unused      |
|-------------------| <-- 0x0000000000000000 // Lowest virtual address
-------------------------------------------------------------------------------
```

<p>
The library has just one function ```lib_func()```. Assume it is present at 0x0000123456789012, the beginning of the library's text. The program now can access that function through the virtual address 0x0000123456789012. This way, the program has access to the library. How will the mapping table look like now? It will have the library's entries too. Now, the library is part of this process. The library's text and data's virtual pages should be accounted for.

Now, one process has access to the library. There comes another process which uses the same library. Now, the library(or atleast some part of it) is already in memory and on demand, the necessary parts of the library can be brought back to memory. With that, we should obviously not load another copy for the sake of the new process. The new process needs to get access to the already present copy of the library. How can that be done? First of all, look at what parts of the library can be shared among multiple processes and what parts need to be private to a particular process. The library has 2 parts - text and data. We know that the text is readable and executable, it is **never** changed because it is not writable. But data is not like that. Data is writable. This means if a variable in the data segment is written into by a process, it should not be changed and must not have access to other processes. It is safe to conclude that all the non-writable parts can be shared among the processes and each process should have its own copy of the writable part of the library. Consider the Standard C library - **libc**. Almost all the processes use it. Its text is shared among all the processes but not the data segment.

That is the idea, but how do we implement it? Take some time and come up with an idea. Lets talk about just the sharable part of the library. Suppose it is already in the memory system(meaning parts of it are in main memory and other parts in secondary storage) and it is mapped to one process's virtual address space. When another process comes in which requires the same library, the same physical frames can be mapped to the second process's virtual space too. Basically, there is one copy of text present in physical memory but it is mapped onto two process's virtual address space. Regarding data segment, new memory needs to be allocated specifically for the second process and that needs to be kept private to the second process - the first process should not be able to access it. With this arrangement, how will the mapping tables of the two processes look like? There won't be much difference. The second process should also have new entries for the virtual pages of library's text and data. Again, note that only certain parts of a shared library can be shared.

We just specified that the text of the library needs to be mapped onto all the necessary processes' virtual address spaces and each process gets a private data segment. We didn't talk about **where** exactly in the virtual address space will the libraries be mapped? Should the library be mapped at the same virtual address ranges of all the processes? or how do you think it should be done? Is it practical to map the library to the same virtual address ranges of all the processes? It is not. We need a flexible mechanism. There are 2 sides to this problem. One is the capability of the operating system to map the library onto any free(available) virtual address range of the process- the operating system should be able to map to any free(available) virtual address range of the process. How do we solve this problem? All we need to do is to make sure that the virtual addresses should not overlap with the program's virtual addresses. We can choose a really large virtual address as a **shared libraries' base virtual address** and start mapping each library from there. Suppose there are 2 libraries **libdummy1.so** and **libdummy2.so**. Let us map the first one now.
</p>


```
-------------------------------------------------------------------------------
|-------------------| <-- 0xffffffffffffffff // Highest virtual address
|                   | 
|                   |
|       unused      | 
|-------------------|<------- Shared Libraries' base address
|   dummy1's data   |
|-------------------| <-- 0x0000123456900000 // libdummy1.so's data starts
|   dummy1's text   | 
|-------------------| <-- 0x0000123456789012 // libdummy1.so's text starts 
|                   |
|                   | 
|                   | 
|                   | 
|                   | 
|                   |  
|       unused      |
|-------------------| <-- program's data ends
|       data        |
|-------------------| <-- 0x0000000000600000 // program's data starts
|       unused      |
|-------------------| <-- program's text ends
|       text        |
```

<p>
Where can we place *libdummy2.so*? Its pretty straight-forward. Instead of placing at some random place, it can be placed right after dummy1's text ends. The operating system can maintain the **next free virtual address**. With just that information, the problem is solved.  dummy2's data and text can be placed at that next free virtual address. Once placed, that address will be updated. This way, the operating system has a simple mechanism to place the shared libraries.

Now, the other side of the problem: Any library can be placed at any random virtual address. This means, the code should work irrespective of where it is placed in the virtual address space. Do you think some special kind of code needs to be generated or the type of code generated for executables is enough for libraries too? Let us dig a little deeper. So far, we have generated only executables which are placed at known addresses and the linker generated the code keeping these addresses in mind. Consider the following program.
</p>

```c
-------------------------------------------------------------------------prog.c
chapter3$ cat prog.c
#include <stdio.h>

int
main()
{
        char *str = "Hello World!";
        printf("main: %s\n", str);

        return 0;
}
-------------------------------------------------------------------------------
```

<p>
We saw that the linker had alloted virtual addresses to 
Let us look at the **LOAD**able segments of this program. Let us see at what virtual addresses these loadable segments need to be mapped.
</p>

```
-------------------------------------------------------------------------------
chapter3$ readelf --segments prog

Elf file type is EXEC (Executable file)
Entry point 0x400400
There are 9 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  .
  .
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x0000000000000708 0x0000000000000708  R E    0x200000
  LOAD           0x0000000000000e08 0x0000000000600e08 0x0000000000600e08
                 0x0000000000000228 0x0000000000000230  RW     0x200000
  .
  .
-------------------------------------------------------------------------------
```

<p>
The first load segment needs to be loaded at the virtual address 0x400000. You can observe that this segment has a lot of sections in it - .text and .rodata(read-only data) sections are two of them.

Let us see in the virtual address range the linker has alloted to the above program's .text and .rodata. *readelf* tool can be used here.
</p>

```
-------------------------------------------------------------------------------
chapter3$ readelf -S prog
.
.
[13] .text             PROGBITS         0000000000400400  00000400
       00000000000001a2  0000000000000000  AX       0     0     16
.
.
[15] .rodata           PROGBITS         00000000004005b0  000005b0
       000000000000001b  0000000000000000   A       0     0     4
-------------------------------------------------------------------------------
```

<p>
Note that there are a lot of other sections too, but these two are enough to understand the scenario. Each and every detail above is interesting. The .text section is at an offset of 0x400 bytes from the segment-base(whose address is 0x400000). The .rodata section is at an offset of 0x5b0 bytes from the segment-base.

**.text** needs to be mapped at the virtual address 0x0000000000400400 and it is 0x1a2 bytes in size. The virtual address range of **.text** would be 0x400400 - 0x4005a2 if it is mapped at 0x400400. Similarly, **.rodata** needs to be mapped at the virtual address 0x00000000004005b0 and it takes 0x1b bytes of memory and its virtual address range is 0x4005b0 - 0x4005cb. Let us take a quick look at ```main()```'s disassembly.
</p>

```asm
-------------------------------------------------------------------------------
chapter3$ objdump -Mintel -d prog
.
.
00000000004004fd <main>:
  4004fd:       55                      push   rbp
  4004fe:       48 89 e5                mov    rbp,rsp
  400501:       48 83 ec 10             sub    rsp,0x10
  400505:       48 c7 45 f8 b4 05 40    mov    QWORD PTR [rbp-0x8],0x4005b4
  40050c:       00 
  40050d:       48 8b 45 f8             mov    rax,QWORD PTR [rbp-0x8]
  400511:       48 89 c6                mov    rsi,rax
  400514:       bf c1 05 40 00          mov    edi,0x4005c1
  400519:       b8 00 00 00 00          mov    eax,0x0
  40051e:       e8 cd fe ff ff          call   4003f0 <printf@plt>
  400523:       b8 00 00 00 00          mov    eax,0x0
  400528:       c9                      leave  
  400529:       c3                      ret    
  40052a:       66 0f 1f 44 00 00       nop    WORD PTR [rax+rax*1+0x0]
.
.
-------------------------------------------------------------------------------
```

<p>
and the read-only data section.
</p>

```
-------------------------------------------------------------------------------
4005b0:  01 00 02 00 48 65 6C 6C 6F 20 57 6F 72 6C 64 21 
4005c0:  00 6D 61 69 6E 3A 20 25 73 0A 00 00
-------------------------------------------------------------------------------
```

<p>
We know that the program will definitely run properly if these sections are mapped to the addresses the linker has requested to. Will the program work if they are mapped at addresses other than 0x400400 and 0x4005b0 respectively? Say I map the segment containing .text and .rodata at 0x800000. This means, .text will be present at the virtual address 0x800400 and .rodata at 0x8005b0. Will the program still run properly?

Read through ```main()```'s disassembly again. Look at the 4th instruction.
</p>

```
-------------------------------------------------------------------------------
400505:       48 c7 45 f8 b4 05 40 00   mov    QWORD PTR [rbp-0x8],0x4005b4
-------------------------------------------------------------------------------
```

<p>
It is clearly visible that 0x4005b4 is the address given to the string **Hello World!** given by the linker. Take a look at the instruction's bytes. The last 4 bytes(**b4-05-40-00**) are the little-endian representation of **0x004005b4**. That this virtual address is hardcoded in that instruction.

Now the two sections are mapped at 0x800400 and 0x8005b0 respectively. This means that the **Hello World!** string is actually present at the virtual address 0x8005b4 but the above instruction still uses the linker-given virtual address 0x4005b4. There is probably nothing mapped at 0x4005b4 and dereferencing it will crash the program.

This means mapping that segment at the new virtual address 0x800000 was a bad idea. The program won't run properly.

Now consider our shared library *libdummy.so*. We already discussed that a shared library should work irrespective of where it is mapped in the virtual address space. It has a function ```lib_func``` which prints a string literal. When the library is linked, the linker allots an virtual address to that string literal. ```lib_func``` would also contain an assembly instruction similar to the one above. That instruction generated also uses string literal's virtual address. This means that the string literal's virtual address is hardcoded into that instruction. With an instruction containing a hardcoded virtual address, will the shared library work irrespective of where it is mapped?

We did the same experiment with *prog*. In *prog*'s ```main()```, a virtual address was hardcoded into an instruction and we mapped it at some other address. We proved that the program would crash. The exact same thing would happen with the library too.

Instructions with hardcoded addresses won't allow shared libraries to function properly if it is mapped elsewhere. But we saw that a shared library needs to work irrespective of where it is mapped. How can this problem be solved? How do we not hardcode addresses in instructions but make it work?

Let us consider *prog* and come up with a solution. First of all, lets list everything we know about *prog*.

1. The first loadable segment is mapped at 0x400000.
2. .text section is present at an offset of 0x400 bytes => It will be mapped at a virtual address 0x400400.
3. .rodata section is present at an offset of 0x5b0 bytes => It will be mapped at a virtual address 0x4005b0.
4. If the loadable segment is mapped at 0x100000, then .text will be present at 0x100400 and .rodata at 0x1005b0.
5. Observe the relative addresses of bytes in .text and .rodata with respect to the segment base. They never change. The string **Hello World!** in *prog* is always at an offset of 0x5b4 bytes from the segment base. The instruction we considered is always at an offset of 0x505 bytes from segment base. Whichever virtual address you map the segment, the relative positions of bytes with respect to the segment base always remains the same.

With the above details, can you think of a solution?

Consider the string literal and the instruction.
</p>

```asm
-------------------------------------------------------------------------------
String literal:
4005b4:  48 65 6C 6C 6F 20 57 6F 72 6C 64 21    ; Hello World!

Instruction:
400505:       48 c7 45 f8 b4 05 40 00   mov    QWORD PTR [rbp-0x8],0x4005b4
-------------------------------------------------------------------------------
```

<p>
It is currently like this. The address 0x4005b4 is used to refer to the string literal. But we need a mechanism to refer to that string without hardcoding it in the instruction.

The string literal is always at an offset of 0x5b4 bytes from the segment base. The instruction is always at an offset of 0x505 bytes. This means, the gap between the string literal and the instruction is always (0x5b4 - 0x505) = 0xaf bytes. At runtime, let the segment be loaded at the virtual address V_base. The instruction will be present at V_i = (V_base + 0x505). The string literal will be present at V_s = (V_i + 0xaf). The instruction now can easily refer the string just by adding an offset of 0xaf to its own virtual address. We need something like this.
</p>

```asm
-------------------------------------------------------------------------------
String literal:
V_base + 0x5b4:  48 65 6C 6C 6F 20 57 6F 72 6C 64 21    ; Hello World!

Instruction:
V_base + 0x505:         mov    QWORD PTR [rbp-0x8], my_own_virtual_address + 0xaf
-------------------------------------------------------------------------------
```

<p>
Our problem has been transfered from hardcoding the string literal's address to hardcoding instruction's address. At runtime, how can the instruction know its address? When this instruction is getting executed, what does the Instruction Pointer(rip) contain? It contains the next instruction's address. Assuming the above instruction's encoding is 8 bytes, address of the next instruction = (V_base + 0x505 + 8). This means, when this instruction is getting executed, rip = (V_i + 8) => V_i = (rip - 8). There you go! We now have the instruction's virtual address. Now, the instruction looks like this.
</p>

```asm
-------------------------------------------------------------------------------
String literal:
V_base + 0x5b4:  48 65 6C 6C 6F 20 57 6F 72 6C 64 21    ; Hello World!

Instruction:
V_base + 0x505:         mov    QWORD PTR [rbp-0x8], (rip - 8) + 0xaf
-------------------------------------------------------------------------------
```

<p>
The instruction is now totally independent of V_base. Now, the segment can be placed at any position in the virtual address space, this code will work irrespective of that. Such code is called **Position Independent Code(PIC)**. You place it at 0x400000, it will work. You place it at 0x800000 it will work.

When the compiler generates code for a shared library, it must position independent code like above. Code which relies on runtime information rather than link time. This way, a shared library will work irrespective of its position in the virtual address space.

We simply constructed some hypothetical instruction there which will make that instruction position independent. But it will work only if the processor ISA supports such an instruction and yes it does. The x64 ISA supports this type which allows you to refer something relative to the Instruction Pointer(rip). This is called RIP-relative addressing mode. You are calculating the address of something using **rip**.

Now, let us look at *libdummy.so*'s code. It should contain code like above. Generate its disassembly and inspect it.
</p>

```
-------------------------------------------------------------------------------
chapter3$ objdump -Mintel -D libdummy.so > dummy.obj
.
.
0000000000000665 <lib_func>:
 665:   55                      push   rbp
 666:   48 89 e5                mov    rbp,rsp
 669:   48 8d 3d 11 00 00 00    lea    rdi,[rip+0x11]        # 681 <_fini+0x9>
 670:   e8 eb fe ff ff          call   560 <puts@plt>   
 675:   5d                      pop    rbp
 676:   c3                      ret    
.
.
0000000000000681 <.rodata>:
 681:   49 6e 73 69 64 65 20 6c 69 62 5f 66 75 6e 63 28 29 00
-------------------------------------------------------------------------------
```

<p>
Look at the third instruction: ```lea    rdi,[rip+0x11]```. That 0x11 is the gap between the fourth instruction and the string literal. The instruction is essentially ```lea rdi, [rip - 7 + 0x18]``` where (rip-7) is the third instruction's address and 0x18 is the gap between the third instruction and the string literal.

I want you to read through the disassembly thoroughly. You won't find a single reference to a hardcoded address in it. Everything will be relative to something else which is decided at runtime.
</p>

## 3.4 Position Independent Executable (PIE) { #pie3.4 }

<p>
We saw that the linker had alloted virtual addresses to *prog*'s segments. The first loadable segment need to be mapped at virtual address 0x400000 and the second at 0x601000. It contained instructions which had virtual addresses hardcoded in it. I want you to go through *prog*'s disassembly thoroughly. The two loadable segments **must** be loaded at 0x400000 and 0x601000 for the program to work properly.

If you think about it, even executables can be made position independent. The compiler should generate position independent code. Executables which are position independent are called Position Independent Executables(PIE). But what is a the need to make it position independent? There are a couple of advantages. It aids in implementing a highly effective security technique called ASLR which we will discuss in one of the future chapters.

This is the reason why **gcc-4.8** was used so far. It generates position dependent executables. Explaining PIC and PIE right in the beginning would be hard and would not have made much sense, so I used an older version of the gcc compiler. Modern versions(6, 7, 8) all generate position independent executables by default. Let us check it out.
<p>

```
-------------------------------------------------------------------------------
chapter3$ gcc --version
gcc (Ubuntu 7.5.0-3ubuntu1~18.04) 7.5.0
Copyright (C) 2017 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
-------------------------------------------------------------------------------
```

<p>
The default compiler version in my system is 7.5.0. Let me compile *prog.c* using this.
</p>

```
-------------------------------------------------------------------------------
chapter3$ gcc prog.c -o prog.7.5.0
chapter3$ ./prog.7.5.0
main: Hello World!
-------------------------------------------------------------------------------
```

<p>
It will run fine. Let us use the **file** command and check *prog* and *prog.7.5.0*.
</p>

```
-------------------------------------------------------------------------------
chapter3$ file prog
prog: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/l, for GNU/Linux 3.2.0, BuildID[sha1]=c0fe22fccfe7b1378021a26fe4bbdcc823f6bcd6, not stripped
chapter3$ file prog.7.5.0
prog.7.5.0: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/l, for GNU/Linux 3.2.0, BuildID[sha1]=43c4cdd09bcd5510bcde2a311ea9eaef50ced92d, not stripped
-------------------------------------------------------------------------------
```

</p>

**file** recognizes *prog.7.5.0* as a shared object because it is position independent. But we know that it is an executable. Then what exactly is the difference between a position independent executable and a shared object? One **defining** difference is that an executable has an **entry point**. If you try running *libdummy.so*, it will segfault because it does not have an entry point. We have seen that both executables and shared libraries can be shared objects. When we refer to something as shared object, it simply means it has position independent code. If it has a well-defined entry point, then it is an executable and if it doesn't, it is a shared library.

Let us try something interesting. Position Independent Executables(PIEs) have all the properties of a shared library. So can it be used as a shared library? Let us check it out.

Let us write a normal C program with a function in it like the following. I will be using my default compiler from now on.
</p>

```c
--------------------------------------------------------------------libdummy2.c
chapter3$ cat libdummy2.c 
#include <stdio.h>

void
func()
{
        printf("Inside func()\n");
}

int
main()
{
        printf("Inside main!\n");
        func();
}
chapter3$ gcc libdummy2.c -o libdummy2.so
-------------------------------------------------------------------------------
```

<p>
It has a function ```func()``` which is called by ```main()```. Note that *libdummy2.so* is a PIE. My question is can this ```func()``` now be called by some other program if it is linked against *libdummy2.so*? Let us write *prog2.c* which calls ```func()```.
</p>

```
-------------------------------------------------------------------------prog.c
chapter3$ cat prog2.c
#include <stdio.h>

int
main()
{
        func();
        return 0;
}
-------------------------------------------------------------------------------
```

<p>
It is a very simple program. Let us compile it and link it against *libdummy2.so*.
</p>

```
-------------------------------------------------------------------------------
chapter3$ gcc prog2.c -o prog -L. -ldummy2
/tmp/ccjPom8V.o: In function `main':
prog2.c:(.text+0xa): undefined reference to `func'
collect2: error: ld returned 1 exit status
-------------------------------------------------------------------------------
```

<p>
Nope! The linker gave an **undefined reference** error. Even though *libdummy2.so* has the function definition in it. the linker could not find it. This is one more difference between a PIE and a shared library. Although both of them are shared objects, only functions present in a shared library can **actually** be accessed by other programs. This implies that position independent code(PIC) is a necessary condition to become a shared object, it is not a sufficient condition. Then what is the sufficient condition?

*libdummy.so* has ```lib_func``` function and we have used it in an external program *prog*. *libdummy2.so* has ```func``` and we couldn't use it. Function definitions are present in both binaries but we could link *prog* against *libdummy.so* but the linker gave an undefined reference error when we tried to link *prog2* against *libdummy2.so*. This means, there should be something in *libdummy.so* which informs the linker that it has ```lib_func``` and other programs can use it. In [first chapter's Init](#Init1.1), we saw how shared libraries are built. Just having function definitions is not sufficient. The linker should know that these function definitions exist in the shared library. A shared library has a list of all the functions exposed to the world whereas a PIE does not have that list.

The Dynamic Symbol Table(**.dynsym** section) present in these objects have the details. Let us checkout *libdummy.so*'s (shared library's) .dynsym section.
</p>

```
-------------------------------------------------------------------------------
chapter3$ readelf --dyn-syms libdummy.so 

Symbol table '.dynsym' contains 13 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterTMCloneTab
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.2.5 (2)
     3: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     4: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _Jv_RegisterClasses
     5: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMCloneTable
     6: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND __cxa_finalize@GLIBC_2.2.5 (2)
     7: 0000000000201028     0 NOTYPE  GLOBAL DEFAULT   23 _edata
     8: 0000000000201030     0 NOTYPE  GLOBAL DEFAULT   24 _end
     9: 0000000000000665    18 FUNC    GLOBAL DEFAULT   12 lib_func
    10: 0000000000201028     0 NOTYPE  GLOBAL DEFAULT   24 __bss_start
    11: 0000000000000530     0 FUNC    GLOBAL DEFAULT    9 _init
    12: 0000000000000678     0 FUNC    GLOBAL DEFAULT   13 _fini
-------------------------------------------------------------------------------
```

<p>
The 10th entry is about ```lib_func```.

Now take a look at *libdummy2.so*'s (PIE's) .dynsym. 
</p>

```
-------------------------------------------------------------------------------
chapter3$ readelf --dyn-syms libdummy2.so

Symbol table '.dynsym' contains 7 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterTMCloneTab
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.2.5 (2)
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@GLIBC_2.2.5 (2)
     4: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     5: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMCloneTable
     6: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND __cxa_finalize@GLIBC_2.2.5 (2)
-------------------------------------------------------------------------------
```

<p>
There you go! This does not have any entry about ```func()```.

Obvious question: Now how do we convert a position independent executable into a shared library? We have a conceptual answer to it. We neeed to expose the functions present in the PIE. How can that be done? Entries about these functions can be added to the **.dynsym** of the PIE. It can be done using the **-rdynamic** option. Take a look.
</p>

```
-------------------------------------------------------------------------------
chapter3$ gcc libdummy2.c -o libdummy2.so -rdynamic
-------------------------------------------------------------------------------
```

<p>
Now let us inspect the new *libdummy2.so*'s .dynsym section.
</p>

```
-------------------------------------------------------------------------------
chapter3$ readelf --dyn-syms libdummy2.so

Symbol table '.dynsym' contains 20 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterTMCloneTab
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.2.5 (2)
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@GLIBC_2.2.5 (2)
     4: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     5: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMCloneTable
     6: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND __cxa_finalize@GLIBC_2.2.5 (2)
     7: 0000000000201010     0 NOTYPE  GLOBAL DEFAULT   23 _edata
     8: 0000000000201000     0 NOTYPE  GLOBAL DEFAULT   23 __data_start
     9: 0000000000201018     0 NOTYPE  GLOBAL DEFAULT   24 _end
    10: 0000000000201000     0 NOTYPE  WEAK   DEFAULT   23 data_start
    11: 00000000000008e0     4 OBJECT  GLOBAL DEFAULT   16 _IO_stdin_used
    12: 0000000000000860   101 FUNC    GLOBAL DEFAULT   14 __libc_csu_init
    13: 0000000000000720    43 FUNC    GLOBAL DEFAULT   14 _start
    14: 0000000000201010     0 NOTYPE  GLOBAL DEFAULT   24 __bss_start
    15: 000000000000083d    33 FUNC    GLOBAL DEFAULT   14 main
    16: 00000000000006d8     0 FUNC    GLOBAL DEFAULT   11 _init
    17: 00000000000008d0     2 FUNC    GLOBAL DEFAULT   14 __libc_csu_fini
    18: 00000000000008d4     0 FUNC    GLOBAL DEFAULT   15 _fini
    19: 000000000000082a    19 FUNC    GLOBAL DEFAULT   14 func
-------------------------------------------------------------------------------
```

<p>
Great! Now the ```func()``` present in *libdummy2* is exposed to the real world for use. Let us try linking *prog2.c* against *libdummy2.so*.
</p>

```
-------------------------------------------------------------------------------
chapter3$ gcc prog2.c -o prog2 -L. -ldummy2
chapter3$ ./prog2
Inside func()
-------------------------------------------------------------------------------
```

<p>
There you go! *libdummy2.so* is now both an executable and a shared library. A PIE is also a shared library.

Try running standard shared libraries like libc.so, ld.so etc., You will find that a couple of them are PIEs which are also shared libraries.

From now onwards, I will be using the default compiler on my system (gcc 7.5.0) now that we know what PIC is.

If a binary is a shared object, it means that it is made up of Position Independent Code(PIC). A binary made up of PIC does not imply that it is a shared library or a Position Independent Executable(PIE). Difference between any executable and a shared library is that the shared library has PIC and its symbols are exposed to the real world whereas the executable may or may not have PIC and its symbols are not exposed by default. Most importantly, the executable has an entry-point. A PIE can be converted into a shared library by exposing its symbols. To avoid confusion, let us stick to 4 terms. PIE means it is just an executable. Shared Library means it is a shared library and not PIE. An Executable Shared Library means it is a PIE and a Shared Library. Shared object is any binary which contains Position Independent Code - it can be PIE or Shared Library.
</p>

## 3.5 Memory Layout of a process { #Section3.5 }

<p>
In all the previous sections, we focused on specific parts of the entire virtual address space. In [Init](#Init3.1), we always considered that a program is made up of only 2 parts - text and data. In [Section 3.3](#SLandPIC3.3), we focused completely on shared libraries and in [Section 3.4](#pie3.4), we came back to executable's sections and saw how it can be made position independent. At this point, we still don't have an idea of how the entire virtual address space of a process looks like, what all does it contain. We know that it contains the program's text, data and shared libraries' text and data. In this section, we will be exploring all the elements present in a process's virtual address space.

What all should be present in a process's virtual address space for it to run properly?

Every program has 3 compulsory parts - text, data and read-only data. It can be dependent on multiple shared libraries where each of those shared library has text, data and read-only. Along with that, each of them has its own metadata. Generally programs would want to allocate memory at runtime. Some part of address space needs to be given to that. We saw in previous chapters that a program needs a stack to run properly. These are the basic elements present in the address space. Let us now consider a program and inspect its elements. 
</p>

```c
------------------------------------------------------------------------prog3.c
chapter3$ cat prog3.c 
#include <stdio.h>

int main()
{
        printf("Hello!\n");
        while(1);
}
chapter3$ gcc prog3.c -o prog3
chapter3$ ./prog3
Hello!

^C
chapter3$
-------------------------------------------------------------------------------
```

<p>
Its a hello program ending which enters an infinite loop after printing the string. The **/proc** directory has details of every process. Let us make use of the information present in that directory. This is what the **/proc** directory looks like.
</p>

```
-------------------------------------------------------------------------------
/proc$ ls
1      1300  142   1495  170    19721  2359   2442  2535  2711   33    404   6135  7126  988          fs           mounts         uptime
10     1307  1422  1496  1771   1985   2380   2446  2540  277    34    405   6153  7165  992          i8k          mtrr           version
1066   1308  1425  15    1774   2      2387   245   2543  278    347   406   6188  7832  997          interrupts   net            version_signature
11     1314  1427  1504  1783   20577  23915  2454  2548  28     35    407   6195  830   999          iomem        pagetypeinfo   vmallocinfo
11413  1333  143   1505  1794   2063   2392   246   2549  28004  352   41    6202  833   acpi         ioports      partitions     vmstat
1190   1357  1436  1506  18     21     23978  247   2552  28879  3538  411   6247  834   asound       irq          pressure       zoneinfo
12     136   144   1507  1808   21159  24     2472  2553  28906  36    42    6248  9     buddyinfo    kallsyms     sched_debug
1200   1365  1451  151   1809   21163  240    2481  2561  29     361   4255  6251  952   bus          kcore        schedstat
1207   137   1452  1515  1871   22     24038  2486  2563  2902   368   43    6253  954   cgroups      keys         scsi
1209   1370  1463  1516  189    2207   2404   2490  2611  2905   374   432   6354  955   cmdline      key-users    self
1212   1372  1471  1522  1916   2208   241    2496  2619  2926   38    438   6367  957   consoles     kmsg         slabinfo
1225   1376  1472  1533  1948   2221   2417   2500  2630  29833  384   450   6740  961   cpuinfo      kpagecgroup  softirqs
1247   1377  1475  154   1955   2229   2419   2509  2642  3      39    473   6756  966   crypto       kpagecount   stat
1263   138   1479  155   19703  2231   242    2510  2653  30     396   5024  6757  973   devices      kpageflags   swaps
1268   1380  1483  156   19704  2238   2423   2511  2656  3078   398   5029  6758  976   diskstats    loadavg      sys
1275   139   1486  1584  19716  2241   2425   2515  2678  3120   4     5042  6773  977   dma          locks        sysrq-trigger
1283   14    1487  159   19717  23     243    2518  2685  31449  40    5123  6788  978   driver       mdstat       sysvipc
1295   140   1488  16    19718  2349   2430   2522  27    31542  400   5318  6818  979   execdomains  meminfo      thread-self
1297   141   149   1623  19719  2352   2437   2530  2702  31757  401   539   6827  984   fb           misc         timer_list
13     1419  1492  17    19720  2357   244    2534  2709  328    402   5685  7120  985   filesystems  modules      tty
-------------------------------------------------------------------------------
```

<p>
Each of those numbers are ProcessIDs. Apart from that, the **/proc** directory also has some interesting details about the operating system. Let us run *prog3* and get into its directory. Note that a directory for a process will be present in **/proc** till that process is running. Once it is terminated, the directory goes away. The infinite loop makes sure the process doesn't terminate. Lets go!
</p>

```
-------------------------------------------------------------------------------
chapter3$ ./prog3
Hello!

-------------------------------------------------------------------------------
```

<p>
Leave it alone, open up a new terminal. Lets get its PID and go to its directory.
</p>

```
-------------------------------------------------------------------------------
chapter3$ ps -e | grep prog3
 7881 pts/4    00:00:44 prog3
/proc/7630$ ls
arch_status      environ    mountinfo      personality   statm
attr             exe        mounts         projid_map    status
autogroup        fd         mountstats     root          syscall
auxv             fdinfo     net            sched         task
cgroup           gid_map    ns             schedstat     timers
clear_refs       io         numa_maps      sessionid     timerslack_ns
cmdline          limits     oom_adj        setgroups     uid_map
comm             loginuid   oom_score      smaps         wchan
coredump_filter  map_files  oom_score_adj  smaps_rollup
cpuset           maps       pagemap        stack
cwd              mem        patch_state    stat
-------------------------------------------------------------------------------
```

<p>
A process has a lot of metadata. We are currently interested in how its virtual address space looks like, what all is present in it. The **maps** file has the necessary details.
</p>

```
-------------------------------------------------------------------------------
56460cb7e000-56460cb7f000 r-xp 00000000 08:05 10775454 XXXXX/chapter3/prog3
56460cd7e000-56460cd7f000 r--p 00000000 08:05 10775454 XXXXX/chapter3/prog3
56460cd7f000-56460cd80000 rw-p 00001000 08:05 10775454 XXXXX/chapter3/prog3
56460ea06000-56460ea27000 rw-p 00000000 00:00 0        [heap]
7f6ed70a6000-7f6ed728d000 r-xp 00000000 08:05 13112024 /lib/x86_64-linux-gnu/libc-2.27.so
7f6ed728d000-7f6ed748d000 ---p 001e7000 08:05 13112024 /lib/x86_64-linux-gnu/libc-2.27.so
7f6ed748d000-7f6ed7491000 r--p 001e7000 08:05 13112024 /lib/x86_64-linux-gnu/libc-2.27.so
7f6ed7491000-7f6ed7493000 rw-p 001eb000 08:05 13112024 /lib/x86_64-linux-gnu/libc-2.27.so
7f6ed7493000-7f6ed7497000 rw-p 00000000 00:00 0
7f6ed7497000-7f6ed74be000 r-xp 00000000 08:05 13111996 /lib/x86_64-linux-gnu/ld-2.27.so
7f6ed768f000-7f6ed7691000 rw-p 00000000 00:00 0
7f6ed76be000-7f6ed76bf000 r--p 00027000 08:05 13111996 /lib/x86_64-linux-gnu/ld-2.27.so
7f6ed76bf000-7f6ed76c0000 rw-p 00028000 08:05 13111996 /lib/x86_64-linux-gnu/ld-2.27.so
7f6ed76c0000-7f6ed76c1000 rw-p 00000000 00:00 0
7fffc5f19000-7fffc5f3b000 rw-p 00000000 00:00 0        [stack]
7fffc5f74000-7fffc5f77000 r--p 00000000 00:00 0        [vvar]
7fffc5f77000-7fffc5f78000 r-xp 00000000 00:00 0        [vdso]
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0 [vsyscall]
-------------------------------------------------------------------------------
```

<p>
Let us talk about the first 3 entries. The first virtual address range is readable-executable. You can guess that it contains the text. It also contains other metadata useful for execution. Next is a read-only address range. It contains all read-only elements like string literals. Third is a read-write address range. It has the data and some metadata useful for execution. To understand what this metadata is, we will have to explore the ELF in detail. Look at the 5th column. It is the **inode-number** of **prog3**. That can be verified.
</p>

```
-------------------------------------------------------------------------------
chapter3$ ls -i prog3
10775454 prog3
-------------------------------------------------------------------------------
```

<p>
The **inode-number** is a unique number given to a file, that is how the filesystem internally refers to that file.

The 4th column mentions where that file is present. Every hard-drive partition is identified by a pair of numbers - **Major number : Minor number**. The **/dev** directory has all the hardware and software devices.
</p>

```
-------------------------------------------------------------------------------
/dev$ ls -l sda*
brw-rw---- 1 root disk 8, 0 May  5 22:57 sda
brw-rw---- 1 root disk 8, 1 May  5 22:57 sda1
brw-rw---- 1 root disk 8, 2 May  5 22:57 sda2
brw-rw---- 1 root disk 8, 3 May  5 22:57 sda3
brw-rw---- 1 root disk 8, 4 May  5 22:57 sda4
brw-rw---- 1 root disk 8, 5 May  5 22:57 sda5
-------------------------------------------------------------------------------
```

<p>
Harddrives(specifically SCSI devices) are identified by the major number **8**. [This](https://www.kernel.org/doc/Documentation/admin-guide/devices.txt) page documents all the major numbers. Each of the partitions are identified by the minor numbers.

The last column tells us what that address range is.

The 4th entry is something called **heap**. Any dynamically allocated memory(using malloc, calloc, realloc) comes from this address range. This heap has nothing to do with the heap data structure(binary heap, binomial heap, fibonacci heap etc.,). This heap is a heap of memory just like a heap/pile of clothes.

The first 4 ranges are close to each other. There is a huge gap between heap and the next entry. The next 4 entries belong to the C standard library(libc). There are 4 address ranges belonging to libc. This is where the shared libraries are mapped. Because this is a simple program, it depends only on the C library. 

There are a couple of address ranges which are readable-writable but don't have any description. Let us discuss about these later.

Then comes the dynamic linker **ld**. Then comes the runtime **stack**. The rest of the ranges are **vvar**, **vdso** and **vsyscall**. Each of these need some explanation which we will discuss later.

We have seen the memory layout of a process. Let us generalize it. It looks like this.
</p>

```
-----------------------------------------------------Memory Layout of a Process
      0xffffffffffffffff------->|                           |
                                |                           |
                                |          Unused           |
                                |                           |
                                |                           |
                                |---------------------------|
                                |                           |
                                |          Stack            |
                                |                           |
                                |     (Grows downwards)     |
                                |---------------------------|
                                |            ||             |
                                |            \/             |
                                |                           |
                                |          Unused           |
                                |                           |
  Shared Libraries' base------->|---------------------------|
                                |                           |
                                |      Shared Libraries     |
                                |                           |
                                |---------------------------|
                                |                           |
                                |                           |
                                |          Unused           |
                                |                           |
                                |     (Grows upwards)       |
                                |            /\             |
                                |            ||             |
                                |---------------------------|
                                |           heap            |
                                |---------------------------|
                                |           data            |
                                |---------------------------|
                                |         read-only         |
                                |---------------------------|
                                |           text            |
                                |---------------------------|
                                |                           |
                                |         Unused            |
                                |                           |
 0x00000000000000000000 ------->|                           |
-------------------------------------------------------------------------------
```

<p>
I want you to go through the **/proc/PID/maps** file again and cross-check the above diagram.

You take any process, its memory layout will be similar to this.
</p>

## 3.6 The mmap() system call

In the previous section, we saw that various files(or its parts) being part of the virtual address space - it could be the executable itself(text, data and read-only) or it could be segments of other shared objects like the dynamic linker and C library. How exactly do these become part of the virtual address space? Does being part of the virtual address space mean memory is allocated? What exactly does mapping mean? Does it mean making something part of the physical memory or virtual address space? Let us explore these questions in this section.

Consider a file of size 10,000 bytes and you want to do some processing on it. Where do you start? There are various ways to do it. You can use the file-related functions and load the part of the file to be processed into a buffer and then process it. Another way is to include the file into the process's virtual address space. Once that is done, you can use virtual addresses to access any part of the file and then process it. Consider the second method. How would you include the file into the process's virtual address space? Can you think of a mechanism to make this work?

Let us start simple. However big the file is, let us first allocate actual physical memory required to load the entire file(here 10000 bytes). To access this memory, we also need to allocate a patch of virtual addresses - an address space of 10000 addresses. Then create mapping table entries to link the physical memory with virtual addresses. From here, it is straight-forward and should work. There is only one downside to this mechanism. We allocate physical memory for the entire file and then proceed. We have already seen cases when we do not have sufficient physical memory and have already solved that problem using **swapping**. So the only change is this. To start with, let us just allocate a patch of virtual addresses. Create entries in the mapping table. As of now, in the mapping table, all these new virtual pages **don't** have corresponding physical frames. When a virtual address is accessed, the corresponding piece of the file can be put into physical memory, the mapping table can be updated and then let the process use it. Let the first virtual address given be 0x12340000. This internally points to the first byte of the file. Say the process tries to access the first 4 bytes at virtual address 0x12340000. These 4 bytes are still not in memory. The first page of the file currently present in secondary storage is loaded into main memory, the mapping table entry is updated. This way, any piece of file is loaded into memory on a **demand-basis** which is a simple way to use memory efficiently. But one question. When the process tries to access 0x12340000, how will we(the operating system) know that the first byte of this file is being accessed? The operating system will have to store the file descriptor along with the virtual address patch - it is basically a mapping between the file and that virtual address patch.

This is exactly what the **mmap()** system call does. Here, I think it would be wrong to tell that mmap maps the file into memory because it is not mapping the entire file into memory. It breaks the file into pages and loads only the pages needed by the process. When it is not needed, the page-frame mechanism throws pieces of file into swap space. What I can tell is that the file is mapped entirely onto the virtual address space. For every byte in the file, there is already a virtual address to access it.

I hope you can appreciate this mechanism. A patch of virtual addresses may be allocated and is linked to a file. But it may not be linked to any physical memory behind. Only when you try accessing it, memory is given to only that part of the file being accessed.

## 3.7 Conclusion

With that, let us conclude this chapter.

We discussed quite a lot in this chapter. Started with simple memory models, broke the memory into pieces, then brought in the concept of virtual memory to make room to run big programs. Then went on to discuss what position independent code is, discussed the differences between shared objects, shared libraries and position independent executables. We then saw how the virtual address space of a generic process looks like and ended the chapter with exploring the very interesting mmap system call.

If it was just about understanding vulnerabilities like buffer overflow or format string, this chapter was not required. We could have done away with this chapter. But the memory subsystem is a very interesting one(the most interesting according to me). A while before writing this chapter, I read the papers on [meltdown and spectre](https://meltdownattack.com/) vulnerabilities. Reading those papers made me understand the intricacies and nuances of the memory subsystem. If you want to understand those vulnerabilities, you will require a solid understanding of the memory subsystem. That is why, this chapter was included.

This chapter contains a lot of theory and some practicals. I urge you to write small programs, dissect them, check if they are PIE or not, look at their **/proc/PID/maps** files and see how the virtual address space looks like, see if you can dig up something interesting.

## 3.8 Further Reading

<p>

1. [Opal: A single address space Operating System](https://homes.cs.washington.edu/~levy/opal/opal.html): Complete design, documentation, papers, reports related to a single address space OS.

2. [Before Memory was Virtual by Peter J. Denning](http://denninginstitute.com/pjd/PUBS/bvm.pdf)

3. [Complete Fair Scheduling(CFS): The linux process scheduling algorithm](https://opensource.com/article/19/2/fair-scheduling-linux)
</p>
