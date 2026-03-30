---
layout: writeup
title: "W1seGuy"
date: 2026-03-29
platform: TryHackMe
description: "A w1se guy 0nce said, the answer is usually as plain as day."
image: /assets/W1seGuy/w1se.png
---

On today's hacking i've been working on a "5 minute hack" from TryHackMe that it looked easy at first, but it took me a bit to complete it.

There are 2 tasks in this challenge, the first is an introductory one that allows us to download the script that the server is running

![W1seGuy](/assets/W1seGuy/task1.png)

```python
import random
import socketserver 
import socket, os
import string

flag = open('flag.txt','r').read().strip()

def send_message(server, message):
    enc = message.encode()
    server.send(enc)

def setup(server, key):
    flag = 'THM{thisisafakeflag}' 
    xored = ""

    for i in range(0,len(flag)):
        xored += chr(ord(flag[i]) ^ ord(key[i%len(key)]))

    hex_encoded = xored.encode().hex()
    return hex_encoded

def start(server):
    res = ''.join(random.choices(string.ascii_letters + string.digits, k=5))
    key = str(res)
    hex_encoded = setup(server, key)
    send_message(server, "This XOR encoded text has flag 1: " + hex_encoded + "\n")
    
    send_message(server,"What is the encryption key? ")
    key_answer = server.recv(4096).decode().strip()

    try:
        if key_answer == key:
            send_message(server, "Congrats! That is the correct key! Here is flag 2: " + flag + "\n")
            server.close()
        else:
            send_message(server, 'Close but no cigar' + "\n")
            server.close()
    except:
        send_message(server, "Something went wrong. Please try again. :)\n")
        server.close()

class RequestHandler(socketserver.BaseRequestHandler):
    def handle(self):
        start(self.request)

if __name__ == '__main__':
    socketserver.ThreadingTCPServer.allow_reuse_address = True
    server = socketserver.ThreadingTCPServer(('0.0.0.0', 1337), RequestHandler)
    server.serve_forever()
```

This is the script the server is running, so, having that said, whenever I start a connection to the server with ``nc`` command it generates a encrypted flag, the flag in question is encrypted with XOR and then Hex. The most important part from this code is that we get a sample of the flag: ``flag = 'THM{thisisafakeflag}'``, this tells a lot, for example that the flag starts with ``THM{`` and ends with the ``}``, Or at least this is my assumption.

![W1seGuy](/assets/W1seGuy/task2.png)

Moving on, this is what I get when I try to connect to the server through port 1337:

```zsh
┌──(tch4vi㉿kali)-[~/THM/W1seGuy]
└─$ nc -v 10.129.181.70 1337    
10.129.181.70: inverse host lookup failed: Unknown host
(UNKNOWN) [10.129.181.70] 1337 (?) open
This XOR encoded text has flag 1: 012b78341f640259211b101b410e1b215756240c140d477c0e392f4c273a27174c7f1a271b7a3d12
What is the encryption key? 
```

The challenge here is to reverse the encoded XOR provided and find the key that was used to generate that encoded text.
It's important to point out a few things about the encoding used here, the XOR. It's a symmetrical one, that means that, it uses a specific key to encode a plain text, and if I find the key, I can use it to decode the encoded text and get the plain text.
Another important thing, it's not just XOR encoding, it's HEX encoded aswell, I found that in the script i downloaded in the first task:

``hex_encoded = xored.encode().hex()`` 

Therefore the full process is: plain text -->key-> XOR --> HEX --> ``012b78341f640259211b101b410e1b215756240c140d477c0e392f4c273a27174c7f1a271b7a3d12``

This is considered a "5 minute hack" but it took me a bit to complete it, my python skills are a bit rusty but we made it.
I wrote the following script:

```python
import string

hex_encoded = "012b78341f640259211b101b410e1b215756240c140d477c0e392f4c273a27174c7f1a271b7a3d12"
encoded = bytes.fromhex(hex_encoded)

known = "THM{"
key_part = ""
for i in range(4):
    key_part += chr(encoded[i] ^ ord(known[i]))

print(f"First 4 chars: {key_part}")

for c in string.ascii_letters + string.digits:
    key = key_part + c
    result = ""
    for i in range(len(encoded)):
        result += chr(encoded[i] ^ ord(key[i % len(key)]))
    if result.startswith("THM{") and result.endswith("}"):
        print(f"Key found: {key}")
        print(f"Flag: {result}")
        break
```

Gonna break the whole script in blocks so I can explain what it does and the purpose of each one.

#### BLOCK 1 - Import Libraries

```python
import string
```

As the title says, import the module ``string`` that contains collections of predefined characters, letters and numbers.

#### BLOCK 2 - Input from the server

