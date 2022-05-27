# Chapter 1: Internals of Compiling { #Chapter1 }

----------------------------------------------------------------------------------------------------
**Summary**
This chapter explores how a C program is converted into a file that can be run on the machine. The
conversion is a multi-step process which involves lot of interesting things including the crazy 
Executable and Linkable Format(ELF). All of these are explored in detail.
----------------------------------------------------------------------------------------------------

## 1.1 Init { #Sec1.1 }

<p>

Let us start with a question. How was the first program ever written? Computers were mechanical machines back then. Handles and gears were used to program them. As time progressed, handles and gears were replaced by electrical signals. Later, a computer was designed to be programmed by just 0s and 1s - the binary language. Because this code is directly run on the machine, it is called **machine code**. This was amazing progress. But it was still not convenient because humans could not read that code, it was hard to debug and hard to write too. Just one mistake and it had to be written all over again. It looked like the following diagram.

</p>

```
-------------------------------------------------------------------------------
 Binary Program
|--------------|               |-------------------|
|00011110101010|               |                   |
|11101101101101| ------------> | Processor/Computer|
|01010101010010|               |                   |
|--------------|               |-------------------|
-------------------------------------------------------------------------------
```

<p>

What could be done to solve this problem? We need a programming language which is easy to write, read, debug and correct mistakes. 

We need a programming language which is a level **higher** than the machine language. A programming language could be designed in such a way that when it is encoded, it produces 0s and 1s. Assume there are 4 operations the computer supports: Addition, Subtraction, Multiplication and Copying data from one place to another. The following is the encoding scheme.

1. 00 - ```add```
2. 01 - ```sub```
3. 10 - ```mul```
4. 11 - ```copy```

It is common to copy constants(or immediate values) into memory locations. That is why, let us consider one more type of copy operation. Let us call it ```copyc``` which stands for copy-constant. With this operation added, we need 3 bits to encode all the operations.

1. 000 - ```add```
2. 001 - ```sub```
3. 010 - ```mul```
4. 011 - ```copy```
5. 100 - ```copyc```

Similarly, to write a full fledged programs with loops, functions etc., we would need a lot more operations like comparison, branching(jumping from one place to another) etc., Assume we can encode all the operations with 8 bits(ie., a maximum of 256 operations).

These operations need operands to work on. Assume length of each address is 16 bits and length of each data element is also 16 bits. This way, we would need(8+16+16)40 bits(or 5 bytes) to encode each operation. Let the result of the operation be stored in the first operand. Our operations now look like the following.

1. ```add Address1, Address2```: Contents at Address1 and Address2 are added and stored back at Address1.
2. ```sub Address1, Address2```: Difference between contents at Address1 and Address2 is computed and stored back into Address1.
3. ```mul Address1, Address2```: Product of contents at Address1 and Address2 is computed and stored at Address1.
4. ```copy Address1, Address2```: Contents at Address2 is copied to memory at Address1.
5. ```copyc Address1, constant```: Copies a 2-byte constant into memory at Address1.

Note that contents mean 2 bytes at a specified address.

```add 0x1000, 0x1004``` will be encoded as ```00000000-0001000000000000-0001000000000100```. This operation adds 2 bytes present at address 0x1000 and 2 bytes present at address 0x1004 and stores the sum at 0x1000.

This high-level language given to the programmers is called **Assembly Language**.

We are at a stage where we can write a program using assembly language. Let us write a program to swap 2 given numbers - call it *swap.S*.

</p>

```asm
-------------------------------------------------------------------------swap.S
0x1000: 0x0010
        0x0020  ; present at 0x1002
        0x0000  ; present at 0x1004

0x0000: copy 0x1004, 0x1000
        copy 0x1000, 0x1002 ; present at 0x0005
        copy 0x1002, 0x1004 ; present at 0x000a

; At this point, the 2 values at 0x1000 and 0x1002 have been swapped.
-------------------------------------------------------------------------------
```

<p>

This program requests the operating system to store the variables at the address 0x1000 and code at address 0x0000.

All this does not happen on its own. Someone should convert these assembly programs into binary programs right? The computer even now understands only the binary language. Whatever high-level language we *invent* to write programs, they must be translated to machine language(0s and 1s) and then fed to the computer for execution. **We need a program/tool which does this translation**. Writing such a translator paved way to write better, cleaner programs quickly. A program which converts assembly language into machine language is known as an **Assembler**. An assembler **assembles** instructions to generate their the machine equivalent code.

This is great progress.

Currently, variables and code are present together. Separating them would make the program **cleaner** and **more secure**. While writing the program, it can be divided into 2 **sections**. One section can have variable declarations/definitions - call it **data section** and other can have assembly instructions - call it **text section**. The assembler takes out a two-section assembly program as input and gives a binary program as output which also has the 2 sections in binary form.

Now, our two-section *swap.S* looks like this.

</p>

```asm
-------------------------------------------------------------------------swap.S
section data

0x1000: 0x0010
        0x0020  ; present at 0x1002
        0x0000  ; present at 0x1004

section text

0x0000: copy 0x1004, 0x1000
        copy 0x1000, 0x1002 ; present at 0x0005
        copy 0x1002, 0x1004 ; present at 0x000a

; At this point, the 2 values at 0x1000 and 0x1002 have been swapped.
-------------------------------------------------------------------------------
```

<p>

We started with programs being a stream of bits entered into the computer. Now, we have divided the program into sections. The binary program would look like the following - *swap.bin*.

</p>

```
-----------------------------------------------------------------------swap.bin
0001000000000000    ; Address 0x1000
0000000000010000    ; 0x0010
0000000000100000    ; 0x0020
0000000000000000    ; 0x0000

0000000000000000    ; Address 0x0000
00000011 0001000000000100 0001000000000000  ; copy 0x1004, 0x1000
00000011 0001000000000000 0001000000000010  ; copy 0x1000, 0x1002
00000011 0001000000000010 0001000000000100  ; copy 0x1002, 0x1004

; At this point, the 2 values at 0x1000 and 0x1002 have been swapped.
-------------------------------------------------------------------------------
```

<p>

Writing all this in hexadecimal will be a lot more easier to read.

</p>

```
-----------------------------------------------------------------------swap.bin
1000
0010
0020
0000

0000
0310041000
0310001002
0310021004
-------------------------------------------------------------------------------
```

<p>

Consider the imaginary asssembler **myasm**. Run it on the two-section *swap.S* to get *swap.bin*.

</p>

```
-------------------------------------------------------------------------------
$ myasm swap.S -o swap.bin
-------------------------------------------------------------------------------
```

<p>

Now, let us think from the operating system's perspective. It should be able to process the binary program in an **unambiguous** manner. Let us build a few rules which the operating system can use to parse the binary program.

1. The binary program always starts with the data section.
2. The data section and text section is separated by 2 newlines.
3. Every section starts by specifying its starting address in memory.

Assuming every data element is 2 bytes and every instruction is 5 bytes, *swap.bin* can be parsed unambiguously and then executed.

Instead of using newlines and spaces to make processing unambiguous, we can have a header-like structure which has all the necessary **metadata** about the binary program. Let us see what metadata we need to put in the header.

1. Relative to the beginning of the file, where is the data section present - **data_section_offset** (in bytes).
2. Size of the data section - **data_section_size** (in bytes).
3. Relative to the beginning of the file, where is the text section present - **text_section_offset** (in bytes).
4. Size of the text section - **text_section_size** (in bytes).

Along with the above 4, we can specify the starting addresses of data and text sections in the header itself - **data_section_addr**, **text_section_addr**.

Assuming each of these to be 2 bytes in size, we can tell that the first 12 bytes will be the above metadata. Right after this metadata, the actual data and text sections are placed.

</p>

```
----------------------------------------------------------------------------BFF
data_section_offset :   bytes 0-1
data_section_size   :   bytes 2-3
data_section_addr   :   bytes 4-5
text_section_offset :   bytes 6-7
text_section_size   :   bytes 8-9
text_section_addr   :   bytes 10-11

; Either data or text section right after the header
-------------------------------------------------------------------------------
```

<p>

We just designed a simple file format for binary programs. Let us call it **BFF - Binary File Format**. The following is how *swap.bin* in BFF looks like.

</p>

```
----------------------------------------------------------------swap.bin in BFF
000c        ; data section offset = 12 bytes
0006        ; data section size = (2+2+2) = 6 bytes
1000        ; data section starting address = 0x1000
0012        ; text section offset = (12 + 6) = 18 bytes
000f        ; text section size = 15 bytes
0000        ; text section starting address = 0x0000
0010        ; data section starts
0020
0000
0310041000  ; text section starts
0310001002
0310021004
-------------------------------------------------------------------------------
```

<p>

The binary program is now self-describing. The part of the operating system that processes the binary program should be aware of the Binary File Format. Our **assembler** should be modified to generate a binary program in BFF.

Now, there is a definite structure to the generated binary programs, but there are a few problems here.

Handling raw addresses can lead to mistakes. Instead of writing ```add 0x1000, 0x1004```, you write ```add 0x0100, 0x1004```, the program definitely won't function properly, it might actually crash. And when dealing with numbers, these are common mistakes we do. How can this problem be solved? **Variable names** can be used instead of raw addresses. Map variable names to addresses in the beginning of the program, later only use the variable names throughout the program. The **assembler** should be updated with this feature.

Along with that, it would be better if the code can be divided into chunks each performing a particular task - similar to functions. Enter **labels**! Let us rewrite the two-section *swap.S* using variable names and labels.

</p>

```asm
------------------------------------------swap.S with variable names and labels
section data

0x1000:

a: 0x0010       ; 0x1000
b: 0x0020       ; 0x1002
temp: 0x0000    ; 0x1004

section text

0x0000:

_start:             ; 0x0000
    jump swap_a_b   ; 0x0000

swap_ret:           ; 0x0005
    add a, b        ; 0x0005
    exit            ; 0x000a

swap_a_b:           ; 0x000f
    copy temp, a    ; 0x000f
    copy a, b       ; 0x0014
    copy b, temp    ; 0x0019
    jump swap_ret   ; 0x001e
-------------------------------------------------------------------------------
```

<p>

Let us write the BFF equivalent of the above *swap.S*. Assume ```jump``` is the 6th instruction, it is encoded as ```00000101``` and ```exit``` as the 256th instruction and is encoded as ```11111111```.

</p>

```
-------------------------------------------------------------------------------
$ myasm swap.S -o swap.bin
-------------------------------------------------------------------------------
```

<p>

The below *swap.bin* is generated in the following manner by our imaginary assembler.

</p>

```
-----------------------------------------------------swap.bin written using BFF
000c        ; data section offset = 12 bytes
0006        ; data section size = 6 bytes
1000        ; data section starting address = 0x1000
0012        ; text section offset = 18 bytes
0023        ; text section size = 35 bytes
0000        ; text section starting address = 0x0000
0010        ; data section starts
0020
0000
050000000f  ; text section starts. swap_a_b = 0x0000000f
0010001002  ; add a, b
ff00000000  ; exit(0)
0310041000
0310001002
0310021004
0500000005  ; jump swap_ret
-------------------------------------------------------------------------------
```

<p>

Labels and variable names are the facilities given to the programmer. It is important to note that variable names and labels are simply **aliases** to addresses. The assembly program has used labels and variable names but the operating system does not care about names or labels. All it cares about are addresses. The assembler needs to be updated to replace all the labels and names with the corresponding memory addresses when converting an assembly program into binary.

Look at the convenience the programmer gets from variable names and labels. Consider the very first instruction - ```jump swap_a_b```. Without the label, he should have written ```jump 0x000f```. What if there are a lot of jumps? Or there are many instructions between ```_start``` and ```swap_a_b``` and the programmer made a mistake in calculating the address. All this grunt work is avoided and is loaded onto the assembler.

Note that we wrote *swap.bin* easily because there were hardly any instructions. Address of every label, variable name was easy to compute. But as discussed above, what happens when it is a long program? Computing the addresses for variable names and labels manually is a pain. This means the assembler needs to be updated to compute addresses and replace them when converting instructions to binary.

**How** should the assembler be modified in order to translate such an assembly program with sections, variable names and labels into binary program?

Let us see how it can be done.

Consider the instruction ```jump swap_a_b```. Without the label, it should actually be ```jump 0x000f```. Finally, it needs to be encoded to ```00000101-00000000000000000000000000001111```. The ```jump``` instruction is encoded as ```00000101```. 0x000f is encoded as a 32-bit number. Whenever the assembler encounters ```swap_a_b```, it needs to be replaced with its address 0x000f. The exact thing needs to be done for all the other variables and labels too. Whenever **a** is encountered, it needs to be replaced with its address 0x1000. The variable names and labels are collectively called **Symbols**. To make this mechanism work, we can construct a simple table with Symbol as Key and its address as Value. Such a table is known as **Symbol Table**. The assembler takes the asssembly program as input and it processes it once top to bottom and constructs the table. The following is the symbol table.

| Symbol     |  Address   |
|------------|-----------:|
|     a      |  0x1000    |
|     b      |  0x1002    |
|   temp     |  0x1004    |
|   _start   |  0x0000    |
| swap_ret   |  0x0005    |
| swap_a_b   |  0x000f    |

With if the variable name and a label are same? To resolve this, the symbol type can be stored.

| Symbol     |    Type    |  Address   |
|------------|:----------:|-----------:|
|     a      | variable   |  0x1000    |
|     b      | variable   |  0x1002    |
|   temp     | variable   |  0x1004    |
|   _start   | label      |  0x0000    |
| swap_ret   | label      |  0x0005    |
| swap_a_b   | label      |  0x000f    |

By processing the assembly program once(in **one pass**), the symbol table is constructed. The binary program is generated in the next pass. Because our assembler processes the assembly program twice(**2 passes**), this is a two-pass assembler.

The ease of programming has increased a lot at the cost of a complex assembler.

This is a lot of progress compared to binary language which was being used in the beginning.

We have come a long way, but there are still issues. Our assembler forces the programmer to write the complete program in a single file. But splitting the code into various files based on their functionality(or any other criteria) is helpful. Let us split our swapping program into 2 parts: *main.S* and *swap.S*.

</p>

```asm
-------------------------------------------------------------------------main.S
section data

0x1000:

a: 0x0010       ; 0x1000
b: 0x0020       ; 0x1002

section text

0x0000:

_start:             ; 0x0000
    jump swap_a_b   ; 0x0000

swap_ret:           ; 0x0005
    add a, b        ; 0x0005
    exit            ; 0x000a
-------------------------------------------------------------------------------

-------------------------------------------------------------------------swap.S
section data

0x2000:

temp: 0x0000        ; 0x2000

section text

0x3000:

swap_a_b:           ; 0x3000
    copy temp, a    ; 0x3000
    copy a, b       ; 0x3005
    copy b, temp    ; 0x300a
    jump swap_ret   ; 0x300f
-------------------------------------------------------------------------------
```

<p>

Slowly go through the above code. I want you to be convinced that splitting the sourcecode actually helps. *swap.S* can behave like a module. Any program which needs to swap 2 numbers - has to name those 2 variables **a** and **b** and they can readily use code present in *swap.S*.

Let us try to translate *main.S* using our assembler.

</p>

```
-------------------------------------------------------------------------------
$ myasm main.S -o main.bin
-------------------------------------------------------------------------------
```

<p>

What do you think? Will this successfully give *main.bin*?

Let us walk through the procedure. The assembler parses *main.S* once and constructs a symbol table.

| Symbol     |    Type    |  Address   |
|------------|:----------:|-----------:|
|     a      | variable   |  0x1000    |
|     b      | variable   |  0x1002    |
|   _start   | label      |  0x0000    |
| swap_ret   | label      |  0x0005    |

In the second pass, every statement in *main.S* is translated into its binary equivalent. There are no issues with the data section. There are 2 symbols(a and b) are both are replaced by their address with the help of the symbol table. Then comes the instruction ```jump swap_a_b```. As usual, the assembler searches for the symbol **swap_a_b** in the symbol table, but no luck. It is not present. It means ```jump swap_a_b``` cannot be assembled to its binary equivalent. The assembler gives an error and exits. The assembler was **unable to resolve a symbol into its address**.

