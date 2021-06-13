# Chapter 4: x64 Program Execution Internals { #Chapter4 }

-------------------------------------------------------------------------------
**Summary**
In this chapter, we explore how programs are executed. We will be analyzing 
various C constructs at assembly level. We explore how function calls work,
how local variables are stored, how parameters are passed to a function and
more in 64-bit programs.
-------------------------------------------------------------------------------

## 4.1 Init { #Section4.1 }

<p>

In the previous chapters, we discussed a lot of things revolving around a program - in what file format the program should be stored, how does the x86 assembly language look like, how does the program look like once it is loaded into memory for execution. We have not taken a deep look at the program itself. A C program we write can have a lot of things - variables, pointers, switch blocks, if-else-if code, loops, function calls, structures and more. What does each of these C constructs look like at assembly level? This chapter explores this question. We will go through each C construct and explore it at the assembly level.

We will be focusing on x64 architecture in this chapter. You are expected to know C. Also suggest you to read [Chapter2](#Chapter2) before proceeding with this chapter.

</p>

## 4.2 How does a function call work? { #Section4.2 }

<p>

In C, all the code is divided into meaningful procedures(or functions). Consider the following program.

</p>

```c
------------------------------------------------------------------------code1.c
  1 void func()
  2 {
  3         return;
  4 }
  5 
  6 
  7 int main()
  8 {   
  9         // Call it
 10         func();
 11 
 12         return 0;
 13 }
-------------------------------------------------------------------------------
```

<p>

It is a very simple program. The ```main()``` function calls the ```func()``` function. Once ```func()``` is done running, it returns the control back to ```main()``` and ```main()``` should continue from where it stopped. By looking at the C code, we could figure this out easily. Take a look at the following diagram.

</p>

```
-------------------------------------------------------------------------------
           main (line 8)
            ||
            ||
            \/
           call func() (line 10)
            ||
            ----------->----------->------------||
                                                ||
                                                \/
                                               func (line 2)
                                                ||
                                                || (func gets executed.)
                                                \/
            ||-----------------------------------                                                
            main (line 11 - code present right after the call to func())
            ||
            ||
            \/
            main is done.
-------------------------------------------------------------------------------
```

<p>

Note that after ```func()``` is done executing, control should be transfered to ```main()``` but not to the starting of ```main()```. It needs to transfer control to line 11 - the line right after ```func()``` is called in ```main()```. This needs to be implemented at assembly level. How can this be implemented? Take some time and come up with a mechanism. Idea is, when ```func()``` is done execution, it should transfer control to some line in ```main()```. To do this, ```func()``` should know which line in ```main()``` it needs to transfer the control to. What can be done is, we store the line number in ```main()``` to which control has to be transfered at some fixed memory location and then **jmp** to ```func()```. Once ```func()``` is done, it accesses that fixed memory location, gets the line number and **jmp**s to that line. We are basically storing the **return line number**.

Now, lets take a slightly complex example.

</p>

```c
------------------------------------------------------------------------code2.c
  1 void func3()
  2 {
  3         return;
  4 }
  5 
  6 void func2()
  7 {   
  8         func3();
  9         return;
 10 }
 11 
 12 void func1()
 13 {   
 14         func2();
 15         return;
 16 }
 17 
 18 
 19 int main()
 20 {   
 21         func1();
 22         return 0;
 23 }
-------------------------------------------------------------------------------
```

<p>

The program is simple to understand. There is a series of function calls and returns. 

1. ```main()``` calls ```func1()```.
2. ```func1()``` calls ```func2()```.
3. ```func2()``` calls ```func3()```.
4. ```func3()``` executes and returns back to ```func2()```.
5. ```func2()``` returns back to ```func1()```.
6. ```func1()``` returns back to ```main()```.

We need to come up with a mechanism to make this work. Try observing how the control flows. One method is to extend our previous mechanism. Have 3 fixed memory locations and store all the return line numbers. When ```main()``` calls ```func1()```, it stores line number 22 in the first fixed location. When ```func1()``` calls ```func2()```, it stores the line number 15 in the second fixed location. When ```func2()``` calls ```func3()```, it stores the line number 9 in the third fixed location. How will a function know in which fixed location it needs to store the return line number? When the return-streak starts, how will ```func3()``` know it needs to return to the line stored in the third fixed location? You can see that this mechanism is not really flexible. When there are hundreds of calls happening (which is a very normal), the caller and callee are forced to remember the fixed locations - all this is a lot of runtime overhead.

We need to come up with a mechanism which is flexible and has low runtime overhead. Which data structure is suitable for the job? I want you to try all the data structures you know of - stack, queue, list etc., Which one works?

You can see that the stack data structure fits like magic in our case. Say the stack is empty when ```main()``` is called.

</p>

```
-------------------------------------------------------------------return-stack
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
-------------------------------------------------------------------------------
```

<p>

Now, before ```main()``` **jmp**s to ```func1()```, it pushes the return line number onto the stack - it pushes the number 22 and then **jmp**s to ```func1()```. Now, the stack looks like this.

</p>

```
-------------------------------------------------------------------return-stack
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
             TOP--->|        22         |
-------------------------------------------------------------------------------
```

<p>

```func1()``` pushes its return line number (line 15) before **jmp**ing to ```func2()```. The stack looks like the following.

</p>

```
-------------------------------------------------------------------return-stack
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
             TOP--->|        15         |
                    |        22         |
-------------------------------------------------------------------------------
```

<p>

Now, control is in ```func2()```. Before it **jmp**s to ```func3()```, it needs to push the return line number - line number 9 onto stack.

</p>

```
-------------------------------------------------------------------return-stack
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
             TOP--->|         9         |
                    |        15         |
                    |        22         |
-------------------------------------------------------------------------------
```

<p>

Now, we are inside of ```func3()```. Now, we need to return back to line number 9 which is part of ```func2()```. All ```func3()``` has to do is to look at the top of the stack. It reads it, pops it off of the stack and **jmp**s to that return line. So, control is transfered to line number 9 - which is part of ```func2()```. The return-stack looks like this now.

</p>

```
-------------------------------------------------------------------return-stack
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
             TOP--->|        15         |
                    |        22         |
-------------------------------------------------------------------------------
```

<p>

Once ```func2()``` is done executing, it looks at the top of stack, reads it, pops it and **jmp**s to that return line. Control is transfered to line number 15 which is part of ```func1()```. The following is the return-stack.

</p>

```
-------------------------------------------------------------------return-stack
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
             TOP--->|        22         |
-------------------------------------------------------------------------------
```

<p>

Now, we are in ```func1()```. It looks at the top of stack and jumps back to ```main()```'s line number 22. The stack is back being empty.

</p>

```
-------------------------------------------------------------------return-stack
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
                    |                   |
-------------------------------------------------------------------------------
```

<p>

That worked fine. Having a stack makes control flow a lot easier. I am assuming this is why a **runtime stack** is present for every process. 

Let us step into a more realistic scenario. In real, we don't deal with line numbers. We deal with addresses and assembly instructions. How will this mechanism look like at the assembly level? It is similar. Instead of pushing the next line number, you push the **address of the next instruction** and later **jmp** to that callee function. Once the callee is done executing, it looks at the top of the stack, pops it and **jmp**s to the instruction present at that address. The **address of the next instruction** is the **return address**. But how exactly do we get the address of the next instruction? We get it from the **rip** register. The Instruction Pointer register always points to the next instruction.

To make things easier, the x64 architecture has two instructions - **call** and **ret**. The **call** instruction does two things. It **push**es the address of the next instruction(the return address) onto the runtime stack and **jmp**s to the callee function. The **call** instruction is a combination of two other instructions.

</p>

```asm
---------------------------------------------------------------------------call
push rip (which always points to the next instruction)
jmp func
-------------------------------------------------------------------------------
```

<p>

Once the callee is done executing, it looks at the stack. It needs to **pop** that address and then **jmp** to that address. The **ret** instruction does exactly that. It is a combination of two other instructions.

</p>

```
----------------------------------------------------------------------------ret
pop rip 
jmp rip
-------------------------------------------------------------------------------
```

<p>

We will see **call** and **ret** in action later in the chapter when we do some practicals.

That is how the control flow part of function calls work. A runtime stack is used to push and pop return addresses. We still need to see how arguments are passed to the callee function, how a callee function returns a value and more. We will explore those in later parts of the chapter.

</p>

## 4.3 What is a StackFrame? { #Section4.3 }

<p>

Functions use different types of variables.

1. All the global, extern and static variables lie in the data segment. These variables are active and present throughout the lifetime of the program. We don't have to manage their lifetimes. They come to life when the program comes to life and dies when the program dies.

2. Then comes dynamically allocated memory - the memory allocated on heap by malloc, calloc or realloc. Lifetime of this memory is completely in the hands of the programmer. We free it when we no longer need it.

3. Another type of variable is the **local variable**. They are called local for a reason. They are local to a function. They have meaning only when the function they belong to is executing. Once that function is done running, the memory allocated to these local variables need to be deallocated. They are also called function's **private variables**. Its lifetime is the lifetime of the function. With such a characteristic, which part of the virtual address space should these variables belong to? Should they be allocated in heap or data or where? How do we control the lifetimes of these variables?

In this section, we will be exploring all the above questions related to local variables.

Let us take a program to explore.

</p>

```c
------------------------------------------------------------------------code3.c
  1 void func()
  2 {
  3         short int x = 5;
  4         int y = 10;
  5         long int z = 20;
  6     
  7         return;
  8 }
  9 
 10 int main()
 11 {
 12         func();
 13         return 0;
 14 }
-------------------------------------------------------------------------------
```

<p>

This is a simple program. ```main()``` calls a function ```func()``` which defines 3 **local variables** x, y and z. Again, remember their characteristic - they come to life when ```func()``` starts its execution and dies when ```func()``` returns back its control to ```main()```. Think like a compiler. What type of code should it emit to bring about this behavior?

Let us start with a simple idea. Let all the local variables be stored in heap. How will ```func()```'s code look like? Take some time and think about it.

1. The compiler knows the total amount of memory needed for all the local variables in a function In our case, ```func()``` requires (2 + 4 + 8) = 14 bytes. Let us implement ```func()``` at assembly level. Let us start with allocating memory for local variables on heap.

</p>

```asm
----------------------------------------------------------------------code3.asm
func:
       call malloc(14)      ; Allocation
-------------------------------------------------------------------------------
```

<p>

2. ```malloc``` returns a virtual address pointing to those 14 bytes of memory. At assembly level, let a function call return a value through the register **rax**. After ```malloc``` is called, on success **rax** has the address pointing to the 14 bytes. Let us load that address into register **r15**.

</p>

```asm
----------------------------------------------------------------------code3.asm
func:
       call malloc(14)      ; Allocation
       mov r15, rax         ; r15 now points to the heap memory
-------------------------------------------------------------------------------
```

<p>

3. Now, first 2 bytes is the variable **x**. It needs to be initialized with the value 5. Lets do that.

</p>

```asm
----------------------------------------------------------------------code3.asm
func:
       call malloc(14)      ; Allocation
       mov r15, rax         ; r15 now points to the heap memory
       mov word[r15], 5     ; short int x = 5;
-------------------------------------------------------------------------------
```

<p>

4. The next 4 bytes is the variable **y** which needs to be initialized with the value 10.

</p>

```asm
----------------------------------------------------------------------code3.asm
func:
       call malloc(14)      ; Allocation
       mov r15, rax         ; r15 now points to the heap memory
       mov word[r15], 5     ; short int x = 5;
       mov dword[r15 + 2], 10      ; int y = 10;
-------------------------------------------------------------------------------
```

<p>

5. The next 8 bytes is the variable **z** which needs to be initialized with the value 20.

</p>

```asm
----------------------------------------------------------------------code3.asm
func:
       call malloc(14)      ; Allocation
       mov r15, rax         ; r15 now points to the heap memory
       mov word[r15], 5     ; short int x = 5;
       mov dword[r15 + 2], 10      ; int y = 10;
       mov qword[r15 + 6], 20      ; long int z = 20;
-------------------------------------------------------------------------------
```

<p>

6. The local variables are now initialized in heap in the following manner.

</p>

```
-------------------------------------------------------------------------------
r15 -> | 00 | 05 | 00 | 00 | 00 | 0a | 00 | 00 | 00 | 00 | 00 | 00 | 00 | 14 |

       |----x----|---------y---------|-------------------z-------------------|
-------------------------------------------------------------------------------
```

<p>

7. This function is a simple one. It does not have any other code. It has code only to initialize the variables. Once they are initialized, they can be used without any problem.

8. Once the function is over, before it returns we need to deallocate it. We can use ```free``` to free it up.

</p>

```asm
----------------------------------------------------------------------code3.asm
func:
       call malloc(14)      ; Allocation
       mov r15, rax         ; r15 now points to the heap memory
       mov word[r15], 5     ; short int x = 5;
       mov dword[r15 + 2], 10      ; int y = 10;
       mov qword[r15 + 6], 20      ; long int z = 20;
       ; Body

       free(r15)            ; Deallocation
-------------------------------------------------------------------------------
```

<p>

9. We are done cleaning up, lets return.

</p>

```asm
----------------------------------------------------------------------code3.asm
func:
       call malloc(14)      ; Allocation
       mov r15, rax         ; r15 now points to the heap memory
       mov word[r15], 5     ; short int x = 5;
       mov dword[r15 + 2], 10      ; int y = 10;
       mov qword[r15 + 6], 20      ; long int z = 20;
       ; Body

       free(r15)            ; Deallocation
       ret                  ; Return back to the caller
-------------------------------------------------------------------------------
```

<p>

Note that the programmer does not have to explicitly call ```malloc``` and ```free```. That is code added automatically by the compiler. If this needs to be implemented, the compiler needs to emit code which also handles the case when ```malloc``` fails.

Now, let us take another approach. Let us see if the runtime stack can be used to store the local variables.

So far, we have used the runtime stack only for storing return addresses. What can we do to to accomodate the local variables?

Every architecture offers a special register - the **Stack Pointer** which always points to the top of the stack. Assume that we are in a state where ```main()``` just **call**ed ```func()``` and we are inside ```func()``` now. The stack looks like this.

</p>

```
-------------------------------------------------------------------return-stack
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
              SP--->|    Return Address     |
-------------------------------------------------------------------------------
```

<p>

We know that the stack has a width. Assume the width is 8 bytes. Any changes we do to the stack pointer, it should adhere to the 8-byte boundary. Even though we need only 14 bytes, we will allocate 16 bytes. Let us start implementing ```func()```.

</p>

```asm
----------------------------------------------------------------------code3.asm
func:
       add sp, 16           ; Allocation
-------------------------------------------------------------------------------
```

<p>

In a normal stack data structure you implement or you see in any library, you can't really change the stack pointer directly. You can only push and pop objects - that will indirectly change the stack pointer. Even here, we could do the same. We can push 0 twice to allocate 16 bytes and to move the stack pointer up. But we can do simple arithmetic on the stack pointer in most of the architectures. So, we will use add/sub to adjust the stack pointer whenever needed.

The stack now looks like this.

</p>

```
--------------------------------------------------------------------------stack
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
              SP--->|  |  |  |  |  |  |  |  |
              SP-8->|  |  |  |  |  |  |  |  |
                    |    Return Address     |
-------------------------------------------------------------------------------
```

<p>

Now, let us initialize the 3 variables. The first free byte is pointed by **sp-8**. Let the 2 bytes pointed by **sp-8** be variable x. Let us initialize it to 5.

</p>

```asm
----------------------------------------------------------------------code3.asm
func:
       add sp, 16           ; Allocation
       mov word[sp-8], 5    ; short int x = 5;
-------------------------------------------------------------------------------
```

<p>

The stack looks like this.

</p>

```
--------------------------------------------------------------------------stack
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
              SP--->|  |  |  |  |  |  |  |  |
              SP-8->|00|05|  |  |  |  |  |  |
                    |    Return Address     |
-------------------------------------------------------------------------------
```

<p>

The next four bytes is belong to **y**. Let us init it.

</p>

```asm
----------------------------------------------------------------------code3.asm
func:
       add sp, 16           ; Allocation
       mov word[sp-8], 5    ; short int x = 5;
       mov dword[sp-6], 10  ; int y = 10;
-------------------------------------------------------------------------------
```

<p>

The stack looks like this.

</p>

```
--------------------------------------------------------------------------stack
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
              SP--->|  |  |  |  |  |  |  |  |
              SP-8->|00|05|00|00|00|0a|  |  |
                    |    Return Address     |
-------------------------------------------------------------------------------
```

<p>

Where do we place the variable z? It can be placed at the 8 bytes pointed by **sp**.

</p>

```asm
----------------------------------------------------------------------code3.asm
func:
       add sp, 16           ; Allocation
       mov word[sp-8], 5    ; short int x = 5;
       mov dword[sp-6], 10  ; int y = 10;
       mov qword[sp], 20    ; long int z = 20;
-------------------------------------------------------------------------------
```

<p>

The stack looks like this now.

</p>

```
--------------------------------------------------------------------------stack
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
              SP--->|00|00|00|00|00|00|00|14|
              SP-8->|00|05|00|00|00|0a|  |  |
                    |    Return Address     |
-------------------------------------------------------------------------------
```

<p>

There are 2 unused bytes. They will always be present, irrespective of the way the 3 variables are arranged in those 16 bytes.

Now comes the body of the function.

</p>

```asm
----------------------------------------------------------------------code3.asm
func: 
       add sp, 16           ; Allocation
       mov word[sp-8], 5    ; short int x = 5;
       mov dword[sp-6], 10  ; int y = 10;
       mov qword[sp], 20    ; long int z = 20;
       ; Body of the function
-------------------------------------------------------------------------------
```

<p>

Once the body is done, we need to deallocate the 16 bytes. How do we deallocate it? It is fairly simple. We simply shift the stack pointer back by 16 bytes. It is one assembly instruction.

</p>

```asm
----------------------------------------------------------------------code3.asm
func:
       add sp, 16           ; Allocation
       mov word[sp-8], 5    ; short int x = 5;
       mov dword[sp-6], 10  ; int y = 10;
       mov qword[sp], 20    ; long int z = 20;
       ; Body of the function
       sub sp, 16           ; Deallocation
-------------------------------------------------------------------------------
```

<p>

The stack looks like this.

</p>

```
--------------------------------------------------------------------------stack
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |00|00|00|00|00|00|00|14|
                    |00|05|00|00|00|0a|  |  |
              SP--->|    Return Address     |
-------------------------------------------------------------------------------
```

<p>

Note that the data is still there, but the memory does not belong to the function anymore. The stack pointer now points to the return address. We can simply **ret** back to the caller.

</p>

```asm
----------------------------------------------------------------------code3.asm
func:
       add sp, 16           ; Allocation
       mov word[sp-8], 5    ; short int x = 5;
       mov dword[sp-6], 10  ; int y = 10;
       mov qword[sp], 20    ; long int z = 20;
       ; Body of the function
       sub sp, 16           ; Deallocation
       ret                  ; Return back to the caller
-------------------------------------------------------------------------------
```

<p>

This shows that even the runtime stack can be used to store the local variables. I want you to stop here and compare the stack and heap techniques to store the local variables.

Consider the following stack state.

</p>

```
--------------------------------------------------------------------------stack
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
              SP--->|00|00|00|00|00|00|00|14|
              SP-8->|00|05|00|00|00|0a|  |  |
                    |    Return Address     |
-------------------------------------------------------------------------------
```

<p>

Note that the 24 bytes starting from the byte pointed by **sp** till the byte pointed by **sp-23** - they are all private to the function ```func()```. These 24 bytes form the **stack frame** of ```func()```. A Stack frame is simply the part of the runtime stack which belongs to a particular function till it is alive and running. Once the function is done running, that stack frame is descructed(or deallocated).

To understand this better, let us consider a slightly complex example.

</p>

```c
------------------------------------------------------------------------code4.c
  1 void func2()
  2 {   
  3         int a = 25;
  4         unsigned int b = 35;
  5     
  6         return;
  7 }
  8 
  9 
 10 void func1()
 11 {
 12         short int x = 5;
 13         int y = 10;
 14         long int z = 20;
 15     
 16         return;
 17 }
 18 
 19 int main()
 20 {
 21         func1();
 22         return 0;
 23 }
-------------------------------------------------------------------------------
```

<p>

Let us go through the above program - write assembly code and make sure we understand how stack is used to store local variables.

```main()```'s code would be pretty simple. It simply calls ```func1()``` and returns. The following is ```main()```.

</p>

```asm
----------------------------------------------------------------------code4.asm
main:
       call func1           ; func1();
       ret                  ; return
-------------------------------------------------------------------------------
```

<p>

When ```main()``` calls ```func1()```, it pushes the address of the next instruction(```ret``` here) and then **jmp**s to the label ```func1()```. The stack would look like this once control is transfered to ```func1()```.

</p>

```
--------------------------------------------------------------------------stack
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
              SP--->|    ReturnAddress1     |
-------------------------------------------------------------------------------
```

<p>

Now we are in ```func1()```. It has 3 local variables x, y and z and the total memory required is (2 + 4 + 8) = 14 bytes. This means, we will have to allocate 16 bytes and lay the variables the way we did in the previous example. The assembly code for ```func1()``` looks like this.

</p>

```asm
----------------------------------------------------------------------code4.asm
func1:
       add sp, 16           ; Allocation
       mov word[sp-8], 5    ; short int x = 5;
       mov dword[sp-6], 10  ; int y = 10;
       mov qword[sp], 20    ; long int z = 20;

main:
       call func1           ; func1();
       ret                  ; return
-------------------------------------------------------------------------------
```

<p>

And the stack would look like this if those instructions were executed.

</p>

```
--------------------------------------------------------------------------stack
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
              SP--->|00|00|00|00|00|00|00|14|    -
              SP-8->|00|05|00|00|00|0a|  |  |    | func1's 24-byte stack-frame
                    |    ReturnAddress1     |    -
-------------------------------------------------------------------------------
```

<p>

Look at the above stack state very carefully. ```func1()```'s **stack-frame** is at the top of the stack. The size of that stack-frame is 24 bytes - 16 bytes for local variables and 8 bytes for ReturnAddress1.

Now is the time to call ```func2()``` from inside of ```func1()```. Once that call is done, ```func1()``` returns.

</p>

```asm
----------------------------------------------------------------------code4.asm
func1:
       add sp, 16           ; Allocation
       mov word[sp-8], 5    ; short int x = 5;
       mov dword[sp-6], 10  ; int y = 10;
       mov qword[sp], 20    ; long int z = 20;
       
       call func2           ; func2();
       

main:
       call func1           ; func1();
       ret                  ; return
-------------------------------------------------------------------------------
```

<p>

When the ```call func2``` instruction is executed, the address of the next instruction is pushed onto the stack and control is transfered to ```func2()```. After the control is in ```func2()```, the stack looks like the following.

</p>

```
--------------------------------------------------------------------------stack
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
              SP--->|    ReturnAddress2     |    -> func2's new stack-frame
                    |00|00|00|00|00|00|00|14|    -
                    |00|05|00|00|00|0a|  |  |    | func1's 24-byte stack-frame
                    |    ReturnAddress1     |    -
-------------------------------------------------------------------------------
```

<p>

We are now in ```func2()```. It has 2 variables a and b of types int and unsigned int. Each of these variable needs 4 bytes => we need a total of 8 bytes. The following is the initialization code for ```func2()```.

</p>

```asm
----------------------------------------------------------------------code4.asm
func2:
       add sp, 8            ; Allocation
       mov dword[sp], 25    ; int a = 25;
       mov dword[sp+4], 35  ; unsigned int b = 35;

func1:
       add sp, 16           ; Allocation
       mov word[sp-8], 5    ; short int x = 5;
       mov dword[sp-6], 10  ; int y = 10;
       mov qword[sp], 20    ; long int z = 20;
       
       call func2           ; func2();
       
       
main:
       call func1           ; func1();
       ret                  ; return
-------------------------------------------------------------------------------
```

<p>

The stack looks like this now.

</p>

```
--------------------------------------------------------------------------stack
                    |  |  |  |  |  |  |  |  |
              SP--->|00|00|00|19|00|00|00|23|    -
                    |    ReturnAddress2     |    | func2's 16-byte stack-frame
                    |00|00|00|00|00|00|00|14|    -
                    |00|05|00|00|00|0a|  |  |    | func1's 24-byte stack-frame
                    |    ReturnAddress1     |    -
-------------------------------------------------------------------------------
```

<p>

Look at the above stack state. The 16 bytes pointed by the stack pointer make up ```func2()```'s **stack-frame**. Because ```func2()``` is currently running, it's stack-frame is at the top of the stack. Right below it's stack-frame is ```func1()``` stack-frame. 

In general, the runtime stack might have a lot of stack-frames. But the stack-frame which is at the top of the stack belongs to the function currently active and running.

```func2()``` does not do more. It initializes variables and then returns. So, now we need to write the clean-up code which deallocates the stack memory. It is just a line - we need to reduce the stack pointer by 8 bytes. The code looks like this now.

</p>

```asm
----------------------------------------------------------------------code4.asm
func2:
       add sp, 8            ; Allocation
       mov dword[sp], 25    ; int a = 25;
       mov dword[sp+4], 35  ; unsigned int b = 35;
       sub sp, 8            ; Deallocation

func1:
       add sp, 16           ; Allocation
       mov word[sp-8], 5    ; short int x = 5;
       mov dword[sp-6], 10  ; int y = 10;
       mov qword[sp], 20    ; long int z = 20;
       
       call func2           ; func2();
       
       
main:
       call func1
       ret
-------------------------------------------------------------------------------
```

<p>

Now that the clean-up is done, the stack looks like this.

</p>

```
--------------------------------------------------------------------------stack
                    |  |  |  |  |  |  |  |  |
                    |00|00|00|19|00|00|00|23|
              SP--->|    ReturnAddress2     |
                    |00|00|00|00|00|00|00|14|
                    |00|05|00|00|00|0a|  |  |
                    |    ReturnAddress1     |
-------------------------------------------------------------------------------
```

<p>

The data is still there, but it does not matter. Now, let us return using the **ret** instruction. It pops the ReturnAddress2 at the top of stack and then **jmp**s to it. The following is the code.

</p>

```asm
----------------------------------------------------------------------code4.asm
func2:
       add sp, 8            ; Allocation
       mov dword[sp], 25    ; int a = 25;
       mov dword[sp+4], 35  ; unsigned int b = 35;
       sub sp, 8            ; Deallocation
       ret                  ; Return back to the caller

func1:
       add sp, 16           ; Allocation
       mov word[sp-8], 5    ; short int x = 5;
       mov dword[sp-6], 10  ; int y = 10;
       mov qword[sp], 20    ; long int z = 20;
       
       call func2           ; func2();
       
       
main:
       call func1
       ret
-------------------------------------------------------------------------------
```

<p>

Once that is done, we are back in ```func1()``` and the stack looks like this.

</p>

```
--------------------------------------------------------------------------stack
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
              SP--->|00|00|00|00|00|00|00|14|
                    |00|05|00|00|00|0a|  |  |
                    |    ReturnAddress1     |
-------------------------------------------------------------------------------
```

<p>

Look how beautifully it fits. Compare the above stack state with the one before ```func2()``` is called. They are identical. Its as if ```func2()``` was never called. Now, we need to write the clean-up instructions for ```func1()```. We need to deallocate 16 bytes. The following does it.

</p>

```asm
----------------------------------------------------------------------code4.asm
func2:
       add sp, 8            ; Allocation
       mov dword[sp], 25    ; int a = 25;
       mov dword[sp+4], 35  ; unsigned int b = 35;
       sub sp, 8            ; Deallocation
       ret                  ; Return back to the caller

func1:
       add sp, 16           ; Allocation
       mov word[sp-8], 5    ; short int x = 5;
       mov dword[sp-6], 10  ; int y = 10;
       mov qword[sp], 20    ; long int z = 20;
       
       call func2           ; func2();

       sub sp, 16           ; Deallocation
       
       
main:
       call func1
       ret
-------------------------------------------------------------------------------
```

<p>

Once that deallocation instruction(```sub sp, 16```) is executed, the stack looks like the following.

</p>

```
--------------------------------------------------------------------------stack
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
                    |  |  |  |  |  |  |  |  |
              SP--->|    ReturnAddress1     |
-------------------------------------------------------------------------------
```

<p>

Finally, we **ret** back to ```main()```. That completes ```func1()```'s code.

</p>

```asm
----------------------------------------------------------------------code4.asm
func2:
       add sp, 8            ; Allocation
       mov dword[sp], 25    ; int a = 25;
       mov dword[sp+4], 35  ; unsigned int b = 35;
       sub sp, 8            ; Deallocation
       ret                  ; Return back to the caller

func1:
       add sp, 16           ; Allocation
       mov word[sp-8], 5    ; short int x = 5;
       mov dword[sp-6], 10  ; int y = 10;
       mov qword[sp], 20    ; long int z = 20;
       
       call func2           ; func2();

       sub sp, 16           ; Deallocation
       ret                  ; Return back to the caller
       
main:
       call func1
       ret
-------------------------------------------------------------------------------
```

<p>

The stack is back to being empty - the way it was before ```func1()``` was called in ```main()```.

There are a couple of observations.

1. Whatever function is running right now, it's stack-frame will always be at the top of the runtime stack.
2. Stack-Frame is constructed when a function is called by moving the stack pointer by suitable amount. It is desctructed when the function execution comes to an end by moving the stack pointer back to its original position.
3. Here, I have assumed that the ReturnAddress is part of the **callee**'s stack-frame. I think it is a matter of definition.

At this point, I hope you have an idea of how function calls work, how stack-frames are constructed and destructed for each function. Everything there was pretty generic. We did not deal into the nitty-gritty details of the x64 architecture. The next section will be about that - how exactly does function calls, stack-frames work in x64?

</p>

## 4.4 Function calls and stack-frames in x64 { #Section4.4 }

<p>

Function calls in x64 happens exactly the way we discussed in [Section4.2](#Section4.2). The runtime stack is used to make function calls work. In this section, we will mainly discuss about stack-frames in x64 architecture.

In the [previous section](#Section4.3), we discussed the construction and destruction of stack-frames as and when control is transfered into and out of a function respectively. There are certain things specific to x64 which we will discuss in this section.

</p>

### 4.4.1 Growth direction of the runtime stack

<p>

Whenever a new stack-frame is constructed, the runtime stack is growing. In the previous section, all the stack diagrams indicated that the stack grows upwards - like any other stack. If you take a closer look at the memory layout of a process presented in [previous chapter](#Chapter3)'s [Section3.5](#Section3.5), the runtime stack actually grows downwards - towards smaller virtual addresses. This can be visualized in two ways.

</p>

```
-------------------------------------------------------------------stack-growth
                                   Higher addresses
                     |                    |
                     |                    |
                     |                    |
                     |                    |
                     |                    |
                     |                    |
                               ||
                               \/    
                                   Lower Addresses
-------------------------------------------------------------------------------
```

<p>

The above is the straight-forward way - it shows the runtime stack the way it is. It grows towards lower addresses.

Because we are accustomed to the stack growing upwards, we can simply show the stack upside-down like the following.
</p>


```
-------------------------------------------------------------------stack-growth
                                   Lower addresses
                              /\
                              ||
                     |                    |
                     |                    |
                     |                    |
                     |                    |
                     |                    |
                     |                    |
                                   Higher addresses
-------------------------------------------------------------------------------
```

<p>

We will be using the second diagram throughout the chapter unless specified otherwise.

In the previous section, to allocate memory on the runtime stack, we did this - ```add sp, 16```. This works only when the stack grows towards higher addresses. In our case, the stack grows towards lower addresses. So, to allocate memory on the runtime stack, we should not add an immediate value. We should instead subtract an immediate - ```sub sp, 16``` will do. Consider the following stack state.

</p>

```
-------------------------------------------------------------------stack-growth
                                   Lower addresses
                     |                    |
                     |                    |
                     |                    |
              SP---->|  ReturnAddress1    |
                     |                    |
                     |                    |
                                   Higher addresses
-------------------------------------------------------------------------------
```

<p>

Now, we need to allocate 16 bytes on the stack. Executing ```add sp, 16``` will change the stack to the following.

</p>

```
-------------------------------------------------------------------stack-growth
                                   Lower addresses
                     |                    |
                     |                    |
                     |                    |
                     |  ReturnAddress1    |
                     |                    |
              SP---->|                    |
                                   Higher addresses
-------------------------------------------------------------------------------
```

<p>

This actually messed up the stack-state. When the stack is growing downwards(towards lower addresses), one should subtract an immediate value to allocate more memory - ```sub sp, 16```. The following is the result.

</p>


```
-------------------------------------------------------------------stack-growth
                                   Lower addresses
                     |                    |
              SP---->|                    |
                     |                    |
                     |  ReturnAddress1    |
                     |                    |
                     |                    |
                                   Higher addresses
-------------------------------------------------------------------------------
```

<p>

I can understand that this is a little confusing. You might have questions like why is the stack even growing downwards? Why didn't they design it to grow upwards and make everything simple and straight-forward? We will reason out these questions later in the chapter. For now, I want you to remember that the stack in Intel/AMD processors always grows downwards - towards lower addresses.

### 4.4.2 Data Storage and encoding

<p>

Consider the following function.

</p>

```asm
--------------------------------------------------------------------------func2
func2:
       sub rsp, 8              ; Allocation
       mov dword[rsp], 25      ; int a = 25;
       mov dword[rsp+4], 35    ; unsigned int b = 35;

       ; No body

       add rsp, 8              ; Deallocation
       ret
-------------------------------------------------------------------------------
```
<p>

Say some function called ```func2()``` and the first instruction ```sub rsp, 8``` is already executed. The stack looks like the following.

</p>

```
--------------------------------------------------------------------------stack
                                   Lower addresses
                     |  |  |  |  |  |  |  |  |
                     |  |  |  |  |  |  |  |  |
                     |  |  |  |  |  |  |  |  |
              rsp--->|  |  |  |  |  |  |  |  |
              rsp+8->|    ReturnAddress1     |
                     |  |  |  |  |  |  |  |  |
                                   Higher addresses
-------------------------------------------------------------------------------
```

<p>

Executing the instruction ```mov dword[rsp], 25``` will change the stack to the following.

</p>

```
--------------------------------------------------------------------------stack
                                   Lower addresses
                     |  |  |  |  |  |  |  |  |
                     |  |  |  |  |  |  |  |  |
                     |  |  |  |  |  |  |  |  |
              rsp--->|00|00|00|19|  |  |  |  |
              rsp+8->|    ReturnAddress1     |
                     |  |  |  |  |  |  |  |  |
                                   Higher addresses
-------------------------------------------------------------------------------
```

<p>

The hexadecimal equivalent of 25 is 0x00000019. This is the natural way to store a number. **rsp** points to the Most Significant byte of the number 0x00000019. What if I tell you that this is not the way in which the number is stored in real memory? It is stored as shown in the following stack diagram.

</p>

```
--------------------------------------------------------------------------stack
                                   Lower addresses
                     |  |  |  |  |  |  |  |  |
                     |  |  |  |  |  |  |  |  |
                     |  |  |  |  |  |  |  |  |
              rsp--->|19|00|00|00|  |  |  |  |
              rsp+8->|    ReturnAddress1     |
                     |  |  |  |  |  |  |  |  |
                                   Higher addresses
-------------------------------------------------------------------------------
```

<p>

The number is stored in a reverse manner. **rsp** points to the Least Significant byte of the number. This is once again a bit confusing. Why on earth did we decide to do everything in the reverse manner?





</p>





With that in mind, lets take *code4.c* and trace the assembly code and stack for it.

```c
-------------------------------------------------------------------------code4.c
  1 void func2()
  2 {
  3         int a = 25;
  4         unsigned int b = 35;
  5 
  6         return;
  7 }
  8 
  9 
 10 void func1()
 11 {
 12         short int x = 5;
 13         int y = 10;
 14         long int z = 20;
 15     
 16         return;
 17 }
 18 
 19 int main()
 20 {
 21         func1();
 22         return 0;
 23 }
-------------------------------------------------------------------------------
```

In x64, the register **rsp** is the stack pointer. From now onwards, we will be using the symbol **rsp** instead of just **sp**.

Consider the following empty runtime stack.

```
```
-------------------------------------------------------------------stack-growth
                                   Lower addresses
                     |                    |
              SP---->|                    |
                     |                    |
                     |  ReturnAddress1    |
                     |                    |
                     |                    |
                                   Higher addresses
-------------------------------------------------------------------------------
```
```





</p>

### 4.4.1 The Base Pointer

In the previous section, we discussed the entire concept of stack-frames with just the stack-pointer. The x64 architecture offers two registers: **rsp** or the Stack Pointer register and **rbp** or the Base Pointer register. **rsp** always points to the top of the stack(or the active stack-frame). **rbp** can be made to point to the base of the stack-frame. But what exactly is the base of a stack-frame?

Consider the following stack-frame.

</p>




```
--------------------------------------------------------------------------stack
                    |  |  |  |  |  |  |  |  |
              SP--->|00|00|00|19|00|00|00|23|    -
                    |    ReturnAddress2     |    | func2's 16-byte stack-frame
                    |00|00|00|00|00|00|00|14|    -
                    |00|05|00|00|00|0a|  |  |    | func1's 24-byte stack-frame
                    |    ReturnAddress1     |    -
-------------------------------------------------------------------------------
```

<p>


</p>


## Some resources

1. Something on 16-byte alignment - https://sourceforge.net/p/fbc/bugs/659/
2. https://raw.githubusercontent.com/wiki/hjl-tools/x86-psABI/x86-64-psABI-1.0.pdf - Latest AMD64 ABI copy
3. https://stackoverflow.com/questions/49391001/why-does-system-v-amd64-abi-mandate-a-16-byte-stack-alignment - good discussion on why stack is 16-byte aligned.
4. https://patchwork.kernel.org/patch/9507697/ - another discussion on why stack should be 16-byte aligned.
5. 


----------------------------

The stack pointer is pointing to the top of the stack. You can observe that the top of the stack is nothing but top of ```func2()```'s stack-frame. From this, we can tell that the **rsp** always points to the top of the **active stack-frame** - the stack-frame of the function currently under execution.
