# RE: Buggy Company
Challenge: Buggy Company  
CTF name: Whitehacks 2024  
Date attempted: 15.04.2024  

Difficulty: Easy  
[Original link](https://whitehacks.computing.smu.edu.sg/)

<hr>

### Understanding the File
I start by running the file to understand what it does.  
1. Bug is printed.
2. Asks for password.
3. Denies entry, no password inputted.

<br><br>
Next I use `strings buggy_company` to see if I can find anything of importance.  
<br><br>
![image](https://github.com/lre72/writeups/assets/136733873/2b6802bb-2c3a-406f-8c65-1eb8281bd904)

PASSPHRASE  
%02x  
ee9e5e5c8408d77e3c3c83dde3070dd8  

I saw `.gnu.hash` in the strings, and this also looks like a hash, so I put it in [Hashes.com](https://hashes.com/en/decrypt/hash) and
decode ee9e5e5c8408d77e3c3c83dde3070dd8 into 'YIPPEE'.  Unfortunately, both YIPPEE and the hash itself is rejected when I enter them
into the program, so something's still missing.  

I move on to Ghidra analysis.
<br><br>


### Static Analysis on Ghidra
I find the `entry` function, which then allows me to find `FUN_00101614`. On observation, this is the program's main function.  

![image](https://github.com/lre72/writeups/assets/136733873/f7b37998-274b-469b-ba94-d7bfcb3cb92f)
  
Since I'm not confident in identifying the local variables yet, I start by renaming the functions.  
The first function is `FUN_001012e9`, which clearly is the function for printing the bug.  

On the next line, there is `iVar1`, which equates to `FUN_00101466`.  
This function uses ptrace, and is used as a debugger check. (Running buggy_company in Terminal with gdb active causes a debugger
alert. Debugger alert had also been triggered when using strings.)  

> By analysis, we have found the first issue:  
> **debugger check**, happens AFTER bug is printed

<br><br>
### Continuing the Static Analysis
This is what we have so far (screenshot excludes variable declaration on top lines).
<br><br>

![image](https://github.com/lre72/writeups/assets/136733873/ed21b800-ef6f-475f-8203-2adeb88092b9)
<br><br>

```c
d = (uchar *)getenv("PASSPHRASE");
```
  
In this line, the program retrieves the environment variable "PASSPHRASE" and type casts it into a `uchar`, then stores it in `d`.  
Normally, the passphrase is something a program contains, but since this program takes it from an environment variable, my computer
must have "PASSPHRASE" stored on it. We keep reading to check.  

```c
  if (d == (uchar *)0x0) {
    FUN_001015d2();
  }
```

`0x0` is a hexadecimal representing the address `0`, and (uchar *)0x0 is basically a null pointer.  
So, if `d` is null, `FUN_001015d2` is executed. This function also happens to be the no loot function -- it prints the 'No loot!' message.  

This confirms our thoughts on PASSPHRASE.  
Environment variables have their **values set outside the program**, so clearly it will always be null unless we set it before running
the program.

> Next issue we find:  
> **Environment variable PASSPHRASE cannot be null**.

<br><br>

After that, I labelled some things and came to the next function.  

![image](https://github.com/lre72/writeups/assets/136733873/47447f3d-d636-4c2e-9d06-dd5d8dc3d9cc)
<br><br>

The MD5 hash function is understandable; the one under that confused me a lot.  
Because I did not want to interpret that, I asked ChatGPT and learnt that the MD5 hash function actually returns a binary array of 16 bytes.
So the loop after that is to **convert the MD5 hash binary into a hexadecimal string**, i.e. the hash format we all recognise.  
<br><br>

> To cite ChatGPT,
> To make this data readable, each byte is converted to its 2-character hexadecimal representation.
For instance, the byte 0xA3 becomes the string "a3", 0x1F becomes "1f", 0x00 becomes "00", etc.
The full MD5 hash as a binary array (16 bytes) will be converted to a 32-character hexadecimal string.  

<br><br>

### Finishing the Static Analysis
Finally, there is a `strcmp` between `local_38` and `ee9e5e5c8408d77e3c3c83dde3070dd8`, which is also the hash we found in 
`strings` in the first part of this writeup, that equates to YIPPEE.  

I'm not too sure what local_38 is, or if we can assume anything.  
But, if the `strcmp` passes, the next function results in the flag being printed.

<br><br>

### Running in Terminal
**First issue**: Since we did static analysis and don't require any significant binary patching, we don't need to
worry about the debugger check here.  
**Second issue**: We use `export PASSPHRASE=YIPPEE` to set the environment variable PASSPHRASE to the decoded hash.  
<br><br>
![image](https://github.com/lre72/writeups/assets/136733873/2e145257-3038-4bd9-bb43-139b638d3890)
<br><br>

This is it!  
> In the middle of the darkness, a hoarding bug skitters up to you. 'Password?'  
> 'One of us!' The bug yelps, and hands you the loot.  
> WH2024{J!0N_TH3_8UG_M4F!4}

<br><br>
### In Summary
1. Open buggy_company in Ghidra.
2. Search for the main function. ⭐ 
    * When you find it, rename it to main.

3. Along the way, rename what you know.
4. You will find that there are 2 main issues with buggy_company
    * Debugger check. So, you cannot run buggy_company while using GDB.
    * Passphrase .env variable is empty, which returns noloot() automatically. Dang.

5. You have the hash from checking strings. This is decoded into YIPPEE.
6. Set YIPPEE into the environment variable into your terminal.
7. Run buggy_company.