Similar thing happens when you try assembling *swap.S*. A few symbols(a, b and swap_ret) won't get resolved. The reason is simple. Those symbols are simply not defined in the respective files. Our assembler currently has no mechanism to support our multiple files idea.

How can we solve this problem? I want you to think of ways to solve this.

We know that the missing symbols in *main.S* are found in *swap.S* and symbols missing in *swap.S* are found in *main.S*. We need to inform this information to the assembler. We need to design a mechanism for the assembler to resolve all the symbol-dependencies in both(or more) files.

Say we inform the assembler in this manner.

</p>

```
-------------------------------------------------------------------------------
$ myasm main.S -o main.bin -depend swap.S
-------------------------------------------------------------------------------
```

<p>

The assembler can parse both *main.S* and *swap.S* once and construct symbol tables for both. In *main.S*'s second pass, the assembler will find that there is no symbol table entry for **swap_a_b**. Instead of raising an error, we have another option. We search for **swap_a_b** in *swap.S*'s symbol table. If it is found, then go ahead and generate binary code for ```jump swap_a_b```. If not found, then it needs to raise an error because there is no where else to search. The same should be done with the symbols not found in *swap.S*. When it encounters **a**, it won't find it in *swap.S*'s table. It needs to search for it in *main.S*'s table. Because there is an entry for **a** in *main.S*'s table, the assembler should be able to resolve it. This is one solution.

When you look at this, it seems like a working solution. But if you dig deeper, it has a few problems.

1. Say there are 100 files. We need to specify the starting addresses of data and text sections present in each of the 100 files. We specified 0x1000 to be starting address of *main.S*'s data section, 0x2000 to be the starting address of *swap.S*'s data section. What if the number of variables in *main.S* exceed 4096 bytes(0x2000-0x1000)? When there are 100 files, taking care of all this is very difficult. Programmer giving out addresses to each section might cause problems.

2. When there are 100 files, there will be 100 data and 100 text sections. Each of these sections have a different starting address. How should these 100+100 sections be organized? Should we change BFF to acoomodate it? Or should we combine all 100 data sections to give a single data section? If yes, how exactly do we do it?

These are problems we need to solve.

So far, the programmer was given the liberty to specify the addresses. It worked well in simple scenarios. But the scenario present above puts unnecessary burden on the programmer, which is common in case of large projects. Will the programmer have an advantage if he specifies the addresses for each data and text section? Not really. All that matters to him is that memory is allocated for these data and text sections. It is best to offload this job to another program or the operating system. Operating system's resources are precious and it needs to be spent carefully. When the operating system is trying to resolve symbols and could not find even a single symbol, it throws an error and all the work done by the operating system is a waste. It is best to write a program to get the job done instead of the operating system doing it All this new program needs to do is to automate the process of giving addresses to all variables and instructions.

The assembler can be extended to do this. But I think it is a stretch. The assembler is responsible for correct translation of assembly code to binary. Resolving unresolved symbols, fixing addresses, stitching all the sections of all sourcefiles to make the binary program is beyond its definition. Let us create another program which does exactly the above listed functions and finally generate the binary program.

How is a symbol resolved? It is used in one sourcefile but is defined in some other. Resolving is nothing but **linking** its usage to its definition. For this reason, let us call this program **linker**. The linker should do all the above listed functions and generate the binary program.

Let us list a few observations which will help us write the linker.

1. The programmer does not specify the addresses of variables, labels and instructions. The linker needs to give addresses.

2. When there are multiple sourcefiles, one of them should contain the **entry-point** to the program ie., the first instruction which should get executed by the operating system.

3. The symbol tables constructed for each sourcefile by the assembler will be helpful in resolving the symbols.

As of now, the assembler constructs a symbol table for each of the sourcefiles. Even if the asssembler has translated 1000s of instructions of a sourcefile into its binary equivalent, when it encounters an undefined symbol, it gives up. It throws an error and all of the work done so far is wasted. It should be noted that only the instructions with unresolved symbols needs to be worked on. Rest of the instructions can be translated to binary without any hustle. That is work gone waste. We want our assembler to do the best(translate all possible variables and instructions) and inform us about all the work it failed to do.

Let us change the assembler to our advantage. The asssembler takes up a file for translation. In the first pass, it generates symbol table of all the symbols defined in the sourcefile. In the second pass, it translates variable by variable, instruction by instruction. If it encounters a symbol not defined in the sourcefile(aka not present in the symbol table), create an empty symbol table entry for this undefined symbol and continue translation. At the end of it, we have a binary equivalent data and text section, along with a symbol table. Along with storing the binary equivalent of data and text sections in the binary file, store the symbol table too. This way, the assembler translated all the possible instructions to binary. Rest of the instructions which had unresolved symbols, information about it is stored in the symbol table.

In the above described process, a few things were abstract. If an undefined symbol is encountered(say in ```jump swap_a_b```), create an entry for this undefined symbol(**swap_a_b**) and **continue translation**. We did not specify what **continue** exactly means? Ideally, ```jump swap_a_b``` would be translated into 5-byte binary, where the first byte specifies that the instruction is ```jump``` and the next 4 bytes - the address of **swap_a_b**. But now, the asssembler does not know **swap_a_b**'s address. What should the 5-byte binary contain then? The first byte is **0x05**, ```jump```'s binary equivalent, but what about rest of the 4 bytes? That information is found only in the next stage when the linker gives addresses.

Consider our standard *main.S* and *swap.S* example.

</p>

```
-------------------------------------------------------------------------main.S
section data

a: 0x0010
b: 0x0020

section text

_start: 
    jump swap_a_b

swap_ret:
    add a, b
    exit
-------------------------------------------------------------------------------

-------------------------------------------------------------------------swap.S
section data

temp: 0x0000

section text

swap_a_b:
    copy temp, a
    copy a, b 
    copy b, temp
    jump swap_ret
-------------------------------------------------------------------------------
```

<p>

The assembler tries to translate *main.S* into a file which should contain 3 parts: data section, text section and symbol table.

1. This is how the data section looks like.

</p>

``` 
-------------------------------------------------------------------------------
0x0000:     ; data section
0010        ; a
0020        ; b
-------------------------------------------------------------------------------
```    

<p>

The 0x0000 signifies the fact that programmer has not given any address. Relative to the beginning of data section, **a** is at an offset of 0x0000, **b** is at an offset of 0x0002. Unlike before, we do not know the absolute addresses of these variables. We only know their addresses(actually offsets) relative to the corresponding section beginning.

2. This is how text section would look like.

</p>

```
-------------------------------------------------------------------------------
0x0000:
05<??>      ; Don't know where swap_a_b is.
00<??><??>  
ff00000000  ; exit(0)
-------------------------------------------------------------------------------
```

<p>

The assembler encounters ```jump swap_a_b```. It sees that **swap_a_b** is not found in the symbol table. It creates an empty symbol table entry for **swap_a_b**. If the assembler knew **swap_a_b**'s absolute address(assume it is 0x9034), it would generate the binary code 0500009034. But now, the assembler does not know its address. What needs to be done?

The assembler should inform the **linker**(next stage) about this.

What we can do is, we can simply put ```0500000000``` - basically **0x00**s instead of the actual address and inform that 4 bytes at an offset of 0x0001 in the text section should be replaced by **swap_a_b**'s absolute address once found. Once the linker finds **swap_a_b** and gives it an address, ```0500000000``` is changed to ```0500009034```.

Let us take up the next instruction ```add a, b```. All the assembler can do is to translate it as ```00-0000-0002```. It can put the relative addresses of **a** and **b**(with respect to the data section). Even this needs fixing. Ideally, absolute addresses of **a** and **b** should be present here. Even in this case, the assembler needs to inform the linker about this. Suppose the linker gives the data section the address 0x6000, then **a**'s address would be 0x6000+0x0000=0x6000(data section's address + offset at which **a** is present). **b**'s address would be 0x6002. Linker changes ```0000000002``` to ```0060006002```.

You can see that there are 2 cases here. In the first case, linker had to find the label, give an address to it and then change the binary code. In the second case, the section had to be given an address which implicitly gives addresses to all the variables in it, then change the binary code of instructions which have used these variables. We discussed that the assembler needs to inform about these 2 cases to the linker. How can that be done? Think about how it can be solved.

3. Symbol table would look like this.

|  Symbol   |    Type     |  Address   |
|-----------|-------------|------------|
|    a      |  variable   |  0x0000    |
|    b      |  variable   |  0x0002    |
|  _start   |  label      |  0x0000    |
| swap_ret  |  label      |  0x0005    |
| swap_a_b  |  undefined  |  ------    |

The addresses in this symbol table are not the actual addresses. They are relative addresses of variables/labels with respect to the sections they belong to.

Coming back to the question at end of (2), how can the assembler inform the linker about what bytes need to be replaced by what? One solution is to create another table for this. Let us call it **fixing table** because it has information about what all bytes should be replaced by what.

1. In the first case, bytes 1-4 in the text section which are currently zeros should be replaced by the address of **swap_a_b**.

2. In the second case, bytes 0x0006 - 0x0007(2 bytes) in the text section whhich currently represent the relative address of **a** should be replaced by the absolute address of **a** once the linker has it. There is one more way to solve this case.
    * Instead of putting the relative addresses in the binary code and later replacing it with the actual address, it can have zeros and finally the linker can replace it with the actual address by taking the symbol table’s help. Consider the variable b. In the add a, b instruction, currently it is translated to 00-0000-0002 – relative addresses of variables are put here. Later, when the data section is given an address(say 0x5000), 0000 is replaced by (0x5000+0x0000) and 0002 is replaced by (0x5000+0x0002). Either this can be done or the assembler can simply translate add a, b into 00-0000-0000 – basically zeros in places which need replacement. Later linker comes in and checks the fixing table. It sees that the first 0000 needs to be replaced by a’s address. With symbol table's help, the linker updates it with absolute addresses.

The first case was an **unresolved label issue**, the second is an **unresolved variable name issue**. These are the only 2 cases we have and we have a solution for it. Let us design the fixing table. It needs to have the following details.

1. Offset at which bytes must be replaced. Observe that the offset is always with respect to the text section. Only the text section needs fixing because the programmer would have used undefined symbols there.

2. At the specified offset, how many bytes need to be replaced. In the first case, 4 bytes had to be replaced. In the second, for each variable, 2 bytes had to be replaced.

3. We have specified the location of replacement and how much to replace. We need to specify what it needs to be replaced **with**. In the first case, the 4 bytes should be replaced with **swap_a_b**'s address. In the second case, 2 bytes need to be replaced with **a**'s address, other 2 bytes with **b**'s address. Basically, we need to specify the symbol.

The fixing table would look like this for *main.S*.

|  Offset   | Number of bytes |   Symbol   |
|-----------|-----------------|------------|
|    1      |        4        | swap_a_b   |
|    6      |        2        |     a      |
|    8      |        2        |     b      |

With the help of symbol table and fixing table, the linker should be able to fix all the instructions in the text section.

Note that the file output by the assembler is not a binary **program**. A program is something which can be run on the machine. It is a file which has a bunch of sections(data, text, symbol-table). It cannot be run because nothing is ready yet. Multiple symbols might be unresolved, its dependent files still need to be stitched with this file. Because these files will be used by our **linker** to generate the binary program, let us call them **linkable files**.

In the beginning, the assembler’s output was a simple file with just a text and a data section. As we progressed, we designed a file format for the assembler’s output – the Binary File Format which had a simple header. Now, we want the assembler to output a file which has the data section, text section, symbol table and the fixing table. Parts of the text section with unresolved variable names, labels will have zeros instead of their addresses. There will be empty symbol table entries, addresses in symbol table are all relative addresses. We call this file a **linkable file**.

We need to update the Binary File Format to match our linkable file requirements.

It has a data and text section, a symbol table and a fixing table. A section can contain anything. Both the tables can be present in the form of sections. The new BFF header would look like this.
</p>

```
-------------------------------------------------------------------------------
data_section_offset     :   bytes 0-1
data_section_size       :   bytes 2-3
data_section_addr       :   bytes 4-5
text_section_offset     :   bytes 6-7
text_section_size       :   bytes 8-9
text_section_addr       :   bytes 10-11
symtab_section_offset   :   bytes 12-13
symtab_section_size     :   bytes 14-15
fixtab_section_offset   :   bytes 16-17
fixtab_section_size     :   bytes 18-19

; Either data or text section right after the header.
-------------------------------------------------------------------------------
```

<p>
Note that the number of sections are steadily increasing. if we need to support program debugging, we can have one more section with debugging details. Then, we will have to add 2 more members to the BFF header. Whenever a new section needs to be introduced, BFF format itself needs changes. Any file format should be flexible to changes. How can BFF be designed in that way? Think about it.

Every section has some metadata related to it - section offset in the file, section size, starting address. Instead of having metadata of all sections in the BFF header, we have have a separate section-metadata header. Because we have a number of sections, we can have an array of such section-metadata headers. When we process a section header, we should be able to get complete information about that section including what type of section it is - is it data, text, symtab or fixtab. Let us call the section-metadata headers ```bff_sec_meta_t```. From now onwards, let us use C to describe these structures.
</p>

```c
-------------------------------------------------------------------------------
typedef sec_meta
{
    unsigned short int type;
    unsigned short int offset;
    unsigned short int size;
    unsigned short int addr;
} sec_meta_t;
-------------------------------------------------------------------------------
```

<p>
We now have 4 types of sections: text, data, symtab and fixtab. Let us write an enumerator for it.</p>

```c
-------------------------------------------------------------------------------
typedef enum
{
    BFF_SEC_TEXT=0,
    BFF_SEC_DATA,
    BFF_SEC_SYMTAB,
    BFF_SEC_FIXTAB,
} bff_sec_type_t;
-------------------------------------------------------------------------------
```

<p>
Let us discuss the symbol table. In all our previous discussions, we discussed about the table in a conceptual manner. We didn't think how to code a symbol table. Intuitively, it is an array of ```<symbol, address>``` pairs. Later, we added another attribute to it - **symbol-type**. A symbol structure would look something like this.
</p>

```c
-------------------------------------------------------------------------------
typedef struct symbol
{
    char name[256];
    unsigned short int type;
    unsigned short int addr;
} bff_sym_t;
-------------------------------------------------------------------------------
```

<p>
There are 3 types of symbols as of now: variable, label and undefined.
</p>

```c
-------------------------------------------------------------------------------
typedef enum
{
    BFF_SYM_VAR=0,
    BFF_SYM_LABEL,
    BFF_SYM_UNDEFINED,
} bff_sym_type_t;
-------------------------------------------------------------------------------
```

<p>
The symbol table should ideally be a hash table. If you input the symbol-name, you get the index where its structure is stored in the array of symbol structures. Let us discuss about the hash function and other details later.

Let us come to the fixing table. As discussed, each entry in the fixing table has 3 members: offset, number of bytes and symbol.
</p>

```c
-------------------------------------------------------------------------------
typedef struct fixtab_entry
{
    unsigned short int offset;
    unsigned short int nbytes;
    char symbol[256];
} bff_fix_ent_t;
-------------------------------------------------------------------------------
```

<p>
With all these defined, we can decide the BFF header structure. All it needs to have is the number of sections. Everything else is defined in the section-metadata array.
</p>

```c
-------------------------------------------------------------------------------
typedef struct bff_header
{
    unsigned short int bff_sec_n;
} bff_hdr_t;
-------------------------------------------------------------------------------
```

<p>
Now that we have all the fundamental elements, we can define the next version of BFF.
</p>

```
---------------------------------------------------------------generic BFF file
bff_hdr_t                   ; Has number of sections
Array of bff_sec_meta_t     ; Array size depends on number of sections
; section1
; section2
.
.
.
; section bff_hdr_t.bff_sec_n
-------------------------------------------------------------------------------
```

<p>
This way, even if you need more sections later, you can add them without changing BFF. If you need one more section to contain debugging details, this version of BFF can easily handle it.

Out assembler **myasm** requires significant updates. By taking a single assembly sourcefile, it should generate a linkable file in BFF.

