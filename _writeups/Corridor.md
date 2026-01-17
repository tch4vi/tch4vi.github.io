---
layout: writeup
title: "Corridor – TryHackMe"
date: 2026-01-17
platform: TryHackMe
---
On today's hacking, I've been working in one small hacking challenge that TryHackMe has. They have some small hacking rooms that offer good concepts to learn in an entertaining way. Today, the room is called "Corridor", working on that room will teach us some learnings on IDOR's, hashing and more. Let's see:


I've started with the meta start of the CTF challenges, which is the enumeration with nmap. When you first enter into the challenge the description of the room makes you expect a web with some path traversal vulnerabilities, that is what I was expecting. So after using the command `nmap -p- -sS -Pn 10.65.150.204` I was expecting the port 80 open:

![Corridor](/assets/Corridor/nmap.png)

`-p-` -> To cover all ports

`-sS` -> To perform a TCP SYN scan (half-open). Sends SYN packets, doesn’t complete the three-way handshake, is faster and generates less logs

`-Pn` -> Disable the host discovery and assume that the host is active


After the enumeration with nmap and seeing the port http 80 open, I've tried to access the web and that's what I see: A corridor. Now the name of the challenge makes sense.


<img src="/assets/Corridor/corridorshow.png" alt="corridorshow" style="width:50%;">

Each door of the corridor is like a directory, you can open them and see what's inside. All the doors offer the same, an empty room with the same pattern on the URL:
`http://10.65.150.204/8f14e45fceea167a5a36dedd4bea2543`
<!--more-->

<img src="/assets/Corridor/roomshow.png" alt="roomshow" style="width:70%;">

This string `8f14e45fceea167a5a36dedd4bea2543`, at first I thought that was encoded as hexadecimal, and I even used CyberChef to try to see if something is hiding inside that string, but, then, you don't have to goo that far, just check the description of the room to see that it's a hash, and tools like JohnTheRipper or Hashcat will be our best allies instead of CyberChef.

![Corridor](/assets/Corridor/tryhackmedesc.png)

After knowing that all those strings are hashes, I wanted to use hashcat, so I decided to add all the hashes into one single file to work on all of them and see if there is any pattern recognizable:


![Corridor](/assets/Corridor/hashlist.png)

`hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt`

`-m 0` -> md5

`usr/share/wordlists/rockyou.txt` -> Path to rockyou.txt wordlist

![Corridor](/assets/Corridor/hashcat.png)

`hashcat -m 0 --show hash.txt`

`-m 0` -> md5

`--show` -> Show previous results

![Corridor](/assets/Corridor/hashcatshow.png)

After the hashcat results, seems like all those hashes are related to numbers from 1 to 13. To check if that is the correct answer we can try the following:

`echo -n "7" | md5sum`

![Corridor](/assets/Corridor/echotest1.png)

We get the same string. So let's check with some other numbers, let's try with the 14 for example, and using the 

`echo -n "14" | md5sum`

`-n` -> No line skip

`md5sum` -> Calculate the hash

![Corridor](/assets/Corridor/echotest2.png)


![Corridor](/assets/Corridor/noresult.png)

No result.
Let's try with the 0 now:

`echo -n "0" | md5sum`

`-n` -> No line skip

`md5sum` -> Calculate the hash

<img src="/assets/Corridor/flag.png" alt="flag" style="width:70%;">

Voilà!
Here is the flag. The clue to discover the flag was the pattern that all the rooms had, they were numbered from 1 to 13, and in most of the cases, boundary values like 0 are often overlooked by developers and are worth testing.

The important point is not the number 0 itself, but the fact that the backend trusted the identifier without checking if it should be accessible.
The lesson learned here is that, If an application is using predictable identifiers and doesn't validate the authorization in the backend, there is an IDOR. Hash ≠ secret.
Never trust identifiers that are obfuscated as a security mechanism. If the backend doesn't validate the authorization over the objects, it's enough by knowing the pattern of the identificators to access resources that are exposed (IDOR). That's what happened with the doors, we identified the hashes, we identified that they were numbered, identified the pattern, tried to access one that at first isn't exposed and found the flag.
