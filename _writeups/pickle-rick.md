---
layout: writeup
title: "Pickle Rick – TryHackMe"
date: 2026-01-09
platform: TryHackMe
---
On today's hacking, i've worked on Pickle Rick, a machine located in TryHackMe platform that shows some clear web vulnerabilities. It's very easy to digest, entertaining and it's themed as Rick & Morty which I love.
To start working on the machine, I did the standard enumeration process with nmap:

`nmap -p- -sS -Pn 10.67.188.134`

`-p-` -> To scan all ports

`-sS` -> To perform a TCP SYN scan (half-open). Sends SYN packets, doesn't complete the three-way handshake, is faster and generates less logs

`-Pn` -> Disable the host discovery and assume that the host is active


![PickleRick](/assets/PickleRick/nmap1.png)

Now, that we get the port 22 and the port 80 open, I tend to perform a second scan to get more information about the active ports:
<!--more-->
`nmap -p22,80 -sCV 10.67.188.134`

`-p22,80` -> Specify the ports

`-sCV` -> Use default nmap scripts and the version of the software detected


![PickleRick](/assets/PickleRick/nmap2.png)

After the nmap scans, I wanted to see what the port 80 was offering me and found this image and the description that specifies that we will need to find three ingredients, those ingredients work as flags to capture.

![PickleRick](/assets/PickleRick/mainpage.png)

 
 Here I wanted to check if I could get some directories information with Feroxbuster tool. Works similar as the dirbuster, gobuster..:
 
`feroxbuster -u http://10.67.188.134/`

![PickleRick](/assets/PickleRick/feroxbuster.png)

Found the /assets directory, but there is nothing that caught my eye there so I keep digging on the main page. The page itself doesn't offer much to do besides viewing the source page, where I found a username:

![PickleRick](/assets/PickleRick/sourcepage.png)


![PickleRick](/assets/PickleRick/username.png)

`R1ckRul3s`

I got an username: R1ckRul3s.
Although I used feroxbuster to identify directories and paths and found nothing, I started trying some common directories based in common web application patterns like /panel.php, /login.php /portal.php /robots.txt and found the login page and a strange string stored in the robots.txt

![PickleRick](/assets/PickleRick/robotstxt.png)

`Wubbalubbadubdub`


Login page:

![PickleRick](/assets/PickleRick/loginpage.png)


At this stage, the login page was identified. While the username was already known, the password was not immediately obvious. The string found in the `robots.txt` file initially appeared misleading, as this file is typically used to define allowed or disallowed paths for web crawlers rather than store credentials. However, further testing confirmed that this value was reused as the login password and after that, we met the exploitable area of the machine:

![PickleRick](/assets/PickleRick/webshell.png)

I don't have permissions to use any other tab other than the Commands one, but that's more than enough.
At this point I wanted to know what user I'm on, and what are the permissions of that user

`whoami`

![PickleRick](/assets/PickleRick/whoami.png)

I'm using www-data, which was expected, then proceed to list directories and files from the current path:

`ls -la`

![PickleRick](/assets/PickleRick/lsla.png)

Here happens something interesting, when you try to use `cat`, `more` or `head` you get this Mr.Meeseek gif and the text description saying that the command is disabled. The interesting part is that, this is not the outcome when you try to use a command where you don't have permissions, so these commands `cat`, `more` and `head` are manually disabled:

`cat Sup3rS3cretPickl3Ingred.txt`

`more Sup3rS3cretPickl3Ingred.txt`

`head Sup3rS3cretPickl3Ingred.txt`


![PickleRick](/assets/PickleRick/cat.png)

This is the outcome when you try to use `ls /root` for example, you get no results because you don't have permissions:

![PickleRick](/assets/PickleRick/nopermissions.png)

But that is totally different with command `less`, this command is not disabled, it works and shows the first ingredient

`less Sup3rS3cretPickl3Ingred.txt`

![PickleRick](/assets/PickleRick/less1.png)

Moving on trying to find the second ingredient, I moved around the directories. I could have spawned a reverse shell but this webshell seemed Ok to me to keep working. Using `ls -la /home` I found 2 directories from 2 different users, `rick` and `ubuntu`, the one that got my attention was user `rick` 

`ls -la /home`

![PickleRick](/assets/PickleRick/lslahome.png)

`ls -la /home/rick`

![PickleRick](/assets/PickleRick/lslahomerick.png)

I found the second ingredient, applying the same logic with the first one, the command `less`gave me the information I needed to complete the machine:

`less /home/rick/"second ingredients`

![PickleRick](/assets/PickleRick/less2.png)

The last step to complete the machine, the third ingredient, I was sure that it had to be on the `/root`folder, but as shown in previous captures, I get no results. 
Some key commands to do privilege escalation is the `sudo -l`one, it shows all the commands that the current user can use as super user. In this case we can use everything.

`sudo -l`

![PickleRick](/assets/PickleRick/sudol.png)

`NOPASSWD` -> No password required

`ALL` -> Any command

After seeing that, I can use super user permissions with the current user www-data and no password is required.

`sudo ls -la /root`

![PickleRick](/assets/PickleRick/sudolsroot.png)

`sudo less /root/3rd.txt`

![PickleRick](/assets/PickleRick/sudolessroot.png)

Completed the machine. Entertaining and easy to digest


![PickleRick](/assets/PickleRick/completed.png)