To firm up our discussion, let us take our *main.S* and *swap.S* example and generate 2 linkable files *main.link* and *swap.link* by hand. The following are the 2 sourcefiles.
</p>

```
-------------------------------------------------------------------------main.S
section data

a: 0x0010
b: 0x0020

section text

_start: 
    jump swap_a_b

swap_ret:
    add a, b
    exit
-------------------------------------------------------------------------------

-------------------------------------------------------------------------swap.S
section data

temp: 0x0000

section text

swap_a_b:
    copy temp, a
    copy a, b 
    copy b, temp
    jump swap_ret
-------------------------------------------------------------------------------
```

<p>
The assembler should take *main.S* and *swap.S* and generate *main.link* and *swap.link*. Let us go through one file at a time.
</p>

```
-------------------------------------------------------------------------------
$ myasm main.S
-------------------------------------------------------------------------------
```

<p>
1. Data section
</p>
```
-------------------------------------------------------------------------------
0x0000:     ; data section
0010        ; a
0020        ; b
-------------------------------------------------------------------------------
```

<p>
2. Text section
</p>

```
-------------------------------------------------------------------------------
0x0000:         ; text section
0500000000      ; jump swap_a_b
0000000000      ; add a, b
ff00000000      ; exit(0)
-------------------------------------------------------------------------------
```

<p>
Notice the first and second instructions. They both need fixing - assembler does not know the actual addresses of **swap_a_b** and **b**.

3. Symbol Table

|  Symbol   |    Type     |  Address   |
|-----------|-------------|------------|
|    a      |  variable   |  0x0000    |
|    b      |  variable   |  0x0002    |
|  _start   |  label      |  0x0000    |
| swap_ret  |  label      |  0x0005    |
| swap_a_b  |  undefined  |  ------    |

The symbol table has the symbols and their relative addresses. If the symbol is a variable, then its address is relative to the data section. If the symbol is a label, then its address is relative to the text section.

4. Fixing Table

|  Offset   | Number of bytes |   Symbol   |
|-----------|-----------------|------------|
|    1      |        4        | swap_a_b   |
|    6      |        2        |     a      |
|    8      |        2        |     b      |


Let us now talk about *swap.S*.

1. Data section
</p>

```
-------------------------------------------------------------------------------
0x0000:     ; data section
0000        ; temp
-------------------------------------------------------------------------------
```

<p>
2. Text section
</p>

```
-------------------------------------------------------------------------------
0x0000:         ; text section
0300000000      ; copy temp, a
0300000000      ; copy a, b
0300000000      ; copy b, temp
0500000000      ; jump swap_ret
-------------------------------------------------------------------------------
```

<p>
This text section needs a lot of fixing.

3. Symbol Table

|  Symbol  |   Type    |  Address  |
|----------|-----------|-----------|
|  temp    | variable  |  0x0000   |
|swap_a_b  | label     |  0x0000   |
|    a     | undefined |  ------   |
|    b     | undefined |  ------   |
|swap_ret  | undefined |  ------   |

4. Fixing Table

|   Offset   | Number of bytes   |  Symbol   |
|------------|-------------------|-----------|
|   0x1      |         2         |   temp    |
|   0x3      |         2         |     a     |
|   0x6      |         2         |     a     |
|   0x8      |         2         |     b     |
|   0xb(11)  |         2         |     b     |
|   0xd(13)  |         2         |   temp    |
|   0x11(16) |         4         | swap_ret  |

With all this, we have generated *main.link* and *swap.link*.

Now to the interesting part. Let us manually link these 2 files to generate an output program *a.out*. We know that a runnable program has only 2 sections - data and text.

Let us follow a very simple strategy. We have 2 data and 2 text sections. let us call them **main.data**, **swap.data** and **main.text**, **swap.text**.

Let us append all of these sections together. Assume we are dumping them into a file. **main.data** will be at the beginning - its file offset is 0x0000. **swap.data** will come right after **main.data** - this is at a file offset of **main.data.size**. **main.text** will be at an offset of (**main.data.size** + **swap.data.size**). **swap.text** will be present at an offset of (**main.data.size** + **swap.data.size** + **main.text.size**).

1. main.data.size = 4 bytes, main.data.off = 0x0000
2. swap.data.size = 2 bytes, swap.data.off = 0x0000+4 = 0x0004
3. main.text.size = 15 bytes, main.text.off = 0x0004+2 = 0x0006
4. swap.text.size = 20 bytes, swap.text.off = 0x0006+15 = 0x0015

Assume that the linker gives an address **0x1000** to the beginning. Automatically all the sections(and their members) get addresses. **addr** = **0x1000**.

1. main.data.addr = addr + main.data.off = 0x1000
2. swap.data.addr = addr + swap.data.off = 0x1004
3. main.text.addr = addr + main.text.off = 0x1006
4. swap.text.addr = addr + swap.text.off = 0x1015

With these at hand, we can update the known symbols' addresses.Updating the addresses is very simple. We have the addresses of the sections above. Symbol table entries have the symbols' relative addresses. **Absolute-address = (section-address + relative-address)**. The following are the updated symbol tables.

1. *main.link*'s symbol table

|  Symbol   |    Type     |  Address   |
|-----------|-------------|------------|
|    a      |  variable   |  0x0000    |
|    b      |  variable   |  0x0002    |
|  _start   |  label      |  0x0000    |
| swap_ret  |  label      |  0x0005    |
| swap_a_b  |  undefined  |  ------    |

                                              converted to

|  Symbol   |    Type     |  Address   |
|-----------|-------------|------------|
|    a      |  variable   |  0x1000    |
|    b      |  variable   |  0x1002    |
|  _start   |  label      |  0x1006    |
| swap_ret  |  label      |  0x100b    |
| swap_a_b  |  undefined  |  ------    |

2. *swap.link*'s symbol table

|  Symbol  |   Type    |  Address  |
|----------|-----------|-----------|
|  temp    | variable  |  0x0000   |
|swap_a_b  | label     |  0x0000   |
|    a     | undefined |  ------   |
|    b     | undefined |  ------   |
|swap_ret  | undefined |  ------   |

                                              converted to
|  Symbol  |   Type    |  Address  |
|----------|-----------|-----------|
|  temp    | variable  |  0x1004   |
|swap_a_b  | label     |  0x1015   |
|    a     | undefined |  ------   |
|    b     | undefined |  ------   |
|swap_ret  | undefined |  ------   |

Note that variable-type entries should be updated using **data.size**, label-type entries using **text.size**.

By combining 2 data sections, we have a single data section. By combining 2 text sections, we have a text section. Starting address of this new data section is **0x1000**, starting address of text section is **0x1006**. Size of data section is 6 bytes, size of text section is 35 bytes.

This is how it looks like.
</p>

```
-------------------------------------------------------------------------------
0010        ; a, 0x1000, data section
0020        ; b
0000        ; temp
0500000000  ; jump swap_a_b, 0x1006, text section
0000000000  ; add a, b
ff00000000  ; exit(0)
0300000000  ; copy temp, a
0300000000  ; copy a, b
0300000000  ; copy b, temp
0500000000  ; jump swap_ret
-------------------------------------------------------------------------------
```

<p>
This needs work. We need to fix everything to generate the program. Let us use the two fixing tables. Let us take a look at *main.link*'s fixing table.

|  Offset   | Number of bytes |   Symbol   |
|-----------|-----------------|------------|
|    1      |        4        | swap_a_b   |
|    6      |        2        |     a      |
|    8      |        2        |     b      |

At offset 1, 4 bytes should be replaced with address of **swap_a_b**. From *swap.link*'s symbol table, we get address of **swap_a_b** = 0x1015. The offset is relative to **main.text**. In the file above, you go to that by adding the offset **1** with **main.text.off** computed above. We get 0x0006+1 = 0x0007. Go to byte 0x0007 and replace **00000000** with **00001015**. We have the following.
</p>

```
-------------------------------------------------------------------------------
0010        ; a, 0x1000, data section
0020        ; b
0000        ; temp
0500001015  ; <---UPDATED jump swap_a_b, 0x1006, text section
0000000000  ; add a, b
ff00000000  ; exit(0)
0300000000  ; copy temp, a
0300000000  ; copy a, b
0300000000  ; copy b, temp
0500000000  ; jump swap_ret
-------------------------------------------------------------------------------
```

<p>
Along with that, update **swap_a_b**'s symbol table entry in *main.link*. Its type is changed to **undefined** to **label**. Its address is updated to 0x1015.

Let us move onto the next entry in the fixing table. It is at an offset of 6 bytes from **main.text**. This means, in the file above, it is present at the offset **main.text.off** + **6** = 0x0006+6 = 0x000c. We need to replace the 2 bytes with **a**'s address. Go to the symbol table and get **a**'s address. It is 0x1000.
</p>

```
-------------------------------------------------------------------------------
0010        ; a, 0x1000, data section
0020        ; b
0000        ; temp
0500001015  ; jump swap_a_b, 0x1006, text section
0010000000  ; <---UPDATED add a, b
ff00000000  ; exit(0)
0300000000  ; copy temp, a
0300000000  ; copy a, b
0300000000  ; copy b, temp
0500000000  ; jump swap_ret
-------------------------------------------------------------------------------
```

<p>
The next entry in the fixing table. We need to go to byte **main.text.off** + **8** = 0x0006+8 = 0x000e. We need to replace the 2 bytes with **b**'s address. Retrive it from the symbol table. Address if 0x1002.
</p>

```
-------------------------------------------------------------------------------
0010        ; a, 0x1000, data section
0020        ; b
0000        ; temp
0500001015  ; jump swap_a_b, 0x1006, text section
0010001002  ; <---UPDATED add a, b
ff00000000  ; exit(0)
0300000000  ; copy temp, a
0300000000  ; copy a, b
0300000000  ; copy b, temp
0500000000  ; jump swap_ret
-------------------------------------------------------------------------------
```

<p>
With that, all the *main.link*'s fixing table entries are satisfied.

Linker's job is to satisfy entries of all fixing tables. Let us move to *swap.link*'s fixing table.

|   Offset   | Number of bytes   |  Symbol   |
|------------|-------------------|-----------|
|   0x1      |         2         |   temp    |
|   0x3      |         2         |     a     |
|   0x6      |         2         |     a     |
|   0x8      |         2         |     b     |
|   0xb(11)  |         2         |     b     |
|   0xd(13)  |         2         |   temp    |
|   0x11(16) |         4         | swap_ret  |

These offsets are relative to **swap.text**. Let us start with the first entry. It is at an offset of **0x1**. In the file above, we need to go to bytes **swap.text.off** + **0x1** = 0x0015+1=0x0015. Let us go to byte 0x0016(22). The 2 bytes here needs to be replaced with **temp**'s address which is **0x1004** - retrieved from *swap.link*'s symbol table. After replacement,
</p>

```
-------------------------------------------------------------------------------
0010        ; a, 0x1000, data section
0020        ; b
0000        ; temp
0500001015  ; jump swap_a_b, 0x1006, text section
0010001002  ; add a, b
ff00000000  ; exit(0)
0310040000  ; <---UPDATED copy temp, a
0300000000  ; copy a, b
0300000000  ; copy b, temp
0500000000  ; jump swap_ret
-------------------------------------------------------------------------------
```

<p>
I hope you get an idea of what is being done. Simply continuing this process, we get the following.
</p>

```
-------------------------------------------------------------------------------
0010        ; a, 0x1000, data section
0020        ; b
0000        ; temp
0500001015  ; jump swap_a_b, 0x1006, text section
0010001002  ; add a, b
ff00000000  ; exit(0)
0310041000  ; <---UPDATED copy temp, a
0310001002  ; copy a, b
0310021004  ; copy b, temp
050000100b  ; jump swap_ret
-------------------------------------------------------------------------------
```

<p>
We are almost there. Everything is resolved, all the fixing tables' entries are satisfied. There is just one thing which needs to be sorted. Which is the first instruction run by the operating system? What is the entry-point to this program?

We know that the symbol ```_start``` which we used in *main.S* is the starting point. Its address is **0x1006**. It did not matter to the assembler and linker. But it matters to the operating system. The operating system will transferr control to the entry-point instruction and leave. We need to specify the **entry-point address**.

This can be done by the linker. The linker can search for ```_start``` symbol. The address aliased by the ```_start``` symbol is the entry-point address. Even when there are hundreds of sourcefiles, the linker will always search for ```_start``` to get the entry-point address. Without ```_start```, the linker should throw an error.

With that, we have successfully stitched 2 linkable files into one single runnable program. This is what the linker needs to do. Let us summarize in short what the linker needs to do.

When multiple sourcefiles are given to the assembler, it generates multiple linkable files, one linkable file per sourcefile. Each linkable file has its own data, text, symtab and fixtab sections. The data section needs no fixing. But the text sections need fixing, specific bytes need replacement – everything that needs fixing is recorded by the assembler in the fixing table. The linker first absolute addresses to all the data and text sections. With that being done, update the symbol table entries’ addresses. Once they are updated, start processing all the fixing tables’ entries. With the help of symbol tables, ideally all the fixing tables’ entries can be satisfied. Even if one fixing table entry is not satisfied, then a binary program cannot be generated, and we end up with a linking error. The linker searches for a symbol(here _start) which signifies the entry-point of the program.

The linker’s output can be run on the Operating System. Such a runnable program is called an executable.

You can see that the BFF can be used for the executable too. An executable has 2 sections - data, text and nothing more. There is just one addition needed in BFF. it needs to have the entry-point address. It can be added to the BFF header. The following is the executable in the new BFF.
</p>

```
-------------------------------------------------------------------------------
0002        ; bff_hdr_t.bdd_sec_n = number of sections = 2
1006        ; bff_hdr_t.bff_entry_addr = 0x1006
0001        ; > bff_sec_meta_t.type = BFF_SEC_DATA
0014        ; data.offset = 20 bytes
0006        ; data.size = 6 bytes
1000        ; data.addr = 0x1000
0000        ; > bff_sec_meta_t.type = BFF_SEC_TEXT
001a        ; text.offset = 26 bytes
0023        ; text.size = 35 bytes
1006        ; text.addr = 0x1006
0010        ; data section starts
0020        
0000        
0500001015  ; text section starts
0010001002  
ff00000000  
0310041000  
0310001002  
0310021004
050000100b
-------------------------------------------------------------------------------
```

<p>
Compare this with the first BFF format. There is almost no difference at all. But the way it was generated was entirely different. As new features came in, complexity of converting an assembly program into an executable increased. Finally, we split the conversion process into 2 steps: Assembling and Linking. The following summarizes the progress so far.
</p>

```
---------------------------------------------------------------------------------------------------
|--------| Assembler  |-----------|
| src1.S |----------->| src1.link |--------->-----------|
|--------|            |-----------|                     |
                                                        |
|--------| Assembler  |-----------|                     |
| src2.S |----------->| src2.link |------->--------|    |
|--------|            |-----------|                |    |
                                                   |    |
     .                                             |    |
     .                                             |    |-->|------|         |-------------------|
     .                                             |------->|Linker|-------->| a.out(Executable) |
     .                                                  |-->|------|         |-------------------|
     .                                                  |
     .                                                  |
     .                                                  |
     .                                                  |
                                                        |
|--------| Assembler  |-----------|                     |
|src100.S|----------->|src100.link|---------------------|
|--------|            |-----------|
---------------------------------------------------------------------------------------------------
```

<p>
This is good progress. The programmer can write code the way he wants – variables, functions, multiple files. He does not have to take care of anything Assembler and Linker will take care of everything.

All programs in general have a few common properties. They will need file manipulation(creating new files, writing to files, reading from files, deleting old files etc.,), the may need standard input/output functions to take in values from the user, to output results to the user. They may also have lot of string related operations etc., I write my own file manipulation code. You write your own file manipulation code. All the programmers write their own file manipulation code. This is a waste of time and effort right? It would be so good if one programmer writes a **library** and shares it with everyone else. A **library** is basically a bunch of functions which are present to facilitate certain functionality. A threading library is present to facilitate multi-threaded programs. Standard I/O library facilitates taking input from users and displaying results to the user.

