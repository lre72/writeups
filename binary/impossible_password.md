# RE: Impossible Password
Challenge: Impossible Password  
CTF name: Hack the Box  
Date attempted: 22.04.2024   

Difficulty: Easy  
[Original link](https://app.hackthebox.com/challenges/Impossible%2520Password)

<hr>

### Understanding the Program
I start by running the file to understand what it does.
<br><br>
![image](https://github.com/lre72/writeups/assets/136733873/5ee3ff3b-65d4-4b83-8622-e184043994ce)  

1. Asterisk printed, signals for user input.
2. User input is repeated back in square brackets [], with a cap of 20 characters.

It looks simple enough. I proceed to `strings impossible_password.bin` as usual, and find something that looks important.  
Apparently, there is a string called `SuperSeKretKey`.  

Unfortunately, I don't know what to make of it yet. So, it's time for static analysis on Ghidra.

<br><br>

### Static Analysis on Ghidra
I open impossible_password.bin on Ghidra and select the entry function.  
In the entry function, I find `FUN_0040085d`, which is the main function.
<br><br>
![image](https://github.com/lre72/writeups/assets/136733873/c7b23c38-a04f-490a-a77f-de937f038b2b)  
This program isn't actually very long, it just has a lot of variables.  

Anyway, from analysis, we can see that the program starts by **printing an asterisk**. This matches what we observed.  
The program intakes user input with `__isoc99_scanf` and stores it in `local_28`.  

Then, it repeats local_28 (the user input) back in square brackets, once again as we observed.  
A `strcmp` is called between local_28 and local_10, which happens to be `SuperSeKretKey`.  

<br><br>

### Past the First Password
Okay, let's try that out.  
In the terminal, we run impossible_password.bin and when the asterisk appears, we type in `SuperSeKretKey`.  

It works and we find the next phase of the program.  
<br><br>
![image](https://github.com/lre72/writeups/assets/136733873/bde932a0-7582-4704-8b61-74d2ead45f0d)  

According to the Ghidra analysis, when the double asterisks ** are printed, **user input is taken in a second time**.  
Based on the code, the flag is printed after this second phase, where user input is compared with `__s2`.

So, what is __s2?  

<br><br>
![image](https://github.com/lre72/writeups/assets/136733873/460c45ee-4ceb-4e84-a82b-271e8c54aa05)
<br><br>

This is the function that `__s2` leads to.  
From here, you can tell that **__s2 is randomly generated every time**.  

It's going to be difficult (or impossible?) to get the password like this, so I'll look at the disassembled code instead of the
decompiled code and see if I can use binary patching.   

![image](https://github.com/lre72/writeups/assets/136733873/05330150-5ed7-46ba-a055-058bf2a193b4)  

We will examine the registers instead.  
⭐This is a useful guide for registers: [x84-64 Architecture Guide](http://6.s081.scripts.mit.edu/sp18/x86-64-architecture-guide.html)
* User input seems to be going to `RAX`.
* The other registers involved are:
    * `RSI`: used to pass 2nd argument to functions
    * `EDI`: call-preserved register
    * `EAX`: accumulator register
    * `RDX`: used to pass 3rd argument to functions
    * `RDI`: used to pass 1st argument to functions
    * `RAX` itself: temp register; return value

Also, `RAX` mainly appears together with `RSI`, `RDX`, and `RDI`.
<br><br>

### Examining Registers with x/s
We'll be using gdb for this.  
I run the program in gdb, and then set a breakpoint at `0x0040092f`, which is the line with the `printf` function for the double asterisks ** in the second phase.  

Note that I used the address here, which might not be possible if the program is PIE (Position Independent Executable), where everytime the program is run it loads
into a different memory address. [See: PIE | Binary Exploitation](https://ir0nstone.gitbook.io/notes/types/stack/pie)  

Spoiler: That didn't work, by the way.  

### Re-evaluating approach
I got an error that I couldn't access the memory address there, and checking `info registers` showed that the main registers I'm interested in are all `0x0` (null)
which is not right.  

So, I decided to set a breakpoint at the line before the `strcmp` in the second phase, at address `0x0040095e`.  
At first, I didn't want to since I thought "What if I need to change my input?" but then I realised it was a stupid approach because I can change the registers
at any point using gdb.  

I deleted my previous breakpoints.  
```gdb
del breakpoints
b *0x0040095e
run
```

![image](https://github.com/lre72/writeups/assets/136733873/98b75fd2-09ba-47ca-9cc8-cc233a6ffa22)  

The strange string in `RDX` register looked like it could be a randomly generated code (__s2).  
Whereas the value in the `RDI` register seemed unlikely to be it.  

So I used gdb to `set $rax=$rdx`, equating the user input to the randomly generated code.  
Use gdb to `continue` the program, and it's done.  

> Continuing.  
> HTB{40b949f92b86b18}

There!  
<br><br>

### In Summary
1. Use Ghidra for static analysis.  
2. The first password is `SuperSeKretKey`, when prompted with * in the program, enter it.
3. Use gdb for binary patching.
4. Set a breakpoint before the `strcmp` for the second password (`__s2`) and user input (`local_28`).
5. (optional) Examine the registers with `x/s`, particularly `RAX` (user input) and `RSI`, `RDX`.

6. Equate registers `RAX` and `RDX`/`RSI` using `set $RAX=$RDX`.
7. Continue in gdb.
