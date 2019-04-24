---
title: "AngstromCTF 2019 Secret Sheep Society Crypto 120 Writeup"
author: "CHERIEF Yassine (omega-coder)"
date: 2019-04-24T09:27:58+02:00
subtitle: "AES CBC Mode bit-flipping attack"
tags: ["ctf", "crypto", "bit-flipping", "aes", "security"]
categories: ["security", "writeups"]
---

# TL;DR

1. Spotting the weakness (AES with CBC Mode).
2. Get token
3. Flipping specific bytes in session json (turn **`flase`** to **`true `**).
4. manipulate token with flipped bytes.
5. Send manipulated token to page.
6. Get the flag. 

<hr>

## Challenge details 

| **Category**  | **Points** | **Solves** |
|-----------|--------|--------|
| Crypto | 120    | 0     |

<hr>

## Challenge Description

> The sheep are up to no good. They have a [web portal](https://secretsheepsociety.2019.chall.actf.co/) for their secret society, which we have the source for. It seems fairly easy to join the organization, but climbing up its ranks is a different story.<br> Author: defund

## Challenge Source

challenge app source can be found [here](https://github.com/omega-coder/ctf-writeups/tree/master/AngstromCTF_2k19_Quals/Secret_Sheep_Society/app_source)

<hr>


## What the app is doing ?

we have three important routes `/`, `/enter` and `/exit`.

1. `/` will show the flag if we have admin access.
2. `/enter` accepts an input called `handle` and sets a cookie named `token` which is simply the base64 encoding of the `session` encryption using `AES` with `CBC` Mode and using Padding also.
3. `/exit` will simply **unset** the **`token`** cookie.


## what we know VS. what we don't.
<hr>

#### We know : 
-----------------------
1. the format of the session which is : `{"admin": false, "handle": "some_input_we_control"}`
2. encryption is using `AES_CBC` with `BLOCK_SIZE = 16`.
3. We have the ciphertext.
<br>
<hr>

#### We don't know:
1. The Key.

<hr>

**`NOTE: I haven't used the browser for this task, i still don't know what the UI looks like!!`**

let's see this piece of code:  

```python
@app.route('/enter', methods=['POST'])
def enter():
	handle = request.form.get('handle')
	session = {
		'admin': False,
		'handle': handle
	}
	token = manager.pack(session)
	response = redirect('/')
	response.set_cookie('token', token)
return response
```

To get a cookie we post to `/enter` with handle data parameter set to anything we want, after that the session variable gets set and passed to `manager.pack(session)` which will return a token base64 encoded.

Let's see now how does the pack method works. (.pack is defined in `manage.py file`).  
**pack()** takes a python dictionary as an argument and return a base64 encoded token.

```python
class Manager:

	BLOCK_SIZE = AES.block_size

	def __init__(self, key):
		self.key = key

	def pack(self, session):
		cipher = AES.new(self.key, AES.MODE_CBC)
		iv = cipher.iv
		dec = json.dumps(session).encode()
		enc = cipher.encrypt(pad(dec, self.BLOCK_SIZE))
		raw = iv + enc
        return base64.b64encode(raw)
```

1. creates a new AES cipher with CBC mode
2. get an IV
3. get string representation of the json object session.
4. pad data and encrypt it with `AES_CBC`.
5. return `base64(iv + enc)`.

> **`Note: iv is not encrypted, the IV is the first 16 bytes of the token found in cookie`**

Our string representation of the session json object is as follow: 

This is the default json object string repr that we will have, provided that we enter `xx` as handle paramter. (Only an example).

```json
{"admin": false, "handle": "xx"}
```

We obviously need to turn that `false` to <code>true&nbsp;</code>, since `false` is 5 bytes long and `true` is only 4 bytes long, we can replace `false` with <code>true&nbsp;</code> with an extra space at the end, that won't make a difference when parser by JSON.

if we provide handle with value `xx` then json string representation will be on exactly two blocks.
(`32 Bytes`).  

The AES encryption will be 64 Bytes (4 Blocks of 16 bytes each), because IV is on 16 bytes, json string on 32 Bytes and padding on One block (16 Bytes)  

> 16 + 32 + 16 = 64

```
.................. = IV (IV is known to us)
{"admin": false,   ==> First Plaintext Block
 "handle": "xx"}   ==> Second Plaintext Block
\x10\x10\x10\x10 ... \x10 ==> last padding block of size 16 

```


Now let's do bit-flipping and change the  <code>false</code> to <code>true&nbsp;</code>.

To understand the attack, we need to closely look how decryption takes place in this mode:

![aes_cbc_decryption_scheme](https://res.cloudinary.com/https-omega-coder-github-io/image/upload/v1556110161/601px-cbc_decryption-svg.png)

Let's suppose our `{"admin": false,` is our first block, this plaintext block is resulting from `XORing` the IV (Initialization Vector) with the first ciphertext block, as you can see from the figure above.We can manipulate the cookie returned by the server to forge a cookie that we decoded get json encoded to `{"admin": true , "handle": "xx"}`. A rule says that whenever you change one byte ('flips') at an offset
in a given ciphertext block, the same offset is changed in the next plaintext block (see figure below)


![cbc_flip_byte](https://res.cloudinary.com/https-omega-coder-github-io/image/upload/v1556114028/082113_1459_cbcbyteflip3.jpg)


## Fully Automated Exploit with Python


let's first try to get a token by posting handle data to `/enter`

```python
import requests
import re
URL = "https://secretsheepsociety.2019.chall.actf.co/"

session = requests.Session()

# generate token from /enter route 
token = None
enter_data = {"handle": "xx"}
print("[+] Sending 'xx' as handle payload!")
token_exp = re.compile(r'token=(.*);')
req = session.post(URL+"enter", data=enter_data, verify=False, allow_redirects=False)
print("[?] Made request to /enter")
if req.status_code == 302: # means we are being redirected to / 
    print("[?] Got 302 Redirect request")
    match = re.search(token_exp, req.headers["Set-Cookie"])
    if match:
        token = match.group(1)
        print("[+] Got token : ", token)
```

Now let's see the full script that will get a token from `/enter` and manipulate it and send it back to man route (index.html) to get the final flag.


```python
import requests
from base64 import b64encode, b64decode
import re
URL = "https://secretsheepsociety.2019.chall.actf.co/"

session = requests.Session()

# generate token from /enter route 
token = None
enter_data = {"handle": "xx"}
# xx is just to make it 2 blocks in size
print("[+] Sending 'xx' as handle payload!")
token_exp = re.compile(r'token=(.*);')
req = session.post(URL+"enter", data=enter_data, verify=False, allow_redirects=False)
# allow redirects is set to False because if set to True we won't be able to retrive
# the cookie set by /enter

print("[?] Made request to /enter")
if req.status_code == 302: # we are being redirected to /
    print("[?] Got 302 Redirect request")
    match = re.search(token_exp, req.headers["Set-Cookie"])
    if match:
        token = match.group(1)
        print("[+] Got token : ", token)

# 'f' is located at offset 10 in the second block.
# 'a' is located at offset 11 in the second block.
# 'l' ....
# 's' ....
# 'e' ....
if token is not None:
    ct = b64decode(token)
    manipulated_token = list(ct)
    manipulated_token[10] = manipulated_token[10] ^ ord('f') ^ ord('t')
    manipulated_token[11] = manipulated_token[11] ^ ord('a') ^ ord('r')
    manipulated_token[12] = manipulated_token[12] ^ ord('l') ^ ord('u')
    manipulated_token[13] = manipulated_token[13] ^ ord('s') ^ ord('e')
    manipulated_token[14] = manipulated_token[14] ^ ord('e') ^ ord(' ')
    print("[+] Flipped Bytes 10, 11, 12, 13 and 14")

    final_token = b64encode(bytes(manipulated_token))

    print("[+] generated new token ", final_token.decode())

if final_token is not None:
    session = requests.Session()
    cookies = {"token": final_token.decode()}
    
    # if token is correct then session will contain {"admin": True, "handle": "xx"} 
    # we will be able to get the flag

    flag_exp = re.compile(r'actf{.*}')
    req = session.get(URL, cookies=cookies)
    if req.status_code == 200:
        m = re.search(flag_exp, req.text)
        if m:
            print("[+] FLAG: ", m.group(0))

```

`FLAG = actf{shep_the_conqueror_slumbers}` 

Thanks for Reading.





















