Let us take the example of file manipulation. We need to write a library. Let us decide on the function names.

1. ```file_create()```: Creates a new file
2. ```file_open()```: Opens up a specified file
3. ```file_read()```: Reads from a specified file
4. ```file_write()```: Writes to a specified file
5. ```file_close()```: Closes a specified file

Say we create 6 sourcefiles. 5 files with implementations of each of the above functions and one sourcefile with all the data structures, variables needed. Let us call them **file_create.S**, **file_open.S**, **file_read.S**, **file_write.S**, **file_close.S** and **file_core.S**.

Share the above 6 files with everyone. Programmers can assemble and link them with their projects.

If the linkable files are shared, they can be linked with the linkable files of our project to generate an executable.

This library has 6 files. Say the generated executable is 800 bytes. Consider a huge library with 100 files. This generates an executable of 11000 bytes. When this is run, 11000 bytes of main memory is allocated for this executable to run. Consider another program you wrote, even that uses the same library. Even its executable is about 10000 bytes. When it is run, a 10000 bytes is allocated. Essentially, there are 2 copies of the library in main memory at this point. This is sheer waste. How can this problem be solved?

Common sense tells us that only one copy of the library code and variables should be present in main memory. The programs we write should **share** it. 

If we use the library’s linkable files while linking our project, we would get one fact executable. We want our executable to remain small and we want the library to be separate. Assuming the library code and data is in main memory, when our executable is run, it should be able to access the library’s code without hustle. We need to devise a mechanism for it. Look at this. We want our executable to communicate during runtime ie., when our executable calls a function in the library, that function should be called. Whatever we have done so far is everything before running the program. Assembling, Linking labels, functions are all before the program is run. But now, there are functions, labels, variables in some library which our executable should be able to call. How can we make this work?

Let us start with what we know. Let us consider our file manipulation library. Think what all a library file has. As usual, it should a text section and a data section. Unlike an executable, it does **not** have an entry-point. Instead, it has 5 labels(file_create, open, read, write, close) which needs to be **exposed** to other executables. How do you inform the world about a bunch of labels and thier addresses you have? Through a symbol table! Along with data section and text section, the library can have a symtab section with details of these 5 labels – their names, types and addresses. This means, a library can be generated in BFF format.

The below is how the file library would look like.
</p>

```
-------------------------------------------------------------------------------
0003; bff_hdr_t.bff_sec_n, number of sections = 3
0000; bff_hdr_t.bff_entry_addr = 0x0000, no use of entry
0001; > bff_sec_meta_t.type = BFF_SEC_DATA
XXXX; data.offset
XXXX; data.size 
XXXX; data.addr
0000; > bff_sec_meta_t.type = BFF_SEC_TEXT
XXXX; text.offset
XXXX; text.size
XXXX; text.addr
0002; > bff_sec_meta_t.type = BFF_SEC_SYMTAB
XXXX; symtab.offset
XXXX; symtab.size
XXXX; symtab.addr
DDDD; data section starts
DDDD
.
.
DDDD
TTTTTTTTTT; text section starts
TTTTTTTTTT
.
.
TTTTTTTTTT
SSSSSSSSSSSSSSSS; symbol table section starts
SSSSSSSSSSSSSSSS
.
.
SSSSSSSSSSSSSSSS
-------------------------------------------------------------------------------
```

<p>
The symbol table would look like this.

|   Symbol    |    Type    |   Address   |
|-------------|------------|-------------|
| file_create |   label    |     0x4000  |
| file_open   |   label    |     0x4123  |
| file_read   |   label    |     0x4345  |
| file_write  |   label    |     0x4456  |
| file_close  |   label    |     0x4567  |

Out linker should be able to generate a library, with a few modifications.
</p>

```
-------------------------------------------------------------------------------
$ mylinker *.link -o file.lib -lib -expose=file_create,file_write,file_read,file_open,file_close
-------------------------------------------------------------------------------
```

<p>
The linker should be modified to generate a library. If the linker is invoked with the **-lib** argument, it should not search for ```_start```(basically the entry-point). Other than that, everything else is the same. In the end, create a symbol table with the labels/variables specified using the **-expose** argument. This can easily be generated using the updated symbol tables used during linking.

We have our library ready. Now coming to our executable. Say we have used the function **file_create**, **file_write**, **file_close** in our project. These symbols will be present in various symbol tables of linkable files. The linker tries to resolve all these symbols only to find out they are not present in our project. It gives a linking error. 

We know that these functions are actually present in the library. Suppose we have an instruction ```jump file_create```. Because assembler does not find **file_create**, it translates the instruction into ```05-00000000```. In all the previous cases, the linker converted ```05-00000000``` to ```05-XXXXXXXX```, where XXXXXXXX is **file_create**’s address. But now, even linker does not find file_create in any of the other linkable files. These unresolved instructions need **fixing**. What it can do is to make a list of all these linker-unresolved symbols and create a symbol table in the executable. As programmers, we can specify in which library these symbols are present in.

|  Symbol   |   Type   |   Address   |   Library   |
|-----------|----------|-------------|-------------|
|file_create|undefined | 0x0000      | file.lib    |
|file_write |undefined | 0x0000      | file.lib    |
|file_close |undefined | 0x0000      | file.lib    |

This is a special version of the symbol table. It contains the library a symbol is present in.

We discussed that the instructions which use the above symbols need fixing. We would need a fixing table too. A fixing table would have the offset at which the bytes need fixing, the number bytes which need fixing and what it needs to be replaced with – basically the symbols.

|   Offset   |   Number of bytes   |   Symbol   |
|------------|---------------------|------------|
|     X      |          4          |file_create |
|     Y      |          4          |file_write  |
|     Z      |          4          |file_close  |

This means, an executable which uses library functions has 3 sections: data, text, special symbol table and fixing table.

Now let us talk about how these symbols are resolved at runtime.

We run the executable. The Operating System copies all the 4 sections. Simply transferring control to entry-point and running the program won’t help. Because when ```jump file_create```(**05-00000000**) is executed, **00000000** may not contain any code and the program might crash. Before running, all the fixing table entries should be satisfied. Let us do that.

First entry is at an offset of X bytes into the text section. The 4 bytes need to be replaced by **file_create**’s address. The OS looks at the symbol table. **file_create** is present in the library *file.lib*. If this library is not present in main memory, it is first put into main memory. Check *file.lib*’s symbol table and get **file_create**’s address. With that, replace **00000000** with **file_create**’s address. 

Do the same with the other 2 entries. Once all the fixing table entries are satisfied, the program is ready to go. With a little bit of work after linking, and a couple of tables in the executable, we are able to save multiple copies of the same library in the main memory.

What did we do? We performed linking here, but at runtime. This way, programmers can write libraries and reuse the code everywhere. BFF can be modified to fit linkable files, executables and libraries.

We started with programs written in binary language(0s and 1s). Then came assembly language, assembler, symbols, symbol table, fixing table, linkable files, linker, executable, library, runtime linking.

This is one thread of progress. Assembly language and the stuff that came later made programming lot more convenient. We have explored this thread to a certain extent.

Let us explore another very interesting thread.

An **assembly language** is specific to a processor. There are a lot of processors out there - Intel 32-bit, Intel 64-bit, AMD, ARM, Sparc etc., Every processor manufacturer offers an **Instruction Set** to the programmer.

1. Intel i7: x64 Instruction Set
2. AMD Ryzen: x64 Instruction Set
3. Intel Pentium: x86 Instruction Set
4. IBM's PowerPC: PPC, PPC64 Instruction Set
5. Oracle's Sparc: SPARC Instruction Set

It is basically all the instructions and other specifications which can be used to program that particular processor.

Say you want to write a sorting program. You are currently working on an Intel Pentium machine. It is a 32-bit architecture. Intel would have published a Programmer's manual which you can use to write the sorting program. Suppose you want to run a sorting program on an ARM machine. Can you run the program you wrote for the pentium machine on this new ARM machine? By definition, an assembly language is specific to a processor. Because of that, a program written for a Pentium machine cannot be run on an ARM machine. You end up writing a sorting program specifically for the ARM machine. Later, you may have to write the same program for a Sparc machine. Do you observe what is happening? The same program is written again and again with different assembly languages. This type of code is known as **non-portable code**. Code intended for one machine cannot be run on another. This is some time and effort which can be saved if there is a way to run a program you write on all the machines. You write it once and run on all machines. How can this problem be solved?

Back then, operating systems, assemblers, linkers, system utilities everything was written in assembly. Assembly language is way more structured than binary language. But think about implementing complex data structures like trees, graphs, lists in assembly. A doubly linked list's node has 3 members: data(say an integer) and 2 references(to next and previous nodes). At assembly level, a node's members would look like this.
</p>

```
-------------------------------------------------------------------------------
data: 0x0000    
prev: 0x0000    
next: 0x0000
-------------------------------------------------------------------------------
```

<p>
If we have an array of structures, it would look like this.
</p>

```
-------------------------------------------------------------------------------
data1: 0x0000    
prev1: 0x0000    
next1: 0x0000
data2: 0x0000
prev2: 0x0000
next2: 0x0000
.
.
.
dataN: 0x0000
prevN: 0x0000
nextN: 0x0000
-------------------------------------------------------------------------------
```

<p>
They are all separate variables, it would help if it is more structured. Having high level data structures like arrays, structs would help.
</p>
```
-------------------------------------------------------------------------------
node
{
    data;
    prev_ref;
    next_ref;
};
-------------------------------------------------------------------------------
```

<p>
There is a need for a higher level language. It can have helpful, often used programming constructs and data structures. Can that language be portable? Can a program written in this language be run on various architectures?

Enter **Fortran**, **COBOL**, **ALGOL**, **C**. The C programming language was invented in 1969 by the god Dennis Ritchie. The UNIX operating system was completely rewritten in C by Dennis Ritchie and Brian Kernighan. It became a very popular language.

How can code written in one of these high-level languages be run on all the architectures? Think about it.

The following diagram explains the problem.
</p>

```
-------------------------------------------------------------------------------
                                |-----------|
                                |Assembly   |
                                |Language1  |
                                |-----------|

                                |-----------|
                                |Assembly   |
                                |Language2  |
                                |-----------|
                                      .
                                      .
|------------|                        .
|   C Code   |                        .
|------------|                        .
                                      .
                                      .
                                      .
                                      .
                                |-----------|
                                |Assembly   |
                                |Language100|
                                |-----------|
-------------------------------------------------------------------------------
```

<p>
From the diagram, we can quickly figure out that code written in high-level languages like C can be run on any architecture if **they can be translated into all these assembly languages**. Once there is assembly code, then we know how it is run on the machine.

The way we designed a translator to translate assembly code to machine code, we now need a translator to translate C code to these assembly languages. Such translators are called **compilers**. High-level code is **compiled** into a specified assembly language.

Let us take a an example of a simple ```for``` loop.
</p>

```c
-------------------------------------------------------------------------------
for(i = 0; i < 100; i++)
{
    ; Body of the loop
}
-------------------------------------------------------------------------------
```

<p>
Try writing its assembly equivalent.
</p>

```
-------------------------------------------------------------------------------
    copyc i, 0  ; i = 0
loop:
    cmp i, 100      ; Compare i with 100
    jifeq loop_out  ; if i == 100, jump to loop_out
    ; Body
    ; of
    ; the
    ; loop
    inc i           ; Increment i
    jump loop       ; Simply go back to the label loop

loop_out:
    ; Rest of the program
-------------------------------------------------------------------------------
```

<p>
To design portable languages, we need a new program - the **compiler**.

With that, let us summarize this section.
</p>

```
----------------------------------------------------------------------------------------------------------
|--------| Compiler |--------| Assembler  |-----------|
| src1.C |--------->| src1.S |----------->| src1.link |-->-----------|
|--------|          |--------|            |-----------|              |
                                                                     |
|--------| Compiler |--------| Assembler  |-----------|              |
| src2.C |--------->| src2.S |----------->| src2.link |---->----|    |  Libraries
|--------|          |--------|            |-----------|         |    |      |
                                                                |    |      |
    .                    .                                      |    |      |
    .                    .                                      |    |-->|------|         |------------|
    .                    .                                      |------->|Linker|-------->| Executable |
    .                    .                                           |-->|------|         |------------|
    .                    .                                           |
    .                    .                                           |
    .                    .                                           |
    .                    .                                           |
                                                                     |
|--------| Compiler |--------| Assembler  |-----------|              |
|src100.C|--------->|src100.S|----------->|src100.link|---->---------|
|--------|          |--------|            |-----------|
----------------------------------------------------------------------------------------------------------
```

## 1.2 Overview { #Overview1.2 }

<p>
In the previous section, we started our exploration with the binary language and then saw high-level languages which can be run on any architecture present. We designed our own toy assembly language, toy binary file format to understand how translation happens.

In this section, we will skim through the process of converting C programs to executables and libraries. We will also see what ELF is in short.

Consider a simple C program *code1.c*.
</p>

```c
------------------------------------------------------------------------code1.c
#include <stdio.h>
#include <stdio.h>

#define NUMBER 100

int a = 10;     // Global variable 'a' initialized to 10
int b;          // Global uninitialized variable

int main()
{
        int c = 123, number = NUMBER;
        char d = 'x';

        printf("Hello World!\n");

        return 0;
}
-------------------------------------------------------------------------------
```

<p>
We will use **gcc**(GNU C Compiler) for our exploration. Compile our C program and see what we get.
</p>

```
-------------------------------------------------------------------------------
chapter1$ gcc-4.8 code1.c -o code1
chapter1$ ls -l
total 16
-rwxr-xr-x 1 dell dell 8440 Mar 22 20:04 code1
-rw-r--r-- 1 dell dell  295 Mar 22 20:04 code1.c
-------------------------------------------------------------------------------
```

<p>
I used gcc version 4.8 to generate the executable.

```file``` is a command which gives a description about a specified file. Let us use it and check out the executable.
</p>

```
-------------------------------------------------------------------------------
chapter1$ file code1
code1: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked,
interpreter /lib64/l, for GNU/Linux 3.2.0, 
BuildID[sha1]=db06fa5d56ff35837b7c96ca06294c76261e7468, not stripped
-------------------------------------------------------------------------------
```

<p>
This gives good amount of information about the generated executable. 

1. It is an ELF file.
2. It targets x86-64 architecture. This implies it targets a 64-bit processor.

In the previous section, we came up with a binary file format(**BFF**) for linkable files, executables, libraries. In *NIX like systems(eg. Linux), a binary file format called **Executable and Linkable Format(ELF)** is used. You can see from its name why it is called so. ELF is used in a variety of operating systems running on a variety of processors.

The ```file``` command output has a lot of other details. It says it is **dynamically linked**, it talks about some **interpreter**, BuildID. It also says it is **not stripped**.

In case you used later versions of gcc(like 6, 7, 8...), your ```file``` output would look like this.</p>

```
-------------------------------------------------------------------------------
chapter1$ file code1
code1: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked,
interpreter /lib64/l, for GNU/Linux 3.2.0,
BuildID[sha1]=d2a757dca057ec6d4405f2dc15d130274b3acc2e, not stripped
-------------------------------------------------------------------------------
```

<p>
This describes *code1* as a **shared object**, not an executable. Understanding the difference needs some background that will be discussed in later chapters.

Open up the executable with a text editor. See if you can understand anything. It is mostly binary, but has a few character-strings.

Conversion of a C sourcefile to an executable is a **multi-step** process.
</p>

```
------------------------------------------------------Conversion of a C program into executable
                                    Preprocessing
                                --------------------- 
  C source code (hello.c)-----> |   Preprocessor    |-----> hello.i (Intermediate C sourcefile)
                                ---------------------

                                    Compiling
                                ---------------------            
                 hello.i------> |     Compiler      |-----> hello.s (Assembly code)
                                ---------------------

                                    Assembling
                                ---------------------
                 hello.s------> |     Assembler     |-----> hello.o (Object code)
                                ---------------------
                    
                                    Linking
                                ---------------------
     hello.o + Libraries------> |       Linker      |-----> hello / a.out (Executable)     
                                ---------------------   
-----------------------------------------------------------------------------------------------
```

