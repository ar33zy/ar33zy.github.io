---
layout: single
title:  "HTB Writeup - Traverxec"
excerpt: "Traverxec is one of the beginner friendly boxes in HTB. This machine is hosting a webserver vulnerable to remote code execution, exposing a backup SSH private key for user pivot, and allowing a non-privileged user invoke journalctl as root leading to machine pwn."
date: "2020-04-17"
classes: wide
categories:
- hackthebox 
- machines
tags:
- nostromo 
- journalctl 
- ssh2john
- gtfobins
---

![](/assets/images/htb/traverxec/logo.png)  

Traverxec is one of the beginner friendly boxes in HTB. This machine is hosting a webserver vulnerable to [remote code execution](https://www.rapid7.com/db/modules/exploit/multi/http/nostromo_code_exec), exposing a backup SSH private key for user pivot, and allowing a non-privileged user invoke journalctl as root leading to machine pwn.

## Recon

A quick masscan + nmap scan gives us the following info.  

![](/assets/images/htb/traverxec/scan.png)  

It seems that the entry point for this machine is limited to 2 ports, via HTTP and via SSH. Credentials are needed for SSH access so let's proceed with checking up the web server. Take note that the web server is running ```nostromo 1.9.6```.

## Nostromo Webserver

Upon checking up the web server, it hosts a static page showing ```David White's``` portfolio.   

![](/assets/images/htb/traverxec/page.png)  

Since this is a easy box, one of the checklists is to search for ```nostromo 1.9.6``` on [Exploit DB](https://www.exploit-db.com/). Searchsploit can be used to query for existing exploit scripts.

![](/assets/images/htb/traverxec/searchsploit.png)  

There is an existing python script that can be used to do an RCE to the webserver. This script can be pulled using ```-m``` parameter.  

The script can be used as ```python 47837.py <Target_IP> <Target_Port> <Command>```.  

A quick check using ```id``` shows that the current user is ```www-data```. There is also a ```netcat``` binary that has ```-e``` option enabled, this command can be used to do a reverse shell for easier access on the machine.

![](/assets/images/htb/traverxec/rce.png)  

## PrivEsc to David

A quick check on ```/etc/passwd/``` shows that there is a user named ```david``` in this machine (david - David White the web developer lol).  

![](/assets/images/htb/traverxec/users.png)  

Sadly, ```www-data``` does not have enough privileges to access ```david's``` home directory. Back to the drawing board.

![](/assets/images/htb/traverxec/david.png)   

Backtracking to nostromo web server. The config file gives us the following info:  


![](/assets/images/htb/traverxec/nhttpd.png)  

There are notable infomation in this conf file:  

```
# HOMEDIRS [OPTIONAL]

homedirs		/home
homedirs_public		public_www
```

According to [nhttpd](https://www.gsp.com/cgi-bin/man.cgi?section=8&topic=nhttpd) documentation,  ```homedirs``` option serves the home directories of the machines' users. This directory can be accessed by entering ```~<user>``` in the URL.

![](/assets/images/htb/traverxec/homedavid.png)    

Next, ```homedirs_public``` restricts the access within the home directory into a single sub directory. This means that our access is limited to ```public_www``` directory.  

We can check the contents of ```/home/david/public_www/``` via shell.  

Upon checking, there is a ```tgz``` file (```backup-ssh-identity-files.tgz```) inside ```/home/david/public_www/protected-file-area/```. This file can be extracted using ```netcat```.

![](/assets/images/htb/traverxec/files.png)    

The ```tgz``` file contains david's backup SSH keys.

![](/assets/images/htb/traverxec/sshkeys.png)    

The private key needs a passphrase, this can be cracked using ```ssh2john```.

![](/assets/images/htb/traverxec/cracked.png)    

The passphrase is ```hunter```. The private key can now be used to login via SSH as ```david```. 

**Note: Do not forget to change the file permission of id_rsa to 400/600.**  

![](/assets/images/htb/traverxec/davidssh.png)    

## PrivEsc to Root

A quick check on the files inside the home directory shows that there is a script worth investigating.

![](/assets/images/htb/traverxec/david_files.png)    

This line in the script can be abused.  

```
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service | /bin/cat
```  

According to [GTFObins](https://gtfo.hackademint.org/gtfobins/journalctl/), journalctl invokes a pager. Pager tools can be abused via shell escape.

![](/assets/images/htb/traverxec/gtfo.png)    

Since ```journalctl``` can be executed as root, we can use this technique to escalate privileges.

![](/assets/images/htb/traverxec/fail.png)    

Executing this command did not ivnoke a pager, and that is because of ```-n``` parameter. According to ```journalctl``` man page, 

```
       -n, --lines=
           Show the most recent journal events and limit the number of events shown. If --follow is used, this option is implied. The argument is a positive integer or
           "all" to disable line limiting. The default value is 10 if no argument is given.
```

The trick we can use is to minimize the terminal, so that we can force to open a pager even if the output is just 5 lines.

![](/assets/images/htb/traverxec/pager.png)    

Executing ```!/bin/sh``` while on a pager gives us root privileges.

![](/assets/images/htb/traverxec/root.png)    

-- 

![ar33zy](https://www.hackthebox.eu/badge/image/26849)