```python
hex_encoded = "012b78341f640259211b101b410e1b215756240c140d477c0e392f4c273a27174c7f1a271b7a3d12"
encoded = bytes.fromhex(hex_encoded)
```
Basically the information provided by the server, here is where I put the encoded XOR text. And in order to be able to operate with the string we need to use ``bytes.fromhex()`` to convert it to raw bytes, this way we can work with them

#### BLOCK 3 - Deduce the first 4 characters from the key

```python
known = "THM{"
key_part = ""
for i in range(4):
    key_part += chr(encoded[i] ^ ord(known[i]))
```

This block is a bit dense:

```python
known = "THM{"
```

We know that all the flags from TryHackMe start with "THM{", that is our advantage in this challenge, so here I put "THM{" as value.

```python
key_part = ""
```

Start an empty string called ``key_part``, where we will add the key every time we find a charactar

```python
for i in range(4):
    key_part += chr(encoded[i] ^ ord(known[i]))
```

In the for loop, we iterate 4 times, one for each character from "THM{".
``Key_part`` is the empty string that we stated above and it will add every character that fits our conditions

``chr(encoded[i])`` -> Byte 'i' from the encoded message, for example:

``encoded[i] = byte from flag[i] XOR key[i]``

``encoded[i] = byte from 'T' XOR key[0]``

``chr`` converts the number back to a characters. If the result is 57, ``chr(57) = '9'``


The ``ord(known[i])`` function converts one character into his ASCII numeric value:

```python
ord('T') = 84 
ord('H') = 72 
ord('M') = 77 
ord('{') = 123
```

And when you merge both functions, is where the magic happens:

flag XOR key = encoded <-- What the server did
encoded XOR flag = key <-- What do we do

We know ``encoded[i]`` (was provided by the server) and we know ``flag[i]`` (it's 'THM{'), then we can discover ``key[i]``
Example with the first character
```python
encoded[0] XOR ord('T') = key[0]
encoded[0] XOR 84 = key[0]
```


#### BLOCK 4 - Brute force the fifth character

```python
for c in string.ascii_letters + string.digits:
    key = key_part + c
```

The key we need to fin has 5 characters, that is information we get from the script we downloaded in the first task:
``res = ''.join(random.choices(string.ascii_letters + string.digits, k=5))``
The ``k=5`` is the length of the key, and we got 4 characters of the flag ('THM{'), with the characters of the flag we can find the four first of the key and brute force the last one. This loop tries the 62 possibilities from the last character (a-z, A-Z, 0-9), building each time a complete candidate key


#### BLOCK 5 - Decipher with each candidate key

```python
result = ""
for i in range(len(encoded)):
    result += chr(encoded[i] ^ ord(key[i % len(key)]))
```

For each candidate key, try to decipher the complete message. Important thing to point out here also. When encoding with XOR, the Key has to be as long as the text to encode, for example, if I want to use XOR to encode "laptop", the key length needs to be 6, If i want to use XOR to encode "mug", the key length will be 3. That's called "One-time Pad". But here it's not the case, we don't know exactly the length of the key nor the flag, so we use ``i % len(key)``, the ``%``(module) plays an important role here, forces the key to repeat constantly until its smaller than the encoded message. 


#### BLOCK 6 - Verify if the key is correct

```python
    if result.startswith("THM{") and result.endswith("}"):
        print(f"Key found: {key}")
        print(f"Flag: {result}")
        break
```

Here it's easy, we compare if the result we got starts with "THM{" and ends with "}". If those 2 conditions occur, we got the flag

The complete flow from this is the following one

HEX from the server --> bytes --> XOR with "THM{" --> 4 chars of the key --> brute force 5th char --> complete key --> decipher message --> flag

Result:

```bash
┌──(tch4vi㉿kali)-[~/THM/W1seGuy]
└─$ python3 test5.py
First 4 chars: Uc5O
Key found: Uc5Oo
Flag: THM{p1alntExtAtt4ckcAnr3alLyhUrty0urxOr}
```

```bash
┌──(tch4vi㉿kali)-[~/THM/W1seGuy]
└─$ nc -v 10.129.181.70 1337    
10.129.181.70: inverse host lookup failed: Unknown host
(UNKNOWN) [10.129.181.70] 1337 (?) open
This XOR encoded text has flag 1: 012b78341f640259211b101b410e1b215756240c140d477c0e392f4c273a27174c7f1a271b7a3d12
What is the encryption key? Uc5Oo
Congrats! That is the correct key! Here is flag 2: THM{BrUt3_ForC1nG_XOR_cAn_B3_FuN_nO?}
```

I wanted to write all of this because I learned a lot of python and cryptography. Wasn't easy for me, but we made it :]