<p>
The conversion process constitutes of 4 sub-processes. They are **Preprocessing**, **Compiling**, **Assembling** and **Linking**.

We saw in the previous section that when a C program is compiled, few files other than the executable(or library) are generated. But **gcc** obviously generates all those files but it stores the end-product(executable/library) on the filesystem. Let us request the compiler to store all the intermediate files between C sourcefile and executable.
</p>

```
-------------------------------------------------------------------------------
chapter1$ gcc code1.c -o code1 --save-temps
chapter1$ ls
code1  code1.c  code1.i  code1.o  code1.s
-------------------------------------------------------------------------------
```

<p>
In the next few sections, each of these sub-processes are explored.
</p>

## 1.3 Preprocessing { #Preprocessing1.3 }

<p>
The **C-Preprocessor**(**CPP**) does the Preprocessing.

CPP takes a normal C sourcefile(*code1.c*) as input and generates an **intermediate** sourcefile(*code1.i*). CPP does a lot of work before sourcefile goes to next sub-process. **gcc** is invoked to generate the executable, it internally invokes the preprocessor, which is a separate program. What exactly does the preprocessor do? Let us take a look at **CPP**'s manual page.
</p>

```
--------------------------------------------------------------cpp's manual page
chapter1$ man cpp
CPP(1)                                GNU                                 CPP(1)

NAME
       cpp - The C Preprocessor
.
.
.
DESCRIPTION
The C preprocessor, often known as cpp, is a macro processor that is used 
automatically by the C compiler to transform your program before compilation.
It is called a macro processor because it allows you to define macros, 
which are brief abbreviations for longer constructs.
.
.
.
-------------------------------------------------------------------------------
```

<p>
The manual page gives all the details about the Preprocessor. It tells that it allows us to define **Macros**. As you read along, you will come to know it helps in doing lot more than that.

Let us use **CPP** on our C sourcefile(*code1.c*) and see what we get.
</p>

```c
------------------------------------------------------------------------code1.i
# 1 "code1.c"
# 1 "<built-in>"
# 1 "<command-line>"
# 31 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
.
.
# 868 "/usr/include/stdio.h" 3 4
# 2 "code1.c" 2

int a = 10;
int b;

int main()
{
        int c = 123, number = 100;
        char d = 'x';

        printf("Hello World!\n");

        return 0;
}
-------------------------------------------------------------------------------
```

<p>
The above code has the first few and last few lines of the complete output. You can check *code1.i* for the complete output. The following are the notable differences between the above and the actual C program present in *code1.c*.

1. A lot of stuff is added - stuff which was never part of the original C program.
2. The 2 ```#include <stdio.h>``` present in *code1.c* have vanished.
3. The macro definition ```#define NUMBER  100``` is not present.
4. The comments next to the global variables have vanished.
5. There is ```number = 100``` instead of ```number = NUMBER```. This means that macro has been replaced with its actual value.

Let us take up one difference at a time.

All the header files are present in **/usr/include** directory. Checkout **stdio.h**. It has a bunch of **header files**, **macros**, **typedefs**, **library functions**, **declarations** etc., as shown below.
</p>

```c
------------------------------------------------------------------------stdio.h
/* The possibilities for the third argument to `fseek'.
   These values should not be changed.  */
#define SEEK_SET        0       /* Seek from beginning of file.  */
#define SEEK_CUR        1       /* Seek from current position.  */
#define SEEK_END        2       /* Seek from end of file.  */
.
.
.
typedef __off_t off_t;
.
typedef __off64_t off_t;
.
.
/* Write formatted output to STREAM.

   This function is a possible cancellation point and therefore not
   marked with __THROW.  */
extern int fprintf (FILE *__restrict __stream,
                    const char *__restrict __format, ...);
/* Write formatted output to stdout.

   This function is a possible cancellation point and therefore not
   marked with __THROW.  */
extern int printf (const char *__restrict __format, ...);
-------------------------------------------------------------------------------
```

<p>
Compare the intermediate file(*code1.i*) with *stdio.h*. You will find that stdio.h is **copied** into *code1.i* with a few changes. You will find the above stuff even in *code1.i*.

Now, moving to the next part.

*stdio.h* starts with the following.
</p>

```c
-------------------------------------------------------------------------------
#ifndef _STDIO_H
#define _STDIO_H        1
-------------------------------------------------------------------------------
```

<p>
and ends with this.
</p>

```c
-------------------------------------------------------------------------------
#endif /* <stdio.h> included.  */
-------------------------------------------------------------------------------
```

<p>
The comment says ```"stdio.h included"```. What does this mean? What is ```#ifndef```, ```#endif```?

We have stumbled upon a piece of preprocessing. It is called **Conditional Compilation**. If you observe, *stdio.h* is included twice in *code1.c*. The header file has macros, typedefs, variables and function declarations. If you include the same file twice, all these macros, variables, declarations will be copied twice. What happens when the same variable(or function) is declared more than once? What happens when the same macro is defined more than once? Check it out. Write a simple program with a variable declared twice. The compiler throws an **error**. This means when the same header is included twice, the **compiler** in the next stage will throw an error. But the executable got generated without any error. How did it happen?

It is because of these.
</p>

```c
-------------------------------------------------------------------------------
#ifndef _STDIO_H
#define _STDIO_H        1
.
.
.
#endif /* <stdio.h> included.  */
-------------------------------------------------------------------------------
```

<p>
When the preprocessor encounters ```#include <stdio.h>``` for the first time, it starts preprocessing *stdio.h*. Now, preprocessor is inside *stdio.h*. Let us carefully understand what the above lines are doing. ```#ifndef _STDIO_H``` requests the preprocessor to check if the macro ```_STDIO_H``` is not defined. It is short for "**if _STDIO_H n**ot **def**ined". If that is true, the preprocessing continues. The next line is ```#define _STDIO_H 1```. Now, **_STDIO_H** has a value **1**. The ```#ifndef``` is ended with an ```#endif```. Only if that macro is not defined, everything between the ```#ifndef``` and its corresponding ```#endif``` is preprocessed. If the macro is already defined, then preprocessing for *stdio.h* stops here.

The preprocessor goes back to *code1.c* and it encounters ```#include <stdio.h>``` for the second time. It starts preprocessing *stdio.h* again. The very first statement is ```#ifndef _STDIO_H```. But **_STDIO_H** already has a value **1**. Because its already defined, preprocessing for *stdio.h* stops here.

This way, *stdio.h* is **not** copied into *code1.i*. For the secodn time and thus **no errors**. This was just one example for Conditional Compilation. Checkout *stdio.h*(or any header file). It will have a lot of such directives - ```#ifdef```, ```#ifndef```, ```#endif```, ```#pragma``` etc., Preprocessor resolves all these preprocessor-directives.

Let us move on to the next part of preprocessing. The ```#define``` macros are resolved by the preprocessor. In *code1.c*, whenever the macro **NUMBER** is used, it is replaced by its value **100**.

It removes the comments because they are useless from compilation perspective.

The following diagram summarizes the preprocessing sub-process.
</p>

```
-------------------------------------------------------------------------------
                                        code1.c
                                |--------------------|
                                | #include <stdio.h> |
                                | #include <stdlib.h>|
                                |--------------------|
                                   /              \
                                 ///             \ \  \ 
                              12///9             8\ \  \ 1
                               ///                 \ \  \ 
                               /                      \
                    stdlib.h  /                        \  stdio.h
                  |--------------------|     |--------------------|
   _STDLIB_H = 1  | #include <stddef.h>|     | #include <stddef.h>| _STDIO_H = 1
                  |--------------------|     |--------------------|
                           /                              \  
                         ///                             \ \ \ 
                      11///10                            7\ \ \ 2
                       ///                                 \ \ \
               stddef.h/                                      \ stddef.h
                   |----------------|                 |-----------------|
                   | #include <a.h> |                 | #include <a.h>  | _STDDEF_H = 1
                   | #include <b.h> |                 | #include <b.h>  |
                   |----------------|                 |-----------------|
                Not preprocessed because                     /    \   
                  _STDDEF_H is already                    6///5 4\ \ \ 3
                  defined.                                 /        \
                                                        |-----|   |-----|
                                               _B_H = 1 | b.h |   | a.h | _A_H = 1
                                                        |-----|   |-----|

-------------------------------------------------------------------------------
```

<p>
Let us end this section by getting a high-level understanding of preprocessing. Preprocessing happens in a **depth-first** manner. Preprocessor encounters ```#include <stdio.h>``` in *code1.c*. Preprocessor jumps to preprocess *stdio.h*. Say *stdio.h* has ```#include <stddef.h>```. Preprocessor jumps to preprocess *stddef.h*. Once *stddef.h* is preprocessed completely, it comes back to preprocess *stdio.h*. Once preprocessing of *stdio.h* is done, preprocessor comes back to *code1.c* and say it encounters ```#include <stdlib.h>```. Preprocessor takes *stdlib.h* for preprocessing. This happens till *code1.c* is entirely preprocessed and a final *code1.i* is generated. *code1.i* will contain contents of *stdio.h*, *stddef.h*, *stdlib.h* etc., in a preprocessed manner - with all preprocessor directives resolved.

At the end of this sub-process, we have a modified C sourcefile which has the all the header files in preprocessed manner and the code in the original C sourcefile.
</p>

## 1.4 Compiling { #Compiling1.4 }

<p>
We generally call the complete converion or a C sourcefile to an executable as Compiling / Compilation. it should be noted that Compiling is one step in the multi-step process.

