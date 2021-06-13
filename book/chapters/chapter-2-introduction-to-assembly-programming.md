# Chapter 2: Introduction to Assembly Programming { #Chapter2 }

-------------------------------------------------------------------------------
**Summary**
Every architecture offers an Instruction Set which can be used to program it. 
This chapter is about understanding what an Instruction Set is and learning 
the x86/x64 Instruction Set in particular. Finally writing programs for
Intel/AMD processors.
-------------------------------------------------------------------------------

## 2.1 Init { #Init2.1 }

<p>
Let us start with a question. What is a programming language? A programming language is a set of instructions and constructs which can be used to program machines. That machine can be hardware, it can be implemented in software(virtual machines). In this chapter, we will be exploring the type of programming language used to program processors.

In [Chapter1's Init](#Init1.1), we defined our own set of instructions and rules to program an imaginary processor. It had 256 instructions - which can be encoded with 1 byte. We just didn't specify instructions, we also specified a lot of other things. We specified a way to access data - only through addresses. We specified that operands of most instructions(add, sub, mul, copy, jump etc.,) are memory addresses. We defined a datatype of size 2 bytes. We can go ahead and define all 256 instructions, how they are encoded, their operands, datatypes etc., With all this, a programmer can write programs which can be run on that processor. All this is collectively called an **Instruction Set**. You write a program using this well defined instruction set, but then what would you want to do? You obviously want to run it. Just having a well defined instruction set is not enough. There should be a **machine** on which you can run that program. Basically, that machine should understand the instruction set and run the instructions listed in the program. For this to happen, that machine should **implement** this instruction set. What is that machine? Is it a piece of hardware? Is it another software program?

The machine which implements that Instruction Set be seen from 2 sides. One is from programmer's side. All the programmer sees is the Instruction Set - whatever is given to him to write programs. This programmer's perspective of a machine is called **Architecture**. That is why, it is generally refered to as **Instruction Set Architecture(ISA)**. Another view is the actual implementation of the machine - how is each instruction implemented? How are the arithmetic instructions implemented? What algorithms are used? etc., This view is called **Organization**. Take a look at the following diagram.
</p>

```
-------------------------------------------------------------------------------
        ------------------------|   
                                | |---------------|
          Programmers,          | |   Machine     |
          Compilers             | |               |
                                | |               |
                                | |      [2]      |
                [1]             | |               |
                                | |---------------|
        ------------------------|
                               ISA        
-------------------------------------------------------------------------------
```

<p>
Let us take a simple example.

In our instruction set, the ```mul``` instruction multiplies contents at the 2 operands and stores it back into the memory pointed by the first operand. Its syntax is ```mul address1, address2```.

Consider two processor manufacturers(you and me) implement the same instruction set. Consider the ```mul``` instruction's [implementation](https://en.wikipedia.org/wiki/Binary_multiplier). There are various multiplication algorithms which can be implemented at the hardware level. I decide to implement algorithm1, you decide to implement algorithm2. There are 2 perspectives. From programmer's perspective, the internal implementation does not matter. The processor manufacturer gives an instruction to multiply 2 numbers and thats all the programmer is concerned about.

Let us go a bit deeper into what what this machine contains which makes it capable of running programs. All modern machines follow an architecture called as **Von Neumann Architecture**.
</p>

```
-------------------------------------------------------Von Neumann Architecture
                        |-----------------------|
                        ||---------------------||
                        |||-------------------|||
                        |||   Control Unit    |||
                        |||-------------------|||
        |--------|      |||       ALU         |||      |----------|
        | Input  |----->|||-------------------|||----->|  Output  |
        |--------|      ||        CPU          ||      |----------|
                        ||---------------------||
                        |          |            |
                        ||---------------------||
                        ||   Memory unit       ||
                        ||---------------------||
                        |-----------------------|
-------------------------------------------------------------------------------
```

<p>
Let us first understand the design.

1. It has one compute unit - the CPU which is present to execute instructions.
2. It has a memory unit to store instructions, data, results etc.,
3. It has Input and Output - the way users interact with the system.

Based on this, we can design what all instructions need to be included in the Instruction Set.

1. There is an **Arithmetic and Logic Unit(ALU)**. This means the CPU has hardware for Arithmetic and Logical operations like Addition, Multiplication, Division, Bit manipulation etc., The instruction set should contain ALU-type instructions through which programmers can use the ALU.

2. There is a connection between the **Memory Unit** and CPU. This means, there should be
* **Memory Access Instructions**: Which are required to load values from the memory unit and store back value/results back to the memory unit.
* Some memory manipulation instructions.

3. To write programs with conditional statements, loops, functions, interrupts etc., there is a need for **Control Instructions**. These instructions handle the control flow of a program.

How does the CPU(processor) communicate with the memory unit, input-output devices? For any device(memory unit or input-output device) to communicate with the CPU, it should be identified first by the CPU. CPU does that by giving **unique addresses** to the memory unit and input-output devices. The CPU is connected to these devices through a series of wires called **bus**. There are 3 types of buses: **Address bus**, **Data bus** and **Control bus**. Got this beautiful diagram from [here](http://www.iust.ac.ir/files/ee/pages/az/mazidi.pdf).
</p>

```
-----------------------------------------------------------------------------------------
|----------|  Address bus
|          |====================================================================
|          |      |         |          |           |          |          |
|   CPU    |  --------- --------- ----------- ------------ -------- ------------ 
|          |  |  ROM  | |  RAM  | | Printer | | Monitor  | | Disk | | Keyboard |
|          |  --------- --------- ----------- ------------ -------- ------------
|          |    | |       | |        | |        |  |        | |       |  |
|          |====|=========|==========|==========|===========|=========|=========
|          |    |         | Data bus |          |           |         |
|          |    |         |          |          |           |         |
|Read/Write|====================================================================
|----------|Control bus

-----------------------------------------------------------------------------------------
```

<p>
Let us discuss the above diagram in detail.

1. **Read Only Memory(ROM)**: This is a piece of memory present in the Motherboard. As the name suggests, it is read-only. An exception is raised if we try writing into it.

2. **Random Access Memory(RAM)**: This it the main memory present in our computers. Every byte in it has a unique address(aka it is **byte addressable**). Data can be read from and written to it.

3. **Printer** and **Monitor** are output devices. Data can only be written to these devices. The output devices process our input and generates suitable output. For example, we send a PDF file to the printer to print it - this is the input. The printer processes that input and prints it - this is the output.

4. **Disk** is a storage device: Data can be read from and written to it.

5. **Keyboard** is an input device. Data can only be read from it. As we type something, the CPU can take that as data-input.

When a device is connected, the CPU gives a unique set of addresses to that device. These addresses are used to communicate with that device.

Let us take an example. Consider a system where

1. ROM size is 1MiB.
2. RAM size is 4GiB.
3. Assume printer would need 1Ki addresses.
4. Monitor would need 1Ki addresses.
5. Disk would need 1Ki of addresses.
6. Keyboard would need 512 addresses.

The CPU starts alloting the requested number of addresses to these devices. Starting from **0**, Address space of

1. ROM: 0x0 - 0xfffff, size = 0x100000 = 1,048,576
2. RAM: 0x100000 - 0x1000fffff, size = 0x100000000 = 4,294,967,296
3. Printer: 0x100100000 - 0x1001003ff, size = 0x400 = 1,024
4. Monitor: 0x100100400 - 0x1001007ff, size = 0x400 = 1,024
5. Disk: 0x100100800 - 0x100100bff, size = 0x400 = 1,024
6. Keyboard: 0x100100c00 - 0x100100dff, size = 0x200 = 512

Complete address space: 0x0 - 0x100100dff, total address space size = 0x100100e00. This address space is called **Physical Address Space**.

The following is my laptop's physical address space.
</p>

```
-------------------------------------------------------------------------------
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
00100000-bd336fff : System RAM
bd337000-bd7b2fff : Reserved
bd7b3000-d9dbafff : System RAM
d9dbb000-d9ef2fff : Reserved
d9ef3000-d9f15fff : ACPI Tables
d9f16000-da686fff : System RAM
da687000-dade2fff : ACPI Non-volatile Storage
dade3000-dba62fff : Reserved
dba63000-dbafefff : Unknown E820 type
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
  101800000-102600e60 : Kernel code
  102600e61-10305063f : Kernel data
  10330b000-1037fffff : Kernel bss
11f800000-11fffffff : RAM buffer
-------------------------------------------------------------------------------
```
<p>
Look at that. There are a bunch of PCI buses, there is Video ROM, System RAM and lot more.

Coming back, CPU has now alloted addresses to all these devices. How exactly does communication happen? Communication is nothing but moving data to and from the device. Suppose I want to **write** some **data** into a particular **address**. The following is done.

1. Set Address Bus = **address**
2. Set Data Bus = **data**
3. Control Bus can be either set to **Read**/**Write**. In this example, we set it to **Write**.

Suppose the CPU wants to **read** from an **address**. The Address Bus is set to the **address** and control bus is set to **Read**. The device which has that address places **data** at that address.

In short, this is how communication happens.

Obviously, the CPU can't support infinite number of addresses. How many addresses can a CPU support? ie., What is the size of the physical address space it can support? **This depends on the size of the Address Bus**. If the address bus' size is 4 bits, then the size of physical address space is (2^4) = 16. If the address bus is 32 bits, then size of physical address space is 2^32 = 4294967296.

**The size of the data bus decides what type of processor it is**. If the length of data bus is 4-bits, then it is a 4-bit processor. Now a days, we mostly deal with 32-bit and 64-bit processors.

Coming back to our discussion on Instruction Sets, we will be exploring the x86 instruction set. The x86 instruction set is a 32-bit instruction set - the data-unit size is 32-bits. Intel and AMD implement these instruction sets in their processors. This instruction sets defines a rich set of instructions, a bunch of datatypes, various addressing modes, exception handling instructions, I/O methods and more. All these can be used by the programmer or can be used to build a compiler.
</p>

## 2.2 What is x86?

<p>
To answer this question, let us take a quick look at history of Intel's processors.

* **Intel 4004**: Intel's one of the first processors. It was a 4-bit processor.
* **Intel 8080**: 8-bit processor.
* **Intel 8086**: A 16-bit processor. Intel provided an [Instruction Set](https://web.itu.edu.tr/kesgin/mul06/intel/index.html). The 8080 was named to 808**6** because it is a 1**6**-bit processor. This is a very interesting processor. We will specifically talk about this at the end of this chapter.
* **Intel 80186**, **Intel 80286**: These are also 16-bit processors.
* **Intel APX 432**: First 32-bit microprocessor by Intel. This design was discontinued.
* **Intel 80386**: A 32-bit microprocessor. This because very famous in the market. It made its mark.
* **Intel 80486**: 80386's successor.
* Then came the legendary **Intel 80586** or **Pentium**!
* Then came Pentium Pro, Pentium II, Pentium III etc.,
* Soon after that, 64-bit processors came into market.

The point is, 8086 had a 16-bit ISA. When Intel introduced 32-bit microprocessors(80386 and later), they came up with a 32-bit ISA which was an **Extension** of the older 16-bit ISA.

8086, 80186, 80286, 80386, 80486, 80586 are the series of microprocessors. Later, they named the ISA as **x86** ISA where **86** stands for the actual 86 in the series and x is like a variable here.

What does extensions mean?

A few new instructions are added to the old ISA to support the new hardware design. This means, all the old instructions will be able to run on the new processor unless they have removed it from the new ISA for some reason. This is called **Backward Compatibility**.
</p>

## 2.3 The x86 ISA { #x86 }

### 2.3.1 Registers

</p>
Before moving to instructions, we need to understand what a **Register** is.

A **Register** is a small storage space **on the chip** of the microprocessor. Generally, there will be multiple registers on the chip. All the registers collectively is known as **Register File**.

Why do you need such storage when there is a proper memory unit? Because the memory unit is outside the microprocessor, it takes certain amount of time(aka latency) to access it, it is better to have on-chip storage for fast access.

The x86 ISA provides **8** registers each of size **32-bits**. They are

1. **eax**: The **a** stands for **accumulator**. An accumulator is a register which is used to store results of certain operations.
2. **ecx**: **c** stands for **counter**. Certain instructions used this register as counter(or iterator) in loops.

3. **edx**: **d** stands for **data**.
4. **ebx**: **b** stands for **base**. Certain instructions store the base value in this register.
5. **esi**: **Source Index** register. Used in certain string related instructions.
6. **edi**: **Destination Index** register. Used in certain string instructions. 
7. **ebp**: **Base Pointer**
8. **esp**: **Stack Pointer**

It should be noted that **only** certain instructions use each of these registers in a particular manner which has been mentioned.

These registers are known as **General Purpose Registers(GPRs)**, but **ebp** and **rsp** are almost never used for general purposes. They have very specific purposes.

Along with the above registers, there are 2 more registers - **eip(Instruction Pointer)** and **eflags**. They have very specific purpose and that is why they are called **Special Purpose Registers**.

Some history about the ISA. When 8-bit processors came into the market, registers were named **a**, **b**, **c**, **d**. They were renamed as **ax**, **bx**, **cx**, **dx** for 16-bit processors. The **x** stands for e**x**tended. When 32-bit processors were designed, the 16-bit set was extended. The **e** in each of those 32-bit registers means **e**xtended.
</p>

```
----------------------------------------------------------------------Registers
|-----------------------------------|
|   32   |   24   |   16   |   8    |
|-----------------------------------|
|                EAX                |
|-----------------------------------|
|                 |       AX        |
|-----------------------------------|
|                 |   AH   |   AL   |
|-----------------------------------|
-------------------------------------------------------------------------------
```
<p>
A programmer(or compiler) can use 8-bit, 16-bit versions of registers in 32-bit programs. For example, **ax**, **si**, **al** can be used in 32-bit programs. You will understand this better when we write programs.

Now coming to the special purpose registers,

1. **eip**:
* Instruction Pointer
* The register stores the address of the next instruction to be executed.

2. **eflags**:
* This is a set of **status** flags(or bits). Each flag can either be 0(not set) or 1(set). The following are a few flags.
        * **zf** - The zero flag: This is set when the result of an operation is zero. If the result is not zero, it is cleared.
        * **cf** - The carry flag: This is set when the result of an operation is too small or too large for destination operand.
        * **sf** - The sign flag: This is set when the result of an operation is negative. If the result is positive, it is cleared. The flag value is equal to the **most-significant bit** of the result(in 2's complement representation).
</p>

### 2.3.2 Instructions

<p>
Instructions can be broadly classified into 4 types:

1. Arithmetic and Logical istructions: ```add```, ```sub```, ```imul```, ```idiv```, ```or```, ```xor```, ```and``` etc.,
2. Data flow instructions: ```mov```, ```lea```, ```push```, ```pop```.
3. Control Flow instructions: ```cmp```, ```jmp``` and its siblings, ```call```, ```ret```.
4. Interrupts: ```syscall```, ```int```, ```sysenter```.

We will take a look at each instruction in detail in the next section.
</p>

## 2.4 x86 Assembly Language

### 2.4.1 Difference between ISA and Assembly language

<p>
What we discussed so far was about x86 ISA. What is the difference between ISA and assembly language?

The ISA is a document which has a list of all the registers, instructions, addressing modes etc., Assembly language is a programming language made out of the ISA. To understand this, let us go back to our toy ISA in [Chapter1's Init](#Init1.1). It has 256 instructions, operands must be memory addresses, 1 datatype of 2 bytes. We took that Instruction Set and slowly came up with an assembly language. We split the program into data and text sections, programmers were allowed to use variable-names, labels and the assembler took care of it. Similar to this, modern assemblers have a lot more features - macros, directives, different syntax of the same ISA. A standard is converted into a language.

Though there is only one x86 ISA, there are a variety of assemblers which offer their own versions of the x86 assembly language. There are a few x86 assemblers out there like [Netwide Assembler(nasm)](https://nasm.us/), [Microsoft Assembler(masm)](https://docs.microsoft.com/en-us/cpp/assembler/masm/microsoft-macro-assembler-reference?view=vs-2019), [Turbo Assembler(tasm)](https://en.wikipedia.org/wiki/Turbo_Assembler), [Yasm Modular Assembler(yasm)](https://yasm.tortall.net/), [GNU Portable Assembler(as)](https://linux.die.net/man/1/as) etc., Each of these assemblers offers a variety of features. We will be using **nasm** throughout the chapter.

Installing **nasm** is just one command away.
</p>

```
-------------------------------------------------------------------------------
$ sudo apt-get install nasm
-------------------------------------------------------------------------------
```

### 2.4.2 Datatypes

<p>
At the assembly level, we will be dealing with bytes. Datatypes like **char**, **int**, **long int** etc., are not present at assembly level. Then, how are these datatypes translated to their assembly equivalent? This is done by **accessing the specific number of bytes a particular datatype in C represents**. Let us take a few examples to understand this.

In 32 systems, ```sizeof(short int)``` = **2 bytes**. As the assembly level, there is no integer datatype. There is only a stream of bytes. Suppose **a** is an ***short int** variable whose address is present in the register **ebx**. If you want to load the contents of variable **a** into another register(say **cx**), this is how its done.
</p>

```
-------------------------------------------------------------------------------
mov cx, word [ebx]
-------------------------------------------------------------------------------
```

<p>
* ```mov``` is the instruction.
* ```cx``` is the destination register.
* A **word** means **2** bytes.
* The instruction tells the assembler to consider **ebx** to be a word pointer. That is, assume that it points to 2 bytes. So, when it is used in an instruction, 4 bytes pointed by ebx is loaded.

Let us take another example.

Let **b** be the **int** variable. In a 32-bit machine, ```sizeof(int)``` = **4 bytes**. Suppose you want to perform an operation ```b = b + 0x123```. Assuming address of **b** is present in **rax**, this is how its done.
</p>

```
-------------------------------------------------------------------------------
add dword[rax], 0x123
-------------------------------------------------------------------------------
```

<p>
* ```add``` is the instruction.
* 0x123 is the constant to be added.
* **dword** is short for **double word** which is 4 bytes.
* This instruction tells the assembler to consider **rax** to be a double word pointer. That is, consider the first 4 bytes pointed by it. So, when it is used in this instruction, all 4 bytes are taken, 0x123 is added to it and then put back into the memory.

Let us put these in a formal manner.

1. How is the size of data measured at assembly level?
* **1 byte** is the small piece of memory that can be accessed.
* **byte** stands for 1 byte.
* **word** stands for 2 bytes.
* **dword / double word** stands for 4 bytes.

Remember that the data-bus size of a 32-bit system is 32-bits. This means you can bring in a maximum of 32-bits of data at a time from RAM. 8-bits, 16-bits of data can also be brought in using a 32-bit long data bus. That is the idea.

2. The following are the methods to access memory.
* ```byte[REG]```: This tells the assembler to consider **REG** as a byte pointer. Consider the first byte it is pointing to. Any operation performed with this as an operand will take only 1 byte directly pointed by **REG**.
* ```word[REG]```: This tells the assembler to consider **REG** as a word pointer.
*  ```dword[REG]```: Tells the assembler to consider 4 bytes pointed by **REG**.
</p>

### 2.4.3 Instructions

<p>
In the previous section, we listed a few instructions. Now, let us see how exactly to use them, their operands.

Most of the instruction operand on operand(s). These operands can be

1. **Registers**: These are mostly general purpose registers discussed in the previous section. There are a few special registers which are also used, we will discuss about them later.
2. **Immediate Values**: These are constants like 100, 0x1234, 0x9484 etc.,
3. **Addresses**: There are 2 ways to represent addresses.
* A Immediate value - a direct number which can be used as an address.
* A pointer: The address is loaded into one of the registers and then used as explained in the previous subsection.

Most of the instructions are of the following form.
</p>

```
-------------------------------------------------------------------------------
Ins Destination, Source
-------------------------------------------------------------------------------
```

<p>
The first operand is Destination and second is the source.

Let us start with the Arithmetic and Logical instructions.
</p>

#### 2.4.3.1 Arithmetic and Logical instructions

<p>
**1. add**: Used to add 2 operands. The general syntax is as follows.

* ```add Reg, Reg```
* ```add Reg, Imm```
* ```add Mem, Reg```
* ```add Reg, Mem```
* ```add Mem, Imm```

Examples:

* ```add eax, ebx```: Adds values in **eax** and **ebx** and stored it back in ```eax```. Note that ```eax``` is the destination. ```eax = eax + ebx```.
* ```add rax, 0x123```: Adds 0x123 to value in rax and stores it in ```rax```. ```rax = rax + 0x123```.
* ```add dword[ebx], eax```: Adds value in register **eax** with 4 bytes at memory pointed by **ebx**. The result is also 4 bytes stored back at the memory pointed by **ebx**.
* ```add eax, dword[ebx]```: Adds value in **eax** with 4 bytes of memory pointed by **ebx**. Stores back the value into **eax**.

It is important to note that both operands cannot be memory locations. 

**2. sub**: This is an instruction to find the difference between 2 operands. The syntax is similar to ```add``` instruction.

**3. imul**: Integer multiplication.

* ```imul``` can have 2 or 3 operands.

The syntax is as follows.

* ```imul op1, op2```: **op1** and **op2** are multiplied and then stored back in **op1**. Note that **op1** must be a register.
* ```imul op1, op2, op3```: **op2** and **op3** are multiplied and then stored in **op1**. **op1** must be a register and **op3** must be an immediate value.

**5.** **or**, **xor**, **and** instructions do bitwise operations between 2 operands.  The pair of operands are the same as that for the add instruction.

**6. inc**: Increment the operand by **1**.
* The syntax is ```inc Operand```, the operand can be a register or a memory location.

**7. dec**: Decrement the operand by **1**. Syntax is same as **inc**.

These are the instructions which are used in mostly every program.
</p>

#### 2.4.3.2 Data Flow instructions

<p>
Let us take at 4 very common instructions: **mov**, **lea**, **push** and **pop**.

**1. mov**: Though the instruction name is **mov**, it is used to copy data from source to operand.

This is the syntax: ```mov Destination, Source```.

* ```mov Reg, Reg```
* ```mov Reg, Imm```
* ```mov Reg, Mem```
* ```mov Mem, Reg```
* ```mov Mem, Imm```

Note that there is no **memory to memory** move instruction. At the assembly is accessed through pointers. Generally, the **memory address is loaded into a register** and then it is accessed. In some cases, direct valid addresses are used.

Here are a few examples.

* Accessing a **char** variable.
    * Suppose we have a C variable ```char a = 'x'```. **a** is a character variable => Its size is **1 byte**. At the assembly level, this is how it is accessed.
    * Load the address of variable **a** into any register(say eax).
    * ```byte[eax]```: Refers to 1 byte of memory pointed by eax.
    * ```mov bl, byte[eax]```: Copies 1 byte pointed by eax into **bl** => **bl** will have the value of **a** after this instruction is executed.

* Accessing a **short int** variable.
    * Suppose we have a variable **short int s = 123**. The size of short int is **2 bytes**.
    * Load the address of variable **s** into a register(say ecx).
    *  ```word[ecx]```: Refers to 2 bytes of memory pointed by ecx. As size of **s** is 2 bytes, ```word[ecx]``` points to **s**.
    * ```mov ax, word[ecx]``` copies variable **s** into **ax**.

* Similar to this, **int** variable can be copied using **dword**.

If you observe the above **mov** instructions, the **size of both operands are always same**. This is important. If there is a size mismatch, you get an assembler error.

Consider ```mov dx, dword[ecx]```: The assembler refuses to assemble this because size of **dx** is 2 bytes and **dword** means 4 bytes. You are trying to copy 4 bytes into a 2-byte register.

**2. lea**: Load Effective Address

While discussing the **mov** instruction, we assumed that the address of a variable is already in a register. Later, we used to access the data. But how do we load the address of a variable into a given register? You can use the **lea** instruction.

The syntax is ```lea Destination, Source```. A few examples.

* Suppose **a** is a variable and its address needs to be loaded to **rbx**. It can done like this: ```lea rbx, [a]```.

**3. push**

Every process is given a **runtime stack** which can be used as a scratch-pad by the process. One example of this is local variables of a function. They should be in the main memory only till that function is executing. Once it returns, they should be deallocated. They can be pushed before the function starts executing and can be popped when the function is done. In 32-bit systems, the stack is 4-byte aligned. The register **esp**(Stack Pointer) always points to the top of this stack.
</p>

```
-------------------------------------------------------------------------------
<------4 bytes------>
|----|----|----|----|
|    |    |    |    | <------------ esp
|----|----|----|----|
|    |    |    |    |
|----|----|----|----|
|    |    |    |    |
|----|----|----|----|
          .
          .
          .
-------------------------------------------------------------------------------
```

<p>
Its syntax is ```push Operand```. Whatever is pushed onto stack, it is pushed as 4 bytes because the stack is 4-byte aligned. Suppose the value in **ax** is 0x1234 and ```push ax``` is executed. Top of stack would be 0x00001234. It should be noted that **esp** is changed accordingly when push is executed. Note that **esp** always points to the top of the stack.

**4. pop**

You pop 4 bytes from the stack into the operand. The syntax is ```pop operand```.
</p>

#### 2.4.3.3 Control Flow instructions

<p>
All the instructions are stored as an array in memory. During execution, the instruction pointer(eip) always points to the next instruction. It is important to note that the instruction next to the current instruction in memory may not be the next instruction to get executed. That is what happens in conditional statements, loops, function calls etc., There should be instructions which will help implement these conditional statements, loops, functions etc.,

**1. cmp**: Compares 2 operands

* Syntax: ```cmp op1, op2```
* Checks **op1** with respect to **op2** and sets the appropriate flags(in eflags).

**2. jmp** and its derivatives

* Syntax: ```jmp Address```
* The address can be a number which is a valid address. While writing code, labels are used.
* This instruction is like **goto** at C level.
    * Example: ```jmp Label```: When this is executed, the instruction at **Label** is executed. It is an **unconditional jump** instruction.

Obviously, mostly we would want conditional jumps ie., we want to jump if some condition is true or false. That is where **jmp**'s derivatives come in.

* The following are a few conditional jump instructions which should be used after a **cmp** instruction. ```cmp op1, op2```. Then,
    * **je**: Jump if op1 == op2
    * **jne**: Jump if op1 != op2
    * **jle**: Jump if op1 <= op2
    * **jge**: Jump if op1 >= op2
    * **jg**: Jump if op1 > op2
    * **jl**: Jump if op1 < op2

**3. call**: Executed to call a function

* Syntax: ```call function_name```
* The call instruction is a sequence of 2 instructions.
</p>

```
-------------------------------------------------------------------------------
push return_address
jmp function_name
-------------------------------------------------------------------------------
```

<p>
**4. ret**: Executed by **callee** function to return back to caller function.
* Syntax: ```ret```
* The ret instruction actually means
</p>

```
-------------------------------------------------------------------------------
pop hidden_reg
jmp hidden_reg
-------------------------------------------------------------------------------
```

<p>
With that, we have covered some common x86 instructions. The x86 ISA has unbelievably large number of instructions. There are variants of the same instruction, there are floating point instructions, there are vector-instructions and more. This is just an introduction. Learning about new, crazy instructions is an exercise.
</p>

### 2.4.4 Variables

<p>
We will be using the [nasm](https://nasm.us/) assembler. It provides a certain way to declare, define variables.

**1**. Global initialized variables - these go into the .data section.

* ```var1: db 0x12``` : var1 is a variable of size 1 byte initialized to a value 0x12.
* ```var2: dw 0x1234```: var2 is a 2-byte variable initialized to a value 0x1234.
* ```var3: dd 0x120a0b33```: var3 is a 4-byte variable initialized to a value 0x120a0b33.
* ```var4: dq 0x1223344556677889```: var4 is a 8-byte variable initialized to that value.

This is how you initialize strings(NULL-terminated character array).
* ```str: db "Hello World!, I am learning assembly", 0x0a, 0x00```.

**2**. Uninitialized variables

* ```buffer: resb 1000```: Reserve 1000 bytes
* ```buffer: resw 1000```: Reserve 1000 words - 2000 bytes
* ```buffer: resb 1```: Reserve 1 byte - probably to store a character.
* ```buffer: resd 1```: Reserve a double word - probably to store an integer.

Note that there are no datatypes like char, int etc., These are just array of bytes. We have defined int to be 4 bytes. This means, we can reserve a dword in memory and store something there, which we call it an integer.

With that, we have discussed a bunch of instructions, variable declaration-definition techniques.
</p>

### 2.4.5 Passing arguments and returning values

</p>
To write programs, we need more. We write functions to get work done. We don't know how arguments are passed to the caller, how values are returned to the callee. The following are standard rules.

* Push the arguments onto the stack.
* Put the return value in **eax** and execute **ret**.

There is a lot to discuss about this. We will take this up in one of the future chapters.
</p>

## 2.5 Hello World!

<p>
We are ready to write our first assembly program! It is a simple hello world program.

Let us write the code in a *hello.s*. It is going to have 2 sections. A .rodata section consisting of the string and a .text section with code.

**1.** Let us define our string.
</p>

```asm
-------------------------------------------------------------------------------
section .rodata

str: db "Hello World!", 0x00
-------------------------------------------------------------------------------
```

<p>
**2.** Let us write some code.

* Our program begins with **_start**. The only thing we need to is print the string and exit.
* Let us write the instructions to print the string.
    * **puts** is a common string printing function. It takes in one argument - pointer to a constant character string. Take a look at its manual page.
</p>

```asm
-------------------------------------------------------------------------------
section .text
    global _start

_start:
    push str    ; Arg: Pointer to our string
    call puts   ; Call the function 
-------------------------------------------------------------------------------
```

<p>
Once it is printed, let us exit out of the program - ```exit(0)```.
</p>

```asm
-------------------------------------------------------------------------------
    push 0      ; Argument for exit
    call exit   ; Call it
-------------------------------------------------------------------------------
```

<p>
Let us assemble it using nasm.
</p>

```
-------------------------------------------------------------------------------
chapter2$ nasm hello.s -f elf32
hello.s:10: error: symbol `puts' undefined
hello.s:13: error: symbol `exit' undefined
-------------------------------------------------------------------------------
```

<p>
The **-f** option is used to specify the relocatable file format. Here, we want to generate a 32-bit ELF executable in the end => We need to generate a 32-bit ELF relocatable file. You can run **nasm -fh** to list the bunch of target formats.

Here, we are getting 2 undefined symbols error. we need to inform the assembler that these 2 are found in some other relocatable file or in some library. The assembler should put relocation records(fixing table entries) for these 2 symbols.

You can use the **extern** keyword in nasm to inform it that these symbols are defined elsewhere and you want the linker to take care of it.

The following is the complete listing of the program.
</p>

```asm
------------------------------------------------------------------------hello.s
section .rodata

str: db "Hello World!", 0x00

extern puts
extern exit

section .text
	global _start

_start:
	push str	; Arg: Pointer to our string
	call puts	; Call the function

	push 0		; Arg: 0
	call exit	; Call it
-------------------------------------------------------------------------------
```

<p>
Let us assemble it.
</p>

```
-------------------------------------------------------------------------------
chapter2$ nasm hello.s -f elf32
chapter2$ file hello.o
hello.o: ELF 32-bit LSB relocatable, Intel 80386, version 1 (SYSV), not stripped
-------------------------------------------------------------------------------
```

<p>
It generated *code1.o*, an 32-bit ELF relocatable file.Now let us link it.
</p>

```
-------------------------------------------------------------------------------
chapter2$ ld hello.o -o hello -melf_i386
hello.o: In function `_start':
hello.s:(.text+0x6): undefined reference to `puts'
hello.s:(.text+0xd): undefined reference to `exit'
-------------------------------------------------------------------------------
```

<p>
**ld** is the GNU Linker. The **-m** option specifies the type of executable we want. We want an i386(basically Intel 32-bit) executable.

We got a linking error. We know that the linker should either find the definitions in other relocatable files or libraries. In our case, there are no other relocatable files. If they are present in the libraries, they linker will add fixing table entries and leave to to get resolved at runtime. At runtime, who resolves these symbols? To do this runtime linking, there is a helper program called the **dynamic linker**. As the name suggests, it does dynamic linking. We know that **puts** and **exit** belong to the standard C library. While linking, we need to specify 2 things. The library these symbols can be found and the dynamic linker.

The dynamic linker is found at ```/lib/ld-linux.so.2``` and the library we want to link our program **against** is **libc**. Let us go ahead and do that.
</p>

```
-------------------------------------------------------------------------------
chapter2$ ld hello.o -o hello --dynamic-linker /lib/ld-linux.so.2 --library c -melf_i386
chapter2$ file hello
hello: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked,
interpreter /lib/ld-, not stripped
-------------------------------------------------------------------------------
```

<p>
You specify the dynamic-linker using the **--dynamic-linker** option, the library using the **--library** option. If you don't get any errors, you got yourself an executable. Try running it.
</p>

```
-------------------------------------------------------------------------------
chapter2$ ./hello 
Hello World!
-------------------------------------------------------------------------------
```

<p>
There you go! Your first assembly program in action!

I want you to inspect *hello.o*(the relocatable file) and *hello*. Take a look at their sections, whats added - the symbol table, fixing table etc.,
</p>

### 2.5.1 Hello World! - Part2

<p>
Let us a modified version of the hello program, one which prints one character at a time.

Just have in mind how you would a C program to do this. You can have a while loop, a character variable traversing through the string till it hits NULL. 

Lets call the program *hello1.s*.

Let us start with the variables. We would have a the hello string and the **format string** for printf.
</p>

```asm
-------------------------------------------------------------------------------
section .rodata

str: db "Hello World!", 0x00
format_string: db "%c", 0x0a, 0x00
-------------------------------------------------------------------------------
```

<p>
Now onto the .text section. Let us start with getting a pointer the string to a string and initializing a register to 0.
</p>

```asm
-------------------------------------------------------------------------------
_start:
	xor ebx, ebx		; Clean up the register
	lea esi, [str]		; Get a pointer to our string
-------------------------------------------------------------------------------
```

<p>
Let us start with our loop.
</p>

```asm
-------------------------------------------------------------------------------
loop:	
	mov bl, byte[esi]	; Get the byte
	cmp bl, 0x00		; Compare it with NULL
	je loop_out		; If it is NULL, then we have hit the end. Get out!
-------------------------------------------------------------------------------
```

<p>
That was the loop initialization. Let us come to the body of the loop.
</p>

```asm
-------------------------------------------------------------------------------
	; Print the character	; printf("%c\n", c);
	push ebx		; Second argument = the character itself
	push format_string	; First argument = address of "%c\n"
	call printf		; Call it
-------------------------------------------------------------------------------
```
<p>
Look at the order in which the arguments are pushed. The last argument is pushed first. This is a standard followed.

Let us write the last part of the loop.
</p>

```asm
-------------------------------------------------------------------------------
	inc esi			; Increment the pointer
	jmp loop		; Jump back to the beginning
-------------------------------------------------------------------------------
```

<p>
You increment the pointer and jump back the beginning of the loop.

**loop_out** has a call to the exit function.

The following is the complete listing.
</p>

```asm
-----------------------------------------------------------------------hello1.s
section .rodata

str: db "Hello World!", 0x00
format_string: db "%c", 0x0a, 0x00

extern printf
extern exit

section .text
	global _start

_start:
	xor ebx, ebx		; Clean up the register
	lea esi, [str]		; Get a pointer to our string
loop:	
	mov bl, byte[esi]	; Get the byte
	cmp bl, 0x00		; Compare it with NULL
	je loop_out		; If it is NULL, then we have hit the end. Get out!

	; Print the character	; printf("%c\n", c);
	push ebx		; Second argument = the character itself
	push format_string	; First argument = address of "%c\n"
	call printf		; Call it
	
	inc esi			; Increment the pointer
	jmp loop		; Jump back to the beginning

loop_out:
	; Exit
	push 0		; Arg: 0
	call exit	; Call it
-------------------------------------------------------------------------------
```

<p>
Assemble and link it.
</p>

```
-------------------------------------------------------------------------------
chapter2$ nasm hello1.s -f elf32
chapter2$ ld hello1.o -o hello1 --dynamic-linker /lib/ld-linux.so.2 --library c -melf_i386
-------------------------------------------------------------------------------
```

<p>
And run it!
</p>

```
-------------------------------------------------------------------------------
chapter2$ ./hello1
H
e
l
l
o
 
W
o
r
l
d
!
-------------------------------------------------------------------------------
```

<P>
That was the second.

With that, we have come to the end of this section. I encourage you to practice because we will be reading lot of assembly code and writing some too.
</p>

## 2.6 The x86_64 ISA

<p>
So far, we explored the 32-bit x86 instruction set. Now, let us take a look at its 64-bit extension - the x86_64 ISA.
</p>

### 2.6.1 Registers

<p>
The x86 ISA offered 8 general purpose registers along with 2 special purpose registers. The x86_64 ISA provides 16 general purpose registers with 2 special registers.

Size of each of these GPRs is **8** bytes. They are **rax**, **rcx**, **rdx**, **rbx**, **rsp**, **rbp**, **rsi**, **rdi**, **r8**, **r9**, **r10**, **r11**, **r12**, **r13**, **r14** and **r15**. The first 8 are direct 64-bit extensions of their corresponding 32-bit registers. The last 8 are newly added to the ISA.

The 2 special purpose registers are **rip** and **eflags**. **rip** is the instruction pointer, 8 bytes in size. **eflags** is still a 32-bit register.

The **r** simply stands for register.
</p>

```
-------------------------------------------------------------------------------
|-----------------------------------------------------------------------|
|   64   |   56   |   48   |   40   |   32   |   24   |   16   |   8    |
|-----------------------------------------------------------------------|
|                                 RAX                                   |
|-----------------------------------------------------------------------|
|                                   |               EAX                 |
|-----------------------------------------------------------------------|
|                                                     |       AX        |
|-----------------------------------------------------------------------|
|                                                     |   AH   |   AL   |
|-----------------------------------------------------------------------|


|-----------------------------------------------------------------------|
|   64   |   56   |   48   |   40   |   32   |   24   |   16   |   8    |
|-----------------------------------------------------------------------|
|                                 R8                                    |
|-----------------------------------------------------------------------|
|                                   |               R8D                 |
|-----------------------------------------------------------------------|
|                                                     |       R8W       |
|-----------------------------------------------------------------------|
|                                                              |   R8B  |
|-----------------------------------------------------------------------|

In the above diagram, R8 can be replaced with R9-R15.
-------------------------------------------------------------------------------
```

### 2.6.2 Datatypes

<p>
We have seen **byte**, **word** and **dword**. One more addition to this. Because the size of data bus is 8 bytes. a new type is added - **qword** / **quad word** which is 8 bytes. Note that all the 4 can be used in a 64-bit program.
</p>

### 2.6.3 Instructions

<p>
Most of the instructions discussed in the previous chapter are part this ISA too. There are a few changes, extensions we need to look at.
</p>

#### 2.6.3.1 push and pop

<p>
The runtime stack in a 64-bit system is 8-byte aligned. 
The stack can be viewed like this.
</p>

```
-------------------------------------------------------------------------------
<---------------8 bytes----------------->
|----|----|----|----|----|----|----|----|
|    |    |    |    |    |    |    |    | <-----rsp
|----|----|----|----|----|----|----|----|
|    |    |    |    |    |    |    |    |
|----|----|----|----|----|----|----|----|
|    |    |    |    |    |    |    |    |
|----|----|----|----|----|----|----|----|
                    .
                    .
                    .
-------------------------------------------------------------------------------
```

<p>
Because the stack is 8-byte aligned, only 8-byte operands can be pushed onto stack and whatever you pop, the destination should be 8 bytes long. Registers rax, r8, r15 can be pushed and popped into but using eax, cl, bx gives an assembler error.

Rest of the instructions we discussed have no changes. Note that data can be 8 bytes long. You can use **qword** in the data movement instructions.
</p>

### 2.6.4 Passing arguments and returning values

<p>
In x86, passing arguments was very easy. You had to push arguments onto stack. In 64-bit, this is how it is done. Because there are a lot more registers, the first few arguments are passed through registers. Rest of them(if there are so many) are pushed to the stack.

1. Arg1 into **rdi**
2. Arg2 into **rsi**
3. Arg3 into **rdx**
4. Arg4 into **rcx**
5. Arg5 into **r8**
6. Arg6 into **r9**
7. If there are more, push them onto stack.
</p>

### 2.6.5 Hello World in x64

<p>
Let us rewrite the second hello program *hello1.s* in x64 assembly language. Lets start with defining the necessary strings in the .rodata section.
</p>

```asm
-------------------------------------------------------------------------------
section .rodata

str: db "Hello World!", 0x00
format_string: db "%c", 0x0a, 0x00
-------------------------------------------------------------------------------
```
<p>
Let us head to the .text section. Start with initializing registers.
</p>

```asm
-------------------------------------------------------------------------------
section .text
	global _start

_start:
	xor r15, r15		; Clean up the register
	lea r14, [str]		; Get a pointer to our string
-------------------------------------------------------------------------------
```

<p>
Let us start the loop.
</p>

```asm
-------------------------------------------------------------------------------
loop:	
	mov r15b, byte[r14]	; Get the byte
	cmp r15b, 0x00		; Compare it with NULL
	je loop_out		; If it is NULL, then we have hit the end. Get out!
-------------------------------------------------------------------------------
```

<p>
Now is the interesting part, the body of the loop.
</p>

```asm
-------------------------------------------------------------------------------
	; Print the character	; printf("%c\n", c);
	lea rdi, [format_string]; Arg1: Address of format string
	mov rsi, r15		; Arg2: The character
	call printf		; Call it
-------------------------------------------------------------------------------
```

<p>
Look at the way arguments are passed. They are being passed through registers, unlike through stack in the 32-bit program. Remember the register order in which arguments are passed.

End the loop and exit.
</p>

```asm
-------------------------------------------------------------------------------
	inc r14			; Increment the pointer
	jmp loop		; Jump back to the beginning

loop_out:
	; Exit
	push 0		; Arg: 0
	call exit	; Call it
-------------------------------------------------------------------------------
```

<p>
Note that **printf** and **exit** are part of **libc**. Use the **extern** keyword and inform the assembler. The following is the complete listing.
</p>

```asm
--------------------------------------------------------------------hello1_64.s
section .rodata

str: db "Hello World!", 0x00
format_string: db "%c", 0x0a, 0x00

extern printf
extern exit

section .text
        global _start

_start:
        xor r15, r15            ; Clean up the register
        lea r14, [str]          ; Get a pointer to our string
loop:
        mov r15b, byte[r14]     ; Get the byte
        cmp r15b, 0x00          ; Compare it with NULL
        je loop_out             ; If it is NULL, then we have hit the end. Get out!

        ; Print the character   ; printf("%c\n", c);
        lea rdi, [format_string]; Arg1: Address of format string
        mov rsi, r15            ; Arg2: The character
        call printf             ; Call it

        inc r14                 ; Increment the pointer
        jmp loop                ; Jump back to the beginning

loop_out:
        ; Exit
        push 0          ; Arg: 0
        call exit       ; Call it
-------------------------------------------------------------------------------
```

<p>
Let us assemble it.
</p>

```
-------------------------------------------------------------------------------
chapter2$ nasm hello1_64.s -f elf64
-------------------------------------------------------------------------------
```

<p>
Now to linking. The way there was a dynamic linker for 32-bit programs, there is a dynamic linker for 64-bit programs too. By default, ```/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2```. Let us go ahead and link it up.
</p>

```
-------------------------------------------------------------------------------
chapter2$ ld hello1_64.o -o hello1_64 -lc --dynamic-linker 
/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
-------------------------------------------------------------------------------
```

Go ahead and run it!

```
-------------------------------------------------------------------------------
chapter2$ ./hello1_64 
H
e
l
l
o
 
W
o
r
l
d
!
-------------------------------------------------------------------------------
```

## 2.7 A couple of things

### 2.7.1 Measuring bytes and spaces

<p>
Hardware manufacturers and programmers, operating system designers use different units to measure memory. 1 GB harddisk means it is a 1 Gigabytes harddisk. Here, 1 GB = 10^9 bytes. In operating systems, everything is a power of 2. If someone mentions 1GB, he probably means (2^10)^3 bytes. This is an official unit called the **Gibibyte(GiB)**

* 1KB = 1,000B, 1KiB = 1,024B
* 1MB = 1,000,000B, 1MiB = 1024*2 = 1,048,576B
* 1GB = 1,000,000,000B, 1GiB = 1024^3 = 1,073,741,824B

This continues: Tebibytes, Pebibytes, Exbibytes...
</p>

### 2.7.2 On the 8086 Intel processor

<p>
The Intel 8086 processor is very interesting. It was originally a 16-bit processor => Its data bus size was 16 bits. Its address bus size was also 16 bits. This meant it could address a maximum of 2^16 = 65536 bytes - 64KiB. Programmers wanted more system space, more memory but the processor could handle only 64KiB. There was a need for the processor manufacturers to support more memory. That is when they came up with the concept of **segmentation**. In this new design, the size of the address was 20-bits ie ie., it could address a maximum of 1MiB which is 16 times of 64KiB. Let us have a quick look at how this works.

The complete address space is divided into **segments**. Each byte in the address space has an address of the form **Segment : Offset** where **Segment** contains that byte at an offset of **Offset**. 

This is when the x86 ISA was introduced with 4 new registers:

1. **ds** - Data Segment
2. **cs** - Code Segment
3. **ss** - Stack Segment
4. **es** - Extra Segment

These are called **segment registers**. Each of these were supposed to point to their segments.

Later, 2 more were introduced - **fs** and **gs**. 

Now, segmentation is outdated but a few of these registers are still used for special purposes.
</p>

### 2.7.3 What all does the ISA contain?

<p>
What we discussed was just tip of the iceburg. The x86 ISA has lots of instructions for floating point operations, vector operations, string operations, cryptographic instructions like sha1, aes etc.,

It has a bunch of other general purpose registers too which are there to handle floating point operations.

It should be noted that the ISA is what the programmer/compiler can see. The processor has a lot of registers, buffers, storage on the chip of the microprocessor which are not exposed to the programmers. **Control Registers** is a class of registers which controls the behavior of a CPU. These control registers have info about paging, interrupts etc.,

[This](https://software.intel.com/en-us/articles/intel-sdm#three-volume) is the official link where you will find x86 software developer manuals. It has the instruction set, it has a system programming guide where it describes memory management, protection, multi-processing, virtualization and a lot more.
</p>

### 2.7.4 Different Syntaxes

<p>
There are 2 prominent assembly language syntaxes out there. They are **AT&T Syntax** and the **Intel Syntax**. We used the Intel syntax in this chapter. GNU toolchain by default uses the AT&T syntax. We used Intel Syntax because it is simple and code is less noisy. Read [this article](https://web.mit.edu/rhel-doc/3/rhel-as-en-3/i386-syntax.html) to know more.
</p>

### 2.7.5 What exactly is the dynamic linker?

For me, the operating system is like god, your program is a baby, the dynamic linker is like the god-given parent to the baby. The dynamic linker is a helper program that makes sure your program runs properly. It loads your program into memory, the dependent libraries into memory, prepares the program to run and it runs it. It resolves links symbols dynamically whenever necessary. We will specifically explore the dynamic linker in one of the future chapters. To learn more, you can read its manual page - ```man ld.so```. Try running the dynamic linker and you will get more details.

## 2.8 Fini
<p>
We have come to the end of this chapter.

In this chapter, we explored the x86 and x64 ISA and the x86 assembly language provided by the nasm assembler. We explored how CPU communicates with devices, phyical address spaces, bunch of common instructions, registers and finally wrote 2 programs.

We tied up a few loose ends present in the [previous chapter](#Chapter1) but there are still a lot of them. As we progress, we will tie up the rest.
</p>

## 2.8 Further Reading

<p>
1. [Abstract Machines for programming language implementation](http://www.inf.ed.ac.uk/teaching/courses/lsi/diehl_abstract_machines.pdf): Whatever we are exploring is basically part of programming languages. This is an interesting paper.
2. [The 8051 Microcontroller and Embedded Systems: Using assembly and C](http://www.iust.ac.ir/files/ee/pages/az/mazidi.pdf)
3. [The Intel Software Developer Manuals](https://software.intel.com/en-us/articles/intel-sdm#three-volume)
</p>