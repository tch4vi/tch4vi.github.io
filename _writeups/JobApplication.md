---
layout: writeup
title: "JobApplication"
date: 2026-01-20
platform: Free
---

On today's hacking, the writeup will be a bit different. I won't be using all the details of the machine i've been working on, this comes from a Job Application and I will keep the identity of the company and the details of the activity censored. But somehow, I wanted to write it so let's wrap it.
I've completed this challenge last week, I was scrolling on Linkedin and saw a job offer that was challenging whoever wanted to take part of the application and shared a code block in hexadecimal:
```
6148523063484d364c79397a5a57
4e31636d6c306553316c59584e30
5a5849745a57646e4c6e4d7a4c57
...
```

As I said above, to not disrupt any selection process or anything I won't share the full details of the challenge (eventho this blog works as a hacked-machines-library for me)

Going back to the challenge. Judging by the integrity of the code block provided, we can see that it's an hexadecimal encoding. I proceed to use CyberChef tool to see what's inside and found that this code, was a hiding a  compressed `.tar` file that had a docker web inside.

![AdevintaOS](/assets/AdevintaOS/cyberchef.png)
<!--more-->
I used CyberChef tool but using `echo "61485230... | xxd -r -p | base64 -d` would work too:

![AdevintaOS](/assets/AdevintaOS/cyberchef2.png)

After downloading the `.tar` file from the url that was provided we got the following files:

![AdevintaOS](/assets/AdevintaOS/lsladocker.png)


![AdevintaOS](/assets/AdevintaOS/catnginx.png)



![AdevintaOS](/assets/AdevintaOS/catdockerfile2.png)

After checking some of the files and I decided that it was a good moment to run docker, start the services and see what was offering to me. In the config file I saw something related to nginx so I was expecting some sort of web vulnerabilities:

`docker build -t easteregg .`

![AdevintaOS](/assets/AdevintaOS/dockerbuildfail.png)
My first try with docker failed but it was because i'm using Podman instead of docker, which seems to be more strict than Docker. I had to edit the Dockerfile with the full path of the `nginx:1.19-apline` which is `docker.io/library/nginx:1.19-alpine` 

![AdevintaOS](/assets/AdevintaOS/catdockefile.png)


After that, running again the `docker build -t easteregg .` worked fine and could take a look arround the webpage that was hosting

![AdevintaOS](/assets/AdevintaOS/terminal.png)

I've tried some commands here and there in the emulated terminal but nothing worked, I assumed that the first flag would be at least get a user with privileges to see file `passwords.txt`. 

![AdevintaOS](/assets/AdevintaOS/terminal2.png)

The web simulates a common terminal, but before digging deeper into the files that the challenge provided me, I tried some stuff in the devtools, from the browser, but it gave no results. Quickly I discovered that, this wasn't the way to solve it, or at least I didn't see it.

The compressed file, came with 3 files that caught my eye; `validate.wasm`, `validate.js` and `terminal.js`

![AdevintaOS](/assets/AdevintaOS/lsladist.png)

I started working with `terminal.js`, the most important thing that I found was the string the terminal spawns once you get the flag

![AdevintaOS](/assets/AdevintaOS/cateterminal2.png)

This is very clarifying because, we get the confirmation that the terminal validate the input received by the .wasm files. After seeing this I went again to the devtools from the browser, but still gave no results (i'm very persistent, in my head I was trying to solve the riddle through the devtools).
The best way to solve this was by completing a reversing with WASM, which, i'm not very experienced on that. But, we have plenty of information at our disposal so I started working on that.

Worked on the tool `wasm2wat`, can be useful, but the best result came from the commands `wasm-objdump` and `wasm-decompile`

`wasm-objdump -d validate.wasm > disasm.txt`


![AdevintaOS](/assets/AdevintaOS/wasmobjdump.png)

![AdevintaOS](/assets/AdevintaOS/catdisasm.png)

![AdevintaOS](/assets/AdevintaOS/wasmobjdump2.png)


`wasm-objdump -x validate.wasm`

![AdevintaOS](/assets/AdevintaOS/wasmobjdump3.png)

From this command `wasm-objdump -x` this is the part that I think is the most important.

```
Export[6]:
 - memory[0] -> "b"
 - func[6] <c> -> "c"
 - func[5] <d> -> "d"
 - func[3] <e> -> "e"
 - func[2] <f> -> "f"
 - func[4] <g> -> "g"
```

This means that WASM is exporting 5 public functions called `c, d, e, f, g` . And my guess is that we need to find the function that is being executed in the terminal when we run the command `su`

After using `wasm-objdump`, I then proceed to run the command `wasm-decompile` that showed the some clear clues to identify the flag:

`wasm-decompile validate.wasm > decompiled.txt`

![AdevintaOS](/assets/AdevintaOS/wasmdecompile.png)

The file contains +1000 code lines:

![AdevintaOS](/assets/AdevintaOS/catdecompiled.png)


This section is the most important:

![AdevintaOS](/assets/AdevintaOS/catdecompile2.png)

That section over here seems to be the flag, but it's in a very obfuscated way. The comparison against 56 reveals that the password has a fixed length of 56 characters.

```
if ( ... == 56 ) {
  (
    a.s == 115 & a.w == 115 & a.db == 109 & a.cb == 111 &
    a.k == 104 & a.da == 103 & a.va == 118 & a.d == 117 &
    a.a == 115 & a.fa == 57 & a.la == 57 & ...
```
 
The structure is as follows:

```
export function d(a:{
 a:ubyte, b:ubyte, c:ubyte, d:ubyte, e:ubyte, f:ubyte, g:ubyte, h:ubyte,
 i:ubyte, j:ubyte, k:ubyte, l:ubyte, m:ubyte, n:ubyte, o:ubyte, p:ubyte,
 q:ubyte, r:ubyte, s:ubyte, t:ubyte, u:ubyte, v:ubyte, w:ubyte, x:ubyte,
 y:ubyte, z:ubyte,
 aa:ubyte, ba:ubyte, ca:ubyte, da:ubyte, ea:ubyte, fa:ubyte, ga:ubyte, ha:ubyte,
 ia:ubyte, ja:ubyte, ka:ubyte, la:ubyte, ma:ubyte, na:ubyte, oa:ubyte, pa:ubyte,
 qa:ubyte, ra:ubyte, sa:ubyte, ta:ubyte, ua:ubyte, va:ubyte, wa:ubyte, xa:ubyte,
 ya:ubyte, za:ubyte,
 ab:ubyte, bb:ubyte, cb:ubyte, db:ubyte
})
```


The decompiled file seems to have in a very obfuscated way the password contained there ,it cannot be seen at first sight but there is a clear pattern. 

I've created the following python script (with some help from ChatGPT) that decrypts and orders the ubytes.
1.- Extracts `a.xxx == number`
2.- Stores it in a dictionary
3.- Reconstructs in the correct order
4.- Converts ASCII -> text


```
python3 - << 'EOF'
import re

fields = [
    "a","b","c","d","e","f","g","h","i","j","k","l","m","n","o","p","q","r","s","t","u","v","w","x","y","z",
    "aa","ba","ca","da","ea","fa","ga","ha","ia","ja","ka","la","ma","na","oa","pa","qa","ra","sa","ta","ua","va","wa","xa","ya","za",
    "ab","bb","cb","db"
]

text = open("decompiled.txt").read()

pairs = re.findall(r"a\.([a-z]+)\s*==\s*(\d+)", text)
mapping = {k: int(v) for k,v in pairs}

result = ""
for f in fields:
    if f not in mapping:
        result += "?"
    else:
        result += chr(mapping[f])

print(result)
print("Length:", len(result))
EOF
```

And after that we found the flag:

![AdevintaOS](/assets/AdevintaOS/flag.png)

![AdevintaOS](/assets/AdevintaOS/flag2.png)

To summarize everything a bit the steps we followed:
Stage 1 - The initial message (hex -> base64 -> URL)
We recognised the encoding, used tools like CyberChef and decoders.

Stage 2 - Analyze the content of the tar.gz file
We discovered that the html was emulating a terminal. The command `su` was calling JS code. That JS was calling a WebAssembly (`validate.wasm`)

Stage 3 - Understand what does the JS code
Discovered that the JS acts only as mediator. Does not validate.

Stage 4 - WASM
We used commands as `wasm-objdump` to see real exports, and `wasm-decompile` so the code is readable.
Discovered the `export function d(a:{ ... 56 bytes ... }):int` that gave us the clues. Every character from the password was being compared to an ASCII value:

```
Example:
115 → 's'
101 → 'e'
99  → 'c'
```

Wrote a Python script that extracts the numbers, stores it in a dictionary, reconstructs in the correct order and converts ASCII into text.

Good challenge