As we discussed in the [Overview](#Overview1.2), the Compiler takes in *code1.i*(preprocessed C sourcefile, which is also a C sourcefile) as input and gives *code1.s*, as assembly sourcefile as output.

Let us checkout *code1.s* with the ```file``` command.
</p>

```
-------------------------------------------------------------------------------
chapter1$ file code1.s
code1.s: assembler source, ASCII text
-------------------------------------------------------------------------------
```

<p>
Let us checkout its contents.
</p>

```c
-------------------------------------------------------------------------code1.s
        .file   "code1.c"
        .globl  a
        .data
        .align 4
        .type   a, @object
        .size   a, 4
a:
        .long   10
        .comm   b,4,4
        .section        .rodata
.LC0:
        .string "Hello World!"
        .text
        .globl  main
        .type   main, @function
main:
.LFB0:
        .cfi_startproc
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register 6
        subq    $16, %rsp
        movl    $123, -8(%rbp)
        movl    $100, -4(%rbp)
        movb    $120, -9(%rbp)
        movl    $.LC0, %edi
        call    puts
        movl    $0, %eax
        leave
        .cfi_def_cfa 7, 8
        ret
        .cfi_endproc
.LFE0:
        .size   main, .-main
        .ident  "GCC: (Ubuntu 4.8.5-4ubuntu8) 4.8.5"
        .section        .note.GNU-stack,"",@progbits
-------------------------------------------------------------------------------
```

This is x86-64 assembly language which is used to program Intel and AMD processors. Let us understand this sourcefile line by line. Please go through *code1.c* again before this analysis.

1. It first specifies the C sourcefile used to generate it.
</p>

```c
-------------------------------------------------------------------------------
.file   "code1.c"
-------------------------------------------------------------------------------
```

<p>
2. This is something interesting. It specifies that the symbol **a** belongs to the **global scope** through the line ```.globl  a```.
</p>

```c
-------------------------------------------------------------------------------
        .globl  a
-------------------------------------------------------------------------------
```

<p>
What does this mean? In C, we know that a global variable can be read and written into by any function written in the same sourcefile. It means that the symbol **a** is visible to everyone in the **.text** section. Thinking from assembler's perspective, at this point it does not know anything about **a** except that it is a symbol and it belongs to **global scope**. It does not know that it is a variable.

3. The data section
</p>

```c
-------------------------------------------------------------------------------
        .data
        .align 4
        .type   a, @object
        .size   a, 4
a:
        .long   10
        .comm   b,4,4
-------------------------------------------------------------------------------
```

<p>
The **.data** signifies the beginning of the **.data** section. ```.align 4``` informs the assembler that this section is 4-byte aligned. What does this mean? Alignment informs us(or tools) how the memory should be viewed and how data needs to be stored in that piece of memory. Take the data section. It is 4-byte aligned. It should be viewed as a piece of memory made of 4-byte chunks.
</p>

```
-------------------------------------------------------------------------------
.data:
    <------4 bytes------>
    |----|----|----|----|
    |    |    |    |    |
    |----|----|----|----|

    |----|----|----|----|
    |    |    |    |    |
    |----|----|----|----|

    |----|----|----|----|
    |    |    |    |    |
    |----|----|----|----|
              .
              .
-------------------------------------------------------------------------------
```

<p>
When the program is put into memory, **a** is stored in the first 4 bytes, **b** is stored in the next 4 bytes(starting from byte 5).

If a piece of memory specified to be 8-byte aligned, then it should be viewed as an array of 8-byte chunks.
</p>

```
-------------------------------------------------------------------------------
.data:
    <-----------------8 bytes--------------->
    |----|----|----|----|----|----|----|----|
    |    |    |    |    |    |    |    |    |
    |----|----|----|----|----|----|----|----|

    |----|----|----|----|----|----|----|----|
    |    |    |    |    |    |    |    |    |
    |----|----|----|----|----|----|----|----|

    |----|----|----|----|----|----|----|----|
    |    |    |    |    |    |    |    |    |
    |----|----|----|----|----|----|----|----|
                        .
                        .
                        .
-------------------------------------------------------------------------------
```

<p>
Continuing, variable **a** is defined.
</p>

```
-------------------------------------------------------------------------------
        .type   a, @object
        .size   a, 4
a:
        .long   10
-------------------------------------------------------------------------------
```
<p>
**a** is of the type **object** and its size is 4 bytes. It is then defined to be an integer **10**. ```.long``` is equivalent to the ```int``` datatype in C.

Next comes **b**, a uninitialized global variable. It is not defined the way **a** is defined.
</p>

```
-------------------------------------------------------------------------------
.comm   b,4,4
-------------------------------------------------------------------------------
```

<p>
This informs the assembler that **b** is a **common** symbol, whose size is 4 bytes and should be 4-byte aligned. The first 4 signifies the size of the variable, the second 4 signifies the alignment. Because **b** is just declared by not defined by a particular value, this information is enough.

```.byte``` is an assembler directive lets you specify a byte value. ```.quad``` lets you specify an 8-byte value.

That was about the data section.

**4**. Next section is a new one to us.
</p>

```
-------------------------------------------------------------------------------
        .section        .rodata
.LC0:
        .string "Hello World!"
-------------------------------------------------------------------------------
```

<p>
This section's name is **.rodata**. It stands for **Read-Only Data**. All the read-only data - mostly strings. In this program, the only string present is **Hello World!**. To identify the string, the label ```.LC0``` is used.

**5**. Now comes the heart of the program, the **.text** section.
</p>

```
-------------------------------------------------------------------------------
        .text
        .globl  main
        .type   main, @function
main:
.LFB0:
        .cfi_startproc
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register 6
        subq    $16, %rsp
        movl    $123, -8(%rbp)
        movl    $100, -4(%rbp)
        movb    $120, -9(%rbp)
        movl    $.LC0, %edi
        call    puts
        movl    $0, %eax
        leave
        .cfi_def_cfa 7, 8
        ret
        .cfi_endproc
.LFE0:
        .size   main, .-main
-------------------------------------------------------------------------------
```
<p>
It declares a symbol **main** of type **function**. It is defined as a label. This is x86_64 assembly language, which is used to program 64-bit Intel and AMD processors.

In the end, it's size is calculated with ```.size main, .-main```.

6.The compiler version
</p>

```
-------------------------------------------------------------------------------
        .ident  "GCC: (Ubuntu 4.8.5-4ubuntu8) 4.8.5"
-------------------------------------------------------------------------------
```

<p>
7. In the end, there is one more section **.note.GNU-stack**.
</p>

```
-------------------------------------------------------------------------------
        .section        .note.GNU-stack,"",@progbits
-------------------------------------------------------------------------------
```
<p>
We will see what this section is when we discuss about ELF.

What all happens when a C sourcefile is compiled? The following are a few things which happen.

1. C source is converted to a particular assembly source.
2. It performs various types of optimizations, which helps in generating least amount of assembly code for a given C source.
3. It eliminates dead code. Sometimes, there will be pieces of code in your source which will never get executed. Such code is never included in the assembly sourcefile.
4. Probably the most significant one. It **strips** or **removes** names and datatypes of local variables. In *code1.c*, the ```main()``` function has 3 local variables - ```int c = 123, number = NUMBER``` and ```char d = 'x'```. If you look at assembly code, you will not find any names or explicit datatypes.

We will explore points 2, 3 and 4 in future chapters.

At the end of the compiling sub-process, the intermediate C sourcefile generates an assembly sourcefile which has a data section, a read-only data section, text section and the .note.GNU-stack section.
</p>

## 1.5 Assembling { #Assembling1.5 }

<p>
We now have an assembly sourcefile *code1.s*. It is assembled to give a linkable file *code1.o*. The linkable file is also known as **Object file** or **Relocatable file**.

What does an object file have? In [Init](#Init1.1), we saw that it has a bunch of sections - text, data, symbol table, fixing table. Open up this file in a text editor. It is binary and hard to understand. For that reason, let us use **readelf** - a tool which will help analyze ELF files to read through *code1.o*.
</p>

```
-------------------------------------------------------------------------------
chapter1$ readelf -S code1.o
There are 13 section headers, starting at offset 0x2e0:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       000000000000002b  0000000000000000  AX       0     0     1
  [ 2] .rela.text        RELA             0000000000000000  00000230
       0000000000000030  0000000000000018   I      10     1     8
  [ 3] .data             PROGBITS         0000000000000000  0000006c
       0000000000000004  0000000000000000  WA       0     0     4
  [ 4] .bss              NOBITS           0000000000000000  00000070
       0000000000000000  0000000000000000  WA       0     0     1
  [ 5] .rodata           PROGBITS         0000000000000000  00000070
       000000000000000d  0000000000000000   A       0     0     1
  [ 6] .comment          PROGBITS         0000000000000000  0000007d
       0000000000000024  0000000000000001  MS       0     0     1
  [ 7] .note.GNU-stack   PROGBITS         0000000000000000  000000a1
       0000000000000000  0000000000000000           0     0     1
  [ 8] .eh_frame         PROGBITS         0000000000000000  000000a8
       0000000000000038  0000000000000000   A       0     0     8
  [ 9] .rela.eh_frame    RELA             0000000000000000  00000260
       0000000000000018  0000000000000018   I      10     8     8
  [10] .symtab           SYMTAB           0000000000000000  000000e0
       0000000000000138  0000000000000018          11     9     8
  [11] .strtab           STRTAB           0000000000000000  00000218
       0000000000000017  0000000000000000           0     0     1
  [12] .shstrtab         STRTAB           0000000000000000  00000278
       0000000000000061  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
-------------------------------------------------------------------------------
```

<p>
ELF also has a concept of section headers. Each section header has metadata about a section. In this object file, there are 13 sections. A few which we know are

**.text**: Has machine code (which may need fixing).

We can dump the contents of a section using **readelf**.
</p>

```
-------------------------------------------------------------------------------
chapter1$ readelf --hex-dump .text code1.o

Hex dump of section '.text':
 NOTE: This section has relocations against it, but these have NOT been applied to this dump.
  0x00000000 554889e5 4883ec10 c745f87b 000000c7 UH..H....E.{....
  0x00000010 45fc6400 0000c645 f778bf00 000000e8 E.d....E.x......
  0x00000020 00000000 b8000000 00c9c3            ...........
-------------------------------------------------------------------------------
```

<p>
This is pure binary. We know that these are binary equivalent of the assembly instructions. Let us get the assembly equivalent of the above machine code. To get the assembly code back, you **disassemble** the assembled machine code. **objdump** is a useful tool which can used to disassemble assembled code.
</p>

```
---------------------------------------------------------------------
chapter1$ objdump -d code1.o

code1.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <main>:
   0:	55                   	push   %rbp
   1:	48 89 e5             	mov    %rsp,%rbp
   4:	48 83 ec 10          	sub    $0x10,%rsp
   8:	c7 45 f8 7b 00 00 00 	movl   $0x7b,-0x8(%rbp)
   f:	c7 45 fc 64 00 00 00 	movl   $0x64,-0x4(%rbp)
  16:	c6 45 f7 78          	movb   $0x78,-0x9(%rbp)
  1a:	bf 00 00 00 00       	mov    $0x0,%edi
  1f:	e8 00 00 00 00       	callq  24 <main+0x24>
  24:	b8 00 00 00 00       	mov    $0x0,%eax
  29:	c9                   	leaveq 
  2a:	c3                   	retq   
---------------------------------------------------------------------
```
<p>
Compare the above assembly code with assembly code present in *code1.s*. There will be some differences. The assembly code in *code1.s* used labels, directives. But this code is derived from assembled code. It means, this code needs fixing. Look at line **0x1f**. Its machine code is **b8-00-00-00-00**. It says ```callq 24```. Go back and look at *code1.s* - it is actually ``` call    puts```. Instead of ```call```(or ```callq```) is encoded as **b8**. The rest of the 4 bytes needs fixing. Those 4 bytes which are zeros need to be replaced with **puts**'s address.

That was about text section.

**.data**: Data section - Has all the global **initialized** variables.<p>

Lets dump its contents using **readelf**.
</p>

```
-------------------------------------------------------------------------------
chapter1$ readelf --hex-dump .data code1.o

Hex dump of section '.data':
  0x00000000 0a000000                            ....
-------------------------------------------------------------------------------
```

<p>
**0x0a** is **10** in hexadecimal system. This is the global variable **a**. **b** is also a global variable. But it is not present here. Where is it?

**.rodata**: Read-Only data section

The following is its dump.
</p>

```
-------------------------------------------------------------------------------
chapter1$ readelf --hex-dump .rodata code1.o

Hex dump of section '.rodata':
  0x00000000 48656c6c 6f20576f 726c6421 00       Hello World!.
-------------------------------------------------------------------------------
```

<p>
It has the one read-only element in our program, the **Hello World!** string.

**.symtab**: Symbol Table

Let us take a look at the symbol table's binary form.
</p>

```
-------------------------------------------------------------------------------
chapter1$ readelf --hex-dump .symtab code1.o

Hex dump of section '.symtab':
  0x00000000 00000000 00000000 00000000 00000000 ................
  0x00000010 00000000 00000000 01000000 0400f1ff ................
  0x00000020 00000000 00000000 00000000 00000000 ................
  0x00000030 00000000 03000100 00000000 00000000 ................
  0x00000040 00000000 00000000 00000000 03000300 ................
  0x00000050 00000000 00000000 00000000 00000000 ................
  0x00000060 00000000 03000400 00000000 00000000 ................
  0x00000070 00000000 00000000 00000000 03000500 ................
  0x00000080 00000000 00000000 00000000 00000000 ................
  0x00000090 00000000 03000700 00000000 00000000 ................
  0x000000a0 00000000 00000000 00000000 03000800 ................
  0x000000b0 00000000 00000000 00000000 00000000 ................
  0x000000c0 00000000 03000600 00000000 00000000 ................
  0x000000d0 00000000 00000000 09000000 11000300 ................
  0x000000e0 00000000 00000000 04000000 00000000 ................
  0x000000f0 0b000000 1100f2ff 04000000 00000000 ................
  0x00000100 04000000 00000000 0d000000 12000100 ................
  0x00000110 00000000 00000000 2b000000 00000000 ........+.......
  0x00000120 12000000 10000000 00000000 00000000 ................
  0x00000130 00000000 00000000                   ........
-------------------------------------------------------------------------------
```

<p>
We can't understand anything from it. Let us dump it in human readable form.
</p>

```
-------------------------------------------------------------------------------
chapter1$ readelf -s code1.o

Symbol table '.symtab' contains 13 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS code1.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    7 
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    8 
     8: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 
     9: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    3 a
    10: 0000000000000004     4 OBJECT  GLOBAL DEFAULT  COM b
    11: 0000000000000000    43 FUNC    GLOBAL DEFAULT    1 main
    12: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND puts
-------------------------------------------------------------------------------
```

<p>
It has 13 symbols. **Sourcefile-name** is itself a symbol. Come to entry 9. Symbol name is **a**. It is an object of 4 bytes and its scope is **GLOBAL**. Look at entry 11 - it belongs to **main**. It is a function of size 43 bytes with global scope. The most interesting one is entry 12- it belongs to **puts**. There was a ```call puts``` instruction in *code1.s*, but assembler didn't find **puts** anywhere. It created a symbol table entry of type **NOTYPE** and left.

Now, let us take up the other sections.

**.rela.text**: This is ELF's version of fixing table. This table has all the entries to fix the .text section.

Object files are also called relocatable files. Fixing formally is called **Relocation**. Take a look at the Relocation records for .text section.
</p>

```
-------------------------------------------------------------------------------
chapter1$ objdump -r code1.o

code1.o:     file format elf64-x86-64

RELOCATION RECORDS FOR [.text]:
OFFSET           TYPE              VALUE 
000000000000001b R_X86_64_32       .rodata
0000000000000020 R_X86_64_PC32     puts-0x0000000000000004
-------------------------------------------------------------------------------
```

<p>
Take a look at the first entry. It is at an offset **0x1b** bytes into the .text section. Type is **R_X86_64_32** - this has 2 pieces of information. First is that this is a relocation(fixing) specific to x86_64 architecture. Second is 32 bits(4 bytes) need fixing. Let us look at the disassembly and see what is present at byte **0x1b**.
</p>

```
-------------------------------------------------------------------------------
Disassembly of section .text:

0000000000000000 <main>:
   0:	55                   	push   %rbp
   1:	48 89 e5             	mov    %rsp,%rbp
   4:	48 83 ec 10          	sub    $0x10,%rsp
   8:	c7 45 f8 7b 00 00 00 	movl   $0x7b,-0x8(%rbp)
   f:	c7 45 fc 64 00 00 00 	movl   $0x64,-0x4(%rbp)
  16:	c6 45 f7 78          	movb   $0x78,-0x9(%rbp)
  1a:	bf 00 00 00 00       	mov    $0x0,%edi
  1f:	e8 00 00 00 00       	callq  24 <main+0x24>
  24:	b8 00 00 00 00       	mov    $0x0,%eax
  29:	c9                   	leaveq 
  2a:	c3                   	retq  
-------------------------------------------------------------------------------
```
<p>
Byte 0x1b belongs to the instructon **bf-00-00-00-00** - ```mov $0x0, %edi```. Take a look at *code1.s*. The 4 bytes need to be replaced with **Hello World!**'s address. That is what the table tells. The value which needs to be put in the place of those 4 zeros is address of .rodata which is the address of the string.

The next entry is to resolve undefined symbol **puts**.

**.comment**: This section has compiler version information.

You can use **readelf** to dump its contents.
</p>

```
-------------------------------------------------------------------------------
chapter1$ readelf --string-dump .comment code1.o

String dump of section '.comment':
  [     1]  GCC: (Ubuntu 4.8.5-4ubuntu8) 4.8.5
-------------------------------------------------------------------------------
```
<p>
**.strtab**: String Table

This section has all the strings used in this object file. The symbols, filename etc., are all stored in this section. Let us take a look at it.
</p>

```c
-------------------------------------------------------------------------------
chapter1$ readelf --string-dump .strtab code1.o

String dump of section '.strtab':
  [     1]  code1.c
  [     9]  a
  [     b]  b
  [     d]  main
  [    12]  puts
-------------------------------------------------------------------------------
```

<p>
I It has the file name **code1.c** and the 4 symbols we had used - **a**, **b**, **main** and **puts**.

**shstrtab**: Table of all the section names.

This section is another string table which contains only section names like .text, .data etc.,
</p>

```
-------------------------------------------------------------------------------
chapter1$ readelf --string-dump .shstrtab code1.o

String dump of section '.shstrtab':
  [     1]  .symtab
  [     9]  .strtab
  [    11]  .shstrtab
  [    1b]  .rela.text
  [    26]  .data
  [    2c]  .bss
  [    31]  .rodata
  [    39]  .comment
  [    42]  .note.GNU-stack
  [    52]  .rela.eh_frame
-------------------------------------------------------------------------------
```

<p>
It is strange that it does not contain **.text**.

That was about the assembling sub-process. It takes in an assembly sourcefile as input and outputs a linkable / relocatable file. The relocatable file contains information in the form of sections. We looked at a few important sections.
</p>

## 1.6 Linking { #Linking1.6 }

<p>
The final step of the conversion process is **linking**. We discussed that linking is key in resolving symbol references, giving addresses, stitching all the sections in the linkable files to generate the executable or library. Because libraries are shared by various programs, they are called **shared libraries**.

It takes in one or more relocatable files as input and generates an executable or a library as output - based on what is requested.

Before moving forward, let us read the GNU linker's manual page.
</p>

```
---------------------------------------------------------------------------------------
man ld
LD(1)           GNU Development Tools            LD(1)

NAME
       ld - The GNU linker

SYNOPSIS
       ld [options] objfile ...

DESCRIPTION
       ld combines a number of object and archive files, relocates their data and ties 
       up symbol references. Usually the last step in compiling a program is to run ld.
       .
       .
---------------------------------------------------------------------------------------
```

<p>
The manual page is a gold mine. It is full of interesting information.

We discussed in [Init](#Init1.1) that linker takes takes essential sections like data, text of one or more relocatable files, give addresses to them, stitch them together, resolve all the symbol references and generates metadata which helps in runtime linking of library functions. In the [previous section](#Assembling1.5), we took a look at symbol table, fixing table(relocatable record) used by the linker.

Linker generates an executable *code1*. Let us take a look at the entry-point address. The executable is also an ELF file. The entry-point address for an executable will be present in the ELF header (similar to BFF header). The ELF header has the metadata about the complete ELF file - it is a **roadmap** to the entire file.
</p>

```
-------------------------------------------------------------------------------
chapter1$ readelf -h code1
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x400400
  Start of program headers:          64 (bytes into file)
  Start of section headers:          6520 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           64 (bytes)
  Number of section headers:         30
  Section header string table index: 29
-------------------------------------------------------------------------------
```

<p>
The ELF header has a lot of metadata. It tells that this file is a 64-bit executable, following the little-endian byte ordering. It also has a type member. Because relocatable files, executables and libraries all are generated as ELF files, a type member is necessary. The type tells that this is an executable. The machine(architecture) it targets is AMD64. **Its entry-point address is 0x400400**. This means when the program is run, the first instruction will be at the address **0x400400** and control is transfered to that instruction.

There is something interesting here. So far, we discussed quite a bit about sections, section headers. We saw that relocatable files are **divided** into several **sections**. Executables and libraries can be copied onto memory and can be run. Such type of **runnable** files are divided into what are called **program segments**. Sections help the linker to generate the executable/library. Segments help the operating system run them. What do these segments contain? Do they have headers? Let us explore them now.

From the ELF header, it can be seen that there are **9** number of **Program Headers**. Program Headers are short form of **Program Segment Headers**. Each of these headers describe a particular segment. Let us list the 9 segments present in this file. Again, **readelf** can be used.
</p>

```
-------------------------------------------------------------------------------
chapter1$ readelf --segments code1

Elf file type is EXEC (Executable file)
Entry point 0x400400
There are 9 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000400040 0x0000000000400040
                 0x00000000000001f8 0x00000000000001f8  R      0x8
  INTERP         0x0000000000000238 0x0000000000400238 0x0000000000400238
                 0x000000000000001c 0x000000000000001c  R      0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x0000000000000700 0x0000000000000700  R E    0x200000
  LOAD           0x0000000000000e08 0x0000000000600e08 0x0000000000600e08
                 0x000000000000022c 0x0000000000000238  RW     0x200000
  DYNAMIC        0x0000000000000e20 0x0000000000600e20 0x0000000000600e20
                 0x00000000000001d0 0x00000000000001d0  RW     0x8
  NOTE           0x0000000000000254 0x0000000000400254 0x0000000000400254
                 0x0000000000000044 0x0000000000000044  R      0x4
  GNU_EH_FRAME   0x00000000000005c4 0x00000000004005c4 0x00000000004005c4
                 0x000000000000003c 0x000000000000003c  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
  GNU_RELRO      0x0000000000000e08 0x0000000000600e08 0x0000000000600e08
                 0x00000000000001f8 0x00000000000001f8  R      0x1

 Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp 
   02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .text .fini .rodata .eh_frame_hdr .eh_frame 
   03     .init_array .fini_array .jcr .dynamic .got .got.plt .data .bss 
   04     .dynamic 
   05     .note.ABI-tag .note.gnu.build-id 
   06     .eh_frame_hdr 
   07     
   08     .init_array .fini_array .jcr .dynamic .got 
-------------------------------------------------------------------------------
```
<p>
There are different types of segments - **PHDR**, **INTERP**, **LOAD**, **DYNAMIC** etc., The **linker** generates these program headers and puts it into the executable. See if relocatable files have program headers.

Dividing an executable/library into segments helps in running it. To analyze them, it is still useful to divide into sections. That is why, generally an executable/library is divided into both segments and sections. A segment would generally contains one or more sections. Look at that Section to Segment mapping. Segment-1 contains some section called the **.interp**. Segment-2 is a collection of lot of sections.

Below is the list of sections.
<p>

```
-------------------------------------------------------------------------------
chapter1$ readelf -S code1
There are 30 section headers, starting at offset 0x1978:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000400238  00000238
       000000000000001c  0000000000000000   A       0     0     1
  [ 2] .note.ABI-tag     NOTE             0000000000400254  00000254
       0000000000000020  0000000000000000   A       0     0     4
  [ 3] .note.gnu.build-i NOTE             0000000000400274  00000274
       0000000000000024  0000000000000000   A       0     0     4
  [ 4] .gnu.hash         GNU_HASH         0000000000400298  00000298
       000000000000001c  0000000000000000   A       5     0     8
  [ 5] .dynsym           DYNSYM           00000000004002b8  000002b8
       0000000000000060  0000000000000018   A       6     1     8
  [ 6] .dynstr           STRTAB           0000000000400318  00000318
       000000000000003d  0000000000000000   A       0     0     1
  [ 7] .gnu.version      VERSYM           0000000000400356  00000356
       0000000000000008  0000000000000002   A       5     0     2
  [ 8] .gnu.version_r    VERNEED          0000000000400360  00000360
       0000000000000020  0000000000000000   A       6     1     8
  [ 9] .rela.dyn         RELA             0000000000400380  00000380
       0000000000000030  0000000000000018   A       5     0     8
  [10] .rela.plt         RELA             00000000004003b0  000003b0
       0000000000000018  0000000000000018  AI       5    23     8
  [11] .init             PROGBITS         00000000004003c8  000003c8
       0000000000000017  0000000000000000  AX       0     0     4
  [12] .plt              PROGBITS         00000000004003e0  000003e0
       0000000000000020  0000000000000010  AX       0     0     16
  [13] .text             PROGBITS         0000000000400400  00000400
       00000000000001a2  0000000000000000  AX       0     0     16
  [14] .fini             PROGBITS         00000000004005a4  000005a4
       0000000000000009  0000000000000000  AX       0     0     4
  [15] .rodata           PROGBITS         00000000004005b0  000005b0
       0000000000000011  0000000000000000   A       0     0     4
  [16] .eh_frame_hdr     PROGBITS         00000000004005c4  000005c4
       000000000000003c  0000000000000000   A       0     0     4
  [17] .eh_frame         PROGBITS         0000000000400600  00000600
       0000000000000100  0000000000000000   A       0     0     8
  [18] .init_array       INIT_ARRAY       0000000000600e08  00000e08
       0000000000000008  0000000000000008  WA       0     0     8
  [19] .fini_array       FINI_ARRAY       0000000000600e10  00000e10
       0000000000000008  0000000000000008  WA       0     0     8
  [20] .jcr              PROGBITS         0000000000600e18  00000e18
       0000000000000008  0000000000000000  WA       0     0     8
  [21] .dynamic          DYNAMIC          0000000000600e20  00000e20
       00000000000001d0  0000000000000010  WA       6     0     8
  [22] .got              PROGBITS         0000000000600ff0  00000ff0
       0000000000000010  0000000000000008  WA       0     0     8
  [23] .got.plt          PROGBITS         0000000000601000  00001000
       0000000000000020  0000000000000008  WA       0     0     8
  [24] .data             PROGBITS         0000000000601020  00001020
       0000000000000014  0000000000000000  WA       0     0     8
  [25] .bss              NOBITS           0000000000601034  00001034
       000000000000000c  0000000000000000  WA       0     0     4
  [26] .comment          PROGBITS         0000000000000000  00001034
       0000000000000023  0000000000000001  MS       0     0     1
  [27] .symtab           SYMTAB           0000000000000000  00001058
       0000000000000630  0000000000000018          28    46     8
  [28] .strtab           STRTAB           0000000000000000  00001688
       00000000000001e4  0000000000000000           0     0     1
  [29] .shstrtab         STRTAB           0000000000000000  0000186c
       0000000000000108  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
-------------------------------------------------------------------------------
```
<p>
Again note that segments are the ones which help in running the executable/library. Sections are just another way to partition it - which helps in analyzing the file better.

Let us start with something we have discussed. Lets starts with the type of symbols present in *code1*. Use **readelf** to dump them.
</p>

```
-------------------------------------------------------------------------------
chapter1$ readelf -s code1

Symbol table '.dynsym' contains 4 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.2.5 (2)
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@GLIBC_2.2.5 (2)
     3: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__

Symbol table '.symtab' contains 66 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000400238     0 SECTION LOCAL  DEFAULT    1 
     2: 0000000000400254     0 SECTION LOCAL  DEFAULT    2 
     3: 0000000000400274     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000400298     0 SECTION LOCAL  DEFAULT    4 
     5: 00000000004002b8     0 SECTION LOCAL  DEFAULT    5 
     6: 0000000000400318     0 SECTION LOCAL  DEFAULT    6 
     7: 0000000000400356     0 SECTION LOCAL  DEFAULT    7 
     8: 0000000000400360     0 SECTION LOCAL  DEFAULT    8 
     9: 0000000000400380     0 SECTION LOCAL  DEFAULT    9 
    10: 00000000004003b0     0 SECTION LOCAL  DEFAULT   10 
    11: 00000000004003c8     0 SECTION LOCAL  DEFAULT   11 
    12: 00000000004003e0     0 SECTION LOCAL  DEFAULT   12 
    13: 0000000000400400     0 SECTION LOCAL  DEFAULT   13 
    14: 00000000004005a4     0 SECTION LOCAL  DEFAULT   14 
    15: 00000000004005b0     0 SECTION LOCAL  DEFAULT   15 
    16: 00000000004005c4     0 SECTION LOCAL  DEFAULT   16 
    17: 0000000000400600     0 SECTION LOCAL  DEFAULT   17 
    18: 0000000000600e08     0 SECTION LOCAL  DEFAULT   18 
    19: 0000000000600e10     0 SECTION LOCAL  DEFAULT   19 
    20: 0000000000600e18     0 SECTION LOCAL  DEFAULT   20 
    21: 0000000000600e20     0 SECTION LOCAL  DEFAULT   21 
    22: 0000000000600ff0     0 SECTION LOCAL  DEFAULT   22 
    23: 0000000000601000     0 SECTION LOCAL  DEFAULT   23 
    24: 0000000000601020     0 SECTION LOCAL  DEFAULT   24 
    25: 0000000000601034     0 SECTION LOCAL  DEFAULT   25 
    26: 0000000000000000     0 SECTION LOCAL  DEFAULT   26 
    27: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    28: 0000000000600e18     0 OBJECT  LOCAL  DEFAULT   20 __JCR_LIST__
    29: 0000000000400440     0 FUNC    LOCAL  DEFAULT   13 deregister_tm_clones
    30: 0000000000400470     0 FUNC    LOCAL  DEFAULT   13 register_tm_clones
    31: 00000000004004b0     0 FUNC    LOCAL  DEFAULT   13 __do_global_dtors_aux
    32: 0000000000601034     1 OBJECT  LOCAL  DEFAULT   25 completed.7098
    33: 0000000000600e10     0 OBJECT  LOCAL  DEFAULT   19 __do_global_dtors_aux_fin
    34: 00000000004004d0     0 FUNC    LOCAL  DEFAULT   13 frame_dummy
    35: 0000000000600e08     0 OBJECT  LOCAL  DEFAULT   18 __frame_dummy_init_array_
    36: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS code1.c
    37: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    38: 00000000004006fc     0 OBJECT  LOCAL  DEFAULT   17 __FRAME_END__
    39: 0000000000600e18     0 OBJECT  LOCAL  DEFAULT   20 __JCR_END__
    40: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS 
    41: 0000000000600e10     0 NOTYPE  LOCAL  DEFAULT   18 __init_array_end
    42: 0000000000600e20     0 OBJECT  LOCAL  DEFAULT   21 _DYNAMIC
    43: 0000000000600e08     0 NOTYPE  LOCAL  DEFAULT   18 __init_array_start
    44: 00000000004005c4     0 NOTYPE  LOCAL  DEFAULT   16 __GNU_EH_FRAME_HDR
    45: 0000000000601000     0 OBJECT  LOCAL  DEFAULT   23 _GLOBAL_OFFSET_TABLE_
    46: 00000000004005a0     2 FUNC    GLOBAL DEFAULT   13 __libc_csu_fini
    47: 0000000000601020     0 NOTYPE  WEAK   DEFAULT   24 data_start
    48: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@@GLIBC_2.2.5
    49: 0000000000601038     4 OBJECT  GLOBAL DEFAULT   25 b
    50: 0000000000601034     0 NOTYPE  GLOBAL DEFAULT   24 _edata
    51: 00000000004005a4     0 FUNC    GLOBAL DEFAULT   14 _fini
    52: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@@GLIBC_
    53: 0000000000601020     0 NOTYPE  GLOBAL DEFAULT   24 __data_start
    54: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
    55: 0000000000601028     0 OBJECT  GLOBAL HIDDEN    24 __dso_handle
    56: 00000000004005b0     4 OBJECT  GLOBAL DEFAULT   15 _IO_stdin_used
    57: 0000000000400530   101 FUNC    GLOBAL DEFAULT   13 __libc_csu_init
    58: 0000000000601040     0 NOTYPE  GLOBAL DEFAULT   25 _end
    59: 0000000000400430     2 FUNC    GLOBAL HIDDEN    13 _dl_relocate_static_pie
    60: 0000000000400400    43 FUNC    GLOBAL DEFAULT   13 _start
    61: 0000000000601030     4 OBJECT  GLOBAL DEFAULT   24 a
    62: 0000000000601034     0 NOTYPE  GLOBAL DEFAULT   25 __bss_start
    63: 00000000004004fd    43 FUNC    GLOBAL DEFAULT   13 main
    64: 0000000000601038     0 OBJECT  GLOBAL HIDDEN    24 __TMC_END__
    65: 00000000004003c8     0 FUNC    GLOBAL DEFAULT   11 _init
-------------------------------------------------------------------------------
```

<p>
We discussed in [Init](#Init1.1) that an executable will contain symbols that need to be resolved at runtime. This type of runtime linking is called **dynamic linking**. We discussed a type of symbol resolution where all the symbols are resolved before the program is run. But in **dynamic linking**, a symbol is resolved only when it is referenced. For example, there is a call to ```printf```. In the method we discussed before, its address is obtained before running the program. Better method is to resolve only when it is referenced. What if it is never called? It never has to be resolved. Be lazy, do work only if needed, when needed. This type of linking is also called **lazy linking**.

Observe the above readelf output. It dumped 2 types of symbol tables.

1. **.dynsym**:
</p>

```
-------------------------------------------------------------------------------
Symbol table '.dynsym' contains 4 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.2.5 (2)
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@GLIBC_2.2.5 (2)
     3: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
-------------------------------------------------------------------------------
```
<p>
As expected, it has **puts**. It has figured out that **puts** is present in the C library. The other 2 entries are added by GNU Linker.

2. **.symtab**: The normal symbol table

This symbol table has lots of symbols. It has undefined symbols like **puts**, **__libc_start_main** - basically which are present in **.dynsym**. It also has symbols which are resolved and have absolute addresses - it has **a**, **b**, **main**. When they are already resolved, why are they present? It helps in debugging. They are **not necessary** to run it. Even if you remove it, the program should run just fine. Let us test that. Run the program once. Also make note of the executable's size.
</p>

```
-------------------------------------------------------------------------------
chapter1$ ls -l code1
-rwxr-xr-x 1 dell dell 8440 Mar 23 00:53 code1
chapter1$ ./code1
Hello World!
-------------------------------------------------------------------------------
```

<p>
It is running properly. There is a tool called **strip** which can be used to remove(or strip) any specified section. Let us remove the **.symtab** section.
</p>

```
-------------------------------------------------------------------------------
chapter1$ strip --remove-section .symtab code1
dell@adwi:~/Documents/projects/books/binary-exploitation/exp/chapter1$ ls -l code1
-rwxr-xr-x 1 dell dell 6224 Mar 23 01:15 code1
-------------------------------------------------------------------------------
```

<p>
Look at how fat the symbol table was. List all the sections and make sure this section is not present in the executable. If you checkout the symbols using ```readelf -s``` command, you will get only the **.dynsym** which **must** be present.

Along with a symbol table containing unresolved symbols, a fixing table(relocation records) should be present which will help in runtime linking.
</p>

```
-------------------------------------------------------------------------------
chapter1$ readelf -r code1

Relocation section '.rela.dyn' at offset 0x380 contains 2 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000600ff0  000200000006 R_X86_64_GLOB_DAT 0000000000000000 __libc_start_main@GLIBC_2.2.5 + 0
000000600ff8  000300000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0

Relocation section '.rela.plt' at offset 0x3b0 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000601018  000100000007 R_X86_64_JUMP_SLO 0000000000000000 puts@GLIBC_2.2.5 + 0
-------------------------------------------------------------------------------
```

<p>
There you go. For every unresolved symbol in **.dynsym**, there is a relocation record.

This is not enough. We need to know the symbol-library relationship - basically information which tells that **puts** is defined in a particular library. These strings are present in the **.dynstr** section.
</p>

```
-------------------------------------------------------------------------------
chapter1$ readelf --string-dump .dynstr code1

String dump of section '.dynstr':
  [     1]  libc.so.6
  [     b]  puts
  [    10]  __libc_start_main
  [    22]  GLIBC_2.2.5
  [    2e]  __gmon_start__
-------------------------------------------------------------------------------
```

<p>
That was about information required for proper dynamic linking.

Checkout the .data and .text sections too.

Going back to exploring segments, the following is the list.

1. PHDR
2. INTERP
3. LOAD
4. LOAD
5. DYNAMIC
6. NOTE
7. GNU_EH_FRAME
8. GNU_STACK
9. GNU_RELRO

Each of these segments has an essential piece of the executable/library without which it cannot be run. The **LOAD** type segments are directly copied into the main memory. The **DYNAMIC** segment has information about dynamic linking - The symbol table with unresolved symbols, library-list of functions/variables, a hash-table which helps in accessing symbol table efficiently and more. The **NOTE** has some information about the file. The **GNU_STACK** and **GNU_RELRO** are necessary to administer important security features. Other segment types will be discussed in future chapters.

We discussed that the section-wise division of an executable does not help in running it. How do we verify it?

The section-headers has information about how the executable is divided into sections - what part of executable is what section. Without section-headers, all this information is lost. Even with that information lost, we should be able to run it. Let us delete all the section-headers in *code1*.

The size of *code1* is 8448 bytes. From the ELF header, the section-headers array starts at a byte offet 6520. There are 30 section headers each of size 64 bytes. Total size of section-headers array is 30X64 = 1920 bytes. By observation, 6520+1920 = 8440 = *code1*'s size. This means, the section-headers array starts from byte 6520 and goes till the end of the file. We will simply chop it off and see if the program can still run. We can use this command called **dd** to get the job done.
</p>

```
-------------------------------------------------------------------------------
chapter1$ dd if=code1 of=code1.nosec bs=1 count=6520
6520+0 records in
6520+0 records out
6520 bytes (6.5 kB, 6.4 KiB) copied, 0.0357093 s, 183 kB/s
-------------------------------------------------------------------------------
```

<p>
The first 6520 bytes are copied into *code1.nosec*. Try running this new file.
</p>

```
-------------------------------------------------------------------------------
chapter1$ chmod u+x code1.nosec 
dell@adwi:~/Documents/projects/books/binary-exploitation/exp/chapter1$ ./code1.nosec 
Hello World!
-------------------------------------------------------------------------------
```

<p>
Look at that. Section headers are not needed.

Experiment with program headers. You will see that corrupting these headers won't allow you to run the program. You will mostly probably end up with a runtime error.

That was about linking. We skimmed through contents of an executable, introduced the concept of segments and did some experimentation. Successful linking of the given bunch of relocatable files will give you an executable/library which can be run. Linker also adds information essential for dynamic linking.
</p>

## 1.7 Further exploration

### 1.7.1 Entry Point of a program

<p>
We saw that there should be an entry-point for any executable. For a C program, it is logical to think that the  ```main()``` function is its entry-point. Is it really the entry point? Let us dig deep.

Using *code1* for this. We saw that the ELF header has the entry-point address. Let us check it out.
</p>

```
-------------------------------------------------------------------------------
chapter1$ readelf -h code1
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x400400
  Start of program headers:          64 (bytes into file)
  Start of section headers:          6520 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           64 (bytes)
  Number of section headers:         30
  Section header string table index: 29
-------------------------------------------------------------------------------
```

<p>
The entry-point address is **0x400400**. Let us see if we can get a symbol in the symbol table for this address.
</p>

```
-------------------------------------------------------------------------------
chapter1$ readelf -s code1
.
.
    58: 0000000000601040     0 NOTYPE  GLOBAL DEFAULT   25 _end
    59: 0000000000400430     2 FUNC    GLOBAL HIDDEN    13 _dl_relocate_static_pie
    60: 0000000000400400    43 FUNC    GLOBAL DEFAULT   13 _start
    61: 0000000000601030     4 OBJECT  GLOBAL DEFAULT   24 a
    62: 0000000000601034     0 NOTYPE  GLOBAL DEFAULT   25 __bss_start
    63: 00000000004004fd    43 FUNC    GLOBAL DEFAULT   13 main
.
.
-------------------------------------------------------------------------------
```

<p>
The ```_start``` symbol is given the address **0x400400**. ```main```'s address is **0x4004fd**. This means ```_start``` is the starting point of this executable. But we never defined the ```_start```. All we did is define the ```main``` function. How do we explain this?

Compare the disassembly of *code1.o* and *code1*. The text section of *code1.o* has ```main``` but *code1* has lot of other things including the ```_start*```.
</p>

```c
-------------------------------------------------------------------------------
Disassembly of section .text:

0000000000400400 <_start>:
  400400:	31 ed                	xor    ebp,ebp
  400402:	49 89 d1             	mov    r9,rdx
  400405:	5e                   	pop    rsi
  400406:	48 89 e2             	mov    rdx,rsp
  400409:	48 83 e4 f0          	and    rsp,0xfffffffffffffff0
  40040d:	50                   	push   rax
  40040e:	54                   	push   rsp
  40040f:	49 c7 c0 a0 05 40 00 	mov    r8,0x4005a0
  400416:	48 c7 c1 30 05 40 00 	mov    rcx,0x400530
  40041d:	48 c7 c7 fd 04 40 00 	mov    rdi,0x4004fd
  400424:	ff 15 c6 0b 20 00    	call   QWORD PTR [rip+0x200bc6]        # 600ff0 <__libc_start_main@GLIBC_2.2.5>
  40042a:	f4                   	hlt    
  40042b:	0f 1f 44 00 00       	nop    DWORD PTR [rax+rax*1+0x0]
-------------------------------------------------------------------------------
```

<p>
This code was **added** by the linker. Execution starts with ```_start```, later ```main``` is called. ```main``` is the entry-point from programmer's perspective, ```_start``` is the entry-point from system's perspective.

Linker's manual page has some details related to this.
</p>

```
-------------------------------------------------------------------------------
       -e entry
       --entry=entry
           Use entry as the explicit symbol for beginning execution of your program, rather than the default entry
           point.  If there is no symbol named entry, the linker will try to parse entry as a number, and use that
           as the entry address (the number will be interpreted in base 10; you may use a leading 0x for base 16,
           or a leading 0 for base 8).
-------------------------------------------------------------------------------
```

<p>
There is an option to specify the entry-point. Let us try that out.
</p>

```
-------------------------------------------------------------------------------
chapter1$ gcc-4.8 code1.c -o code1.newstart --entry=main
-------------------------------------------------------------------------------
```

<p>
Checkout the ELF header.
</p>

```
-------------------------------------------------------------------------------
chapter1$ readelf -h code1.newstart
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x4004fd
  Start of program headers:          64 (bytes into file)
  Start of section headers:          6520 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           64 (bytes)
  Number of section headers:         30
  Section header string table index: 29

chapter1$ readelf -s code1
.
.
    60: 0000000000400400    43 FUNC    GLOBAL DEFAULT   13 _start
    61: 0000000000601030     4 OBJECT  GLOBAL DEFAULT   24 a
    62: 0000000000601034     0 NOTYPE  GLOBAL DEFAULT   25 __bss_start
    63: 00000000004004fd    43 FUNC    GLOBAL DEFAULT   13 main
.
.
-------------------------------------------------------------------------------
```

<p>
Now, entry-point is **main**. Lets run it.
</p>

```
-------------------------------------------------------------------------------
chapter1$ ./code1.newstart
Hello World!
Segmentation fault (core dumped)
-------------------------------------------------------------------------------
```

<p>
The program ran, but it crashed.

To understand this, we will have to see what was executed between **_start** and **main**.
</p>

### 1.7.2 Can intermediate files be generated separately?

<p>
There are 3 intermediate files which are generated when a C sourcefile is compiled - preprocessed sourcefile(**.i**), assembly sourcefile(**.s**) and relocatable file(**.o**). They were stored on the filesystem by using the **--save-temps** compiler option.

It helps sometimes to generate each of these files separately. Let us see how it can be done.

When **gcc** is invoked to generate the executable/library, it feels that it is generating all the intermediate files. But that is not the case. It is invoking other programs which generate these files.

**1**. Preprocessed sourcefile: 

The **cpp** can be separately invoked and any code can be just preprocessed. Take a look.
</p>

```c
-------------------------------------------------------------------------------
chapter1$ cpp
#define NUMBER 100
int a = NUMBER;
int main()
{
	int b = NUMBER;
	printf("Hello!\n");
	return 0;
}
^D
# 1 "<stdin>"
# 1 "<built-in>"
# 1 "<command-line>"
# 31 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 32 "<command-line>" 2
# 1 "<stdin>"

int a = 100;
int main()
{
 int b = 100;
 printf("Hello!\n");
 return 0;
}
-------------------------------------------------------------------------------
```

<p>
Look at that. Preprocessed. Invoking **cpp** on *code1.c** will give **code1.i**.

**2**. Assembly sourcefiles

**gcc** actually compiles the preprocessed sourcefile to assembly file. The **-S** option can be used to generate just the **.s** file.
</p>

```
-------------------------------------------------------------------------------
chapter1$ ls code3*
code3.c

chapter1$ gcc code3.c -S

chapter1$ ls code3*
code3.c  code3.s
-------------------------------------------------------------------------------
```
<p>
**3**. Relocatable files

There are 2 ways to generate **.o** files. **gcc** along with **-c** option can be used to generate just the relocatable file.
</p>

```
-------------------------------------------------------------------------------
chapter1$ rm code3.s

chapter1$ gcc -c code3.c

chapter1$ ls code3*
code3.c  code3.o
-------------------------------------------------------------------------------
```

### 1.7.3 How are libraries made?

<p>
We focused on executable generation in previous sections. Let us take a look at how a shared object is generated.

Consider the following example.
</p>

```c
--------------------------------------------------------------------------lib.c
#include <stdio.h>

void lib_func()
{
	printf("Inside lib_func()\n");
}
-------------------------------------------------------------------------------
```

<p>
It is a dummy library with one function. Let us generate a library out of it.

1. Generate the corresponding relocatable file with a special option **fPIC**.
</p>

```
-------------------------------------------------------------------------------
chapter1$ ls lib*
lib.c
chapter1$ gcc-4.8 lib.c -c -fPIC
chapter1$ ls lib*
lib.c  lib.o
-------------------------------------------------------------------------------
```

<p>
**PIC** stands for **Position Independent Code**. For executables, we saw that linker gave addresses. Here, we are requesting the operating system to give addresses using the **fPIC** option. Why is this necessary? This will be discussed in one of the future chapters. Position Independent Code itself is a very interesting topic and there are a lot of other concepts attached to it.

Now, let us request the linker to generate a shared library using the **--shared** option.
</p>

```
-------------------------------------------------------------------------------
~/Documents/projects/books/binary-exploitation/exp/chapter1$ gcc-4.8 lib.o -o libdummy.so --shared
chapter1$ file libdummy.so
libdummy.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=03d8ebadfe41d8619f88b033cfe5c2835bef3927, not stripped
-------------------------------------------------------------------------------
```

<p>
With that, we have our shared library ready. A shared library is also called a shared object. I want you to analyze *libdummy.so* using readelf, objdump etc., the way we analyzed *code1*.

Every library has corresponding header file(s). The header files informs the programmer what functions, data structures are offered by that library. Let us write a small header file *lib.h*.
</p>

```c
--------------------------------------------------------------------------lib.h
#ifndef _LIB_H
#define _LIB_H 1

void lib_func();

#endif	/* _LIB_H */
-------------------------------------------------------------------------------
```
<p>
Let us use the above library and write a program.
</p>

```c
------------------------------------------------------------------------code2.c
#include "lib.h"
#include <stdio.h>

int main()
{
	printf("Inside main, before lib_func()\n");
	lib_func();
	printf("Inside main, after lib_func()\n");

	return 0;
}
-------------------------------------------------------------------------------
```
<p>
Try compiling *code2.c*.
</p>

```
-------------------------------------------------------------------------------
chapter1$ gcc-4.8 code2.c -o code2
/tmp/cczqD3tj.o: In function `main':
code2.c:(.text+0x16): undefined reference to `lib_func'
collect2: error: ld returned 1 exit status
-------------------------------------------------------------------------------
```

<p>
As expected. We got an **undefined reference** error. The symbol **lib_func** is used, but it is not defined anywhere in *code2.c*. We expect this to be resolved at runtime. We know that it is defined in *libdummy.so*, but we need to inform the linker about this right? We need to specify where our library is present and the name of the library.
</p>

```
-------------------------------------------------------------------------------
chapter1$ gcc-4.8 code2.c -o code2 -L. -ldummy
chapter1$ ls -l code2
-rwxr-xr-x 1 dell dell 8336 Mar 23 12:54 code2
-------------------------------------------------------------------------------
```


<p>
**-L** option is used to specify the location of the library. **-l** option is used to specify the library. It successfully compiled to give the executable. Try running it.
</p>

```
-------------------------------------------------------------------------------
chapter1$ ./code2
./code2: error while loading shared libraries: libdummy.so: cannot open shared object file: No such file or directory
-------------------------------------------------------------------------------
```

<p>
You compiled properly. The loader is responsible for loading all the libraries a program depends on. The loader does not know where *libdummy.so* is. Let us inform it. The ```LD_LIBRARY_PATH``` environment variable can be updated. We can update it this way.
</p>

```
-------------------------------------------------------------------------------
chapter1$ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:.
-------------------------------------------------------------------------------
```

<p>
The **.** is representative of the present working directory. Add it to the environment variable. Now let us run the program.
</p>

```
-------------------------------------------------------------------------------
chapter1$ ./code2
Inside main, before lib_func()
Inside lib_func()
Inside main, after lib_func()
-------------------------------------------------------------------------------
```


Check the dynamic linking information present in *code2* using readelf, objdump.
</p>

### 1.7.4 Other binary file formats?

<p>
In this chapter, we created a crude binary file format to understand certain concepts. We also explored parts of ELF. Are there other binary file formats? What format is used for Windows executables? for macOS?

There are lot of other binary file formats out there.

Windows uses an Executable Format called **Portable Executable(PE)** and object file format called **Common Object File Format(COFF)**. Apple uses **Mach object file format(Mach-O)**.

These are the popular formats. Many operating systems use their own formats.

Now that you have a general understanding of ELF, you can go ahead and try reading about PE, Mach-O. They are all similar in a sense: All have the basic sections, symbol table, string table, relocatable table, file-header etc., Even if there are differences, you should be able to understand it.
</p>

## 1.8 Fini { #Fini1.8 }

<p>
With that, we have come to the end of this chapter.

We explored the 4 steps involved in converting C programs into executables. The 4 steps are Preprocessing, Compiling, Assembling and Linking. We vaguely explored the Executable and Linkable Format(ELF).

A few important concepts were conveniently avoided during the chapter. When we were discussing about addresses and memory in [Init](#Init1.1), it was not specified what type of addresses we are talking about - physical or virtual addresses. In the swap program we wrote, all 3 were global variables. I did not mention anything about local variables. And a few things were pushed to future chapters because they needed background. ELF is a complex file format. It is a tightly knit pack of variety of data structures and exploring it in detail will have to be done in another chapter. Because binary file formats are closely bound to the compilation process, a few details of ELF was introduced to understand the internals of compiling. Reading and understanding assembly code was not the aim of this chapter, so it was avoided. I kept telling that the OS copies program into memory. Today's operating system does the bare minimum to copy a program into memory. It checks the validity of the program(does it abide by the rules of file format, OS etc.,). If it does, then it invokes a helper program to do the rest - copy the complete program into memory, check its dependent libraries and copy them, transfer control to entry-point of the program, dynamic linking etc.,. Exploring all this will take up a complete chapter. To compile *code1*, gcc-4.8 was used but my system comes with gcc-7. The differences between their outputs is difficult to explain without background - so gcc-4.8 was used.

When I first tried to understand Internals of Compiling with gcc, readelf, objdump etc., it felt pretty heavy and hard to understand. Understanding ELF was difficult because it has lots of things in it. Simply listing that a relocatable file has text, data, symbol table, fixing table etc., felt dry. Even once I knew why a symbol table is used, why a fixing table is used, it was not satisfying. Why is something present? What problem did it solve? Needed an answer to that. The [Init](#Init1.1) section was a very joyous experiment. Life started out with 0s and 1s. We now have a complex binary file format which tells how programs should be. I know the starting point and the ending point. There was a direction - whatever was done, it was to make programmers' life easier, run programs faster, easy debugging. With that, a path was followed in [Init](#Init1.1) to slowly build from 0s and 1s to a binary file format. I don't know if this path was taken to develop ELF starting from 0s and 1s, but I hope that the [Init](#Init1.1) section gives at least an intuition in that direction.
</p>

## 1.9 Further Reading

<p>
1. Manual pages of
    * elf
    * as
    * ld
    * any tool you use. There is always something interesting!

2. ELF related resources
    * [ELF Specification](https://refspecs.linuxbase.org/elf/elf.pdf)
    * [Linux Foundation Referenced Specifications](https://refspecs.linuxbase.org/)
    * [Introduction to ELF](https://people.redhat.com/mpolacek/src/devconf2012.pdf) by RedHat

3. Resources related to other binary file formats
     * [Mach-O File format reference](https://github.com/aidansteele/osx-abi-macho-file-format-reference)
     * [Official doc for Windows' PE](https://docs.microsoft.com/en-us/windows/win32/debug/pe-format)
</p>