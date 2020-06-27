---
layout: single
title: encryptCTF 2019 Pwn Write-up 3 of 5
excerpt: Third part of my encryptCTF 2019 Pwn write-up series. This challenge tackles stack buffer overflow — creating a ROP chain to call gets() -> main() -> system(), leading to a shell.
date: '2019-04-04T10:13:10.467Z'
classes: wide
categories:
- ctf
- encryptctf
tags:
- pwn
- rop
- bof
---

| ![First pwn board wipe of the year. hsb represent! :)](https://cdn-images-1.medium.com/max/800/1*ycEO30rNNBaEmC9qIWf8VQ.png) |
|:--:|
| First pwn board wipe of the year. hsb represent! :) |

### Pwn2 Solution (100 pts.)

> This challenge tackles stack buffer overflow — creating a ROP chain to call gets() -> main() -> system(), leading to a shell.

Challenge description:

> I made a simple shell which allows me to run some specific commands on my server can you test it for bugs?

Let’s examine the binary.

![Commands used: file and gdb checksec](https://cdn-images-1.medium.com/max/800/1*q7sHTSG5xY9FFTJd9AYBfg.png)

Again, the file is a 32-bit ELF executable, and Canary, PIE and RelRo are disabled. Hence, we can try to do a buffer overflow to overwrite the saved return address.

Let’s try to run the binary and enter a long unique string to determine if we can overwrite the saved return address.

Generate pattern using msf-pattern_create.rb

```
root@kali:pwn2# msf-pattern_create -l 300
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9
```

Let’s use the string pattern as our input.

![Segfault again :)](https://cdn-images-1.medium.com/max/800/1*o5K1sfoRva7OmtwjUc76qg.png)

It seems we have overwritten the saved return address with 0x35624134. Let’s compute for the offset using msf-pattern_offset.rb.

```
root@kali:pwn2# msf-pattern_offset -q 0x35624134  
[*] Exact match at offset 44
```

We got the offset at 44. Our payload template should look like this:

```
payload = [44 bytes buffer] + [addr of next function]
```

Now, let’s inspect the binary with radare2 and look for useful functions.

![run_command_ls??](https://cdn-images-1.medium.com/max/800/1*6aAIM9OyN2cEsVy8NQKDZw.png)

run_command_ls function is promising, since it uses the function system(). Upon checking the value at 0x8048670 —

```
gef➤  x/xs 0x8048670  
0x8048670: "ls"
```

With the data above, run_command_ls works  like this —

```
run_command_ls() = system('ls')
```

With all the details we got, it seems that the run_command_ls() is useless. We need to think of another way to get the flag / spawn a shell.

Based on the functions listed on radare2, we can utilize system() function by calling system(“/bin/sh”). But there is no /bin/sh stored on the program, hence we still can’t do a simple chain of `system(‘/bin/sh’)`.

Luckily, we have a gets() function. We can use the gets function to store “/bin/sh” on a known writable address, then loop back to main function so that the program will not terminate.

```
Quick note: Template for 32-bit ROP chain  
<buffer> + <function_call> + <return_address after func> + <func_arg>
```

function argument for `gets()` == address where `gets()` will write the user input

```
payload = [44 bytes buffer] + [gets()] + [main()] + [known writable address]
```

We can use the .bss segment to store our “/bin/sh”. Let’s use objdump to get the address of the .bss segment.

![.bss segment is on 0x804a040](https://cdn-images-1.medium.com/max/800/1*kQ7dYWm4ZTFMxQJp3FwCkw.png)

Let’s check the .bss segment using gdb.

![There is something on the given .bss address.](https://cdn-images-1.medium.com/max/800/1*CjFayxjhEHKyFWAt1U1n2w.png)

Since `stdout@GLIBC_2.0` is written on the first part of the `.bss` segment, let’s just use `0x804a050` as the storage of our `/bin/sh`.

Now, our stage 1 payload would look like this —

```
payload = [44 bytes buffer] + [gets()] + [main()] + [0x804a050]
```

The program will loop back to main() once the string `/bin/sh` is written to `0x804a050`.

Now, let’s continue our exploit.

Since we already have a known address of `/bin/sh`, we can set up our stage 2 payload — chain that calls `system(‘/bin/sh’)`,  and send it once the program loops back to the main() function.

Let’s compute again the offset needed to overwrite the saved return address.

Again, we will use msf-pattern_create.rb and msf-pattern_offset.rb

Generate pattern using msf-pattern_create.rb

```
root@kali:pwn2# msf-pattern_create -l 300

Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9
```

Run the program with our first stage exploit, then send the pattern.

![Segfault :)](https://cdn-images-1.medium.com/max/800/1*F_UbFlTvy3cyKRN9LVWUsg.png)

The program leads to a segmentation fault, let’s check the core dump generated by the program.

![saved return address overwritten by 0x41326241](https://cdn-images-1.medium.com/max/800/1*3fIS0RBoVPrmPWmOkihSkw.png)

We can see that the saved return address is overwritten by `0x41326241`. Let’s use msf-pattern_offset.rb to get the offset.

```
root@kali:pwn2# msf-pattern_offset -q 0x41326241  
[*] Exact match at offset 36
```

The offset is 36. We can now create our stage 2 payload with all the information.
```
payload = [36 bytes buffer] + [system()] + [4 bytes garbage] + [/bin/sh address]
```

Let’s try to send our stage 2 payload after looping back to main. Here’s the sample run of the exploit.

![We got a shell! :)](https://cdn-images-1.medium.com/max/800/1*Q2tj6HTsUgwaDuFtDxe_-A.png)

The exploit worked! And we got the flag :)

Flag: encryptCTF{N!c3_j0b_jump3R}

-

— ar33zy

[hackstreetboys](https://hackstreetboys.ph/) aka [hsb] is a CTF team from the Philippines.

Please do like our [Facebook Page](https://www.facebook.com/hackstreetboys/) and Follow us on [Twitter](https://twitter.com/_hackstreetboys), [Medium](https://medium.com/hackstreetboys), and [GitHub](https://github.com/hackstreetboysph).
