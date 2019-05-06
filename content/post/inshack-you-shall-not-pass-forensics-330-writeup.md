---
title: "INShAck 2019 :: You Shall Not Pass :: Forensics 330 Writeup"
author: "CHERIEF Yassine Aka. omega_coder"
date: 2019-05-05T18:03:05+02:00
subtitle: "Data Extraction and Port knocking"
image: "https://res.cloudinary.com/https-omega-coder-github-io/image/upload/v1557078174/Screenshot_2019-05-05_18-43-57.png"
bigimg: [{"src": "https://res.cloudinary.com/https-omega-coder-github-io/image/upload/b_rgb:000000,o_37/v1557078174/Screenshot_2019-05-05_18-43-57.png", "desc": "Port knocking and Data Extraction"}]
tags: ["INShAck", "writeup", "ntfs", "port-knocking", "forensics"]
categories: ["security", "writeups", "INShAck", "forensics"]
---

# Challenge details.

| **Category**  | **Points** | **Solves** |
|-----------|--------|--------|
| Forensics | 330    |  11    |

# 1. Challenge description

> One of my friends is a show-off and I don't like that.<br> Help me find the backdoor he just boasted about! :D<br>You'll find an image of his USB key [here](https://static.ctf.insecurity-insa.fr/3b89ef8bb51773c8f3478bf356271ac762ec96c3.tar.gz).<br>And one last thing, my friend owns  `you-shall-not-pass.ctf.insecurity-insa.fr`.


# 2. Solution

## 2.1 <u>Data Extraction Part</u>

By extracting the compressed file that was given to us, we find a raw image of an NTFS filesystem, you can verify that using the `file` command on linux. (probably made with `dd` considering the filename :D )

```bash
file dd.img
```
result of the `file` command is shown below.

![file_command_output](https://res.cloudinary.com/https-omega-coder-github-io/image/upload/v1557080003/Screenshot_2019-05-05_19-15-58.png)


First I tried to mount the filesystem and see what we've got inside. (first you create a folder `/mnt/you_shall_not_pass`), then mount the image using the `mount` command on linux.

```shell
sudo mount -o loop -t ntfs dd.img /mnt/you_shall_not_pass
```

After mounting the filesystem, We got a bunch of weird folders & files, majority of them were text files containing some **`Bacon Ipsum`** (not really important)  and one MP4 video file,  the video file contained a scene from `LOTR` (Lord Of The Ring) movie (`That was a good scene, where Gandalf says "YOU SHALL NOT PAAASSSSS!"`).

Well, let's start the work, I watched the video, read the texts with no progress at all, then i said let's check the filesystem for some `hidden` files or `hidden` strings, I first tried the `strings` command, didn't help much. Then I tried to do `tail` on the filesystem, after doing `tail` on the image, something caught my EYE :D, which looks like a `Base64 encoded string`.

![base64_encoded_string](https://res.cloudinary.com/https-omega-coder-github-io/image/upload/v1557081352/Screenshot_2019-05-05_19-38-33.png)

```base64
H4sIAOq1yVwAA6tWUFBKyc8vUrJSMDIAAh0gvzi1sDQ1LzkVKBatAATVSgX5RSVAnqGBgSFQhVJB
UX5JPpCvFOoSoFSrg67Gkgg1RihqQpyxqSHGLjMizLEgwhxi7DIgwi4DIswxIcIcI0xzFBRiQbGT
X5CaF1+cWpyYC4ogJXdPX19XhRAPVwU3H0d3hQCfKD09PSVoNMZn5pWkFpUl5oAN1YHGNbKoaS0A
gssCMwICAAA=
```

It starts with **`BGZ`** and ends with **`EGZ`** !,well i guessed that right away, it's a `gzip compressed` file.

let's decode the base64 and see whats inside the file,

```bash
echo "H4sIAOq1yVwAA6tWUFBKyc8vUrJSMDIAAh0gvzi1sDQ1LzkVKBatAATVSgX5RSVAnqGBgSFQhVJB
UX5JPpCvFOoSoFSrg67Gkgg1RihqQpyxqSHGLjMizLEgwhxi7DIgwi4DIswxIcIcI0xzFBRiQbGT
X5CaF1+cWpyYC4ogJXdPX19XhRAPVwU3H0d3hQCfKD09PSVoNMZn5pWkFpUl5oAN1YHGNbKoaS0A
gssCMwICAAA=" | base64 -d > file.gz
```

Let's unzip the file using `gunzip` comamnd, and make sure you have `.gz` file extension.

```bash
gunzip file.gz
```

You will get in result a file named `file.tar`, but its just a text file, not a tar archive, you can verify it using the `file` command.

here is the content of the `file.tar`

```json
{
    "door": 20000,
    "sequence": [{
        "port": 10010,
        "proto": "UDP"
    }, {
        "port": 10090,
        "proto": "UDP"
    }, {
        "port": 10020,
        "proto": "TCP"
    }, {
        "port": 10010,
        "proto": "UDP"
    }, {
        "port": 10060,
        "proto": "TCP"
    }, {
        "port": 10080,
        "proto": "UDP"
    }, {
        "port": 10010,
        "proto": "UDP"
    }, {
        "port": 10000,
        "proto": "TCP"
    }, {
        "port": 10000,
        "proto": "UDP"
    }, {
        "port": 10040,
        "proto": "TCP"
    }, {
        "port": 10020,
        "proto": "UDP"
    }],
    "open_sesame": "GIMME THE FLAG PLZ...",
    "seq_interval": 10,
    "door_interval": 5
}
```

## 2.2 <u>Port Knocking Task</u>

We got a `JSON` file, it have a bunch of ports and some infos. `[seq_interval, door_interval, open_sesame] and door`

This is `OBVIOUSLY` a `port knocking` task, if you don't know about `port knocking` just **GOOGLE IT!!**

> In computer networking, `port knocking` is a method of externally opening `ports` on a firewall by generating a connection attempt on a set of prespecified `closed ports`.

<br>

So Basically, we gonna knock the ports in the given sequence according to the protocol in an interval of `10s` (_this was clarified by the challenge author @Paul_) to be able to connect to the service on port `20000` (_Our door :D_).

Here is my script for doing the port knocking and getting the flag.
It knocks the sequence then connects to service on port `20000`, sends the `open_sesame` string and gets the flag.

```python
#!/usr/bin/env python3

import time
import socket
import select
import json

class Knocker(object):
    def __init__(self, ports_proto: list, delay=400, udp=False, host="127.0.0.1", timeout=200):
        self.timeout = timeout / 1000
        self.delay = delay / 1000
        self.default_udp = udp
        self.ports_proto = ports_proto

        self.address_family, _, _, _, (self.ip_address, _) = socket.getaddrinfo(
                host=host,
                port=None,
                flags=socket.AI_ADDRCONFIG
            )[0]

    def knock_it(self):
        last_index = len(self.ports_proto) - 1
        for i, port in enumerate(self.ports_proto):
            use_udp = self.default_udp
            if port.find(':') != -1:
                port, protocol = port.split(':', 2)
                if protocol == 'TCP':
                    use_udp = False
                elif protocol == 'UDP':
                    use_udp = True
                else:
                    error = 'WTF!'
                    raise ValueError(error.format(protocol))

            s = socket.socket(self.address_family, socket.SOCK_DGRAM if use_udp else socket.SOCK_STREAM)
            s.setblocking(False)

            socket_address = (self.ip_address, int(port))
            if use_udp:
                print("Knocking port {} using UDP".format(port))
                s.sendto(b'', socket_address)
            else:
                print("Knoocking port {} using TCP".format(port))
                s.connect_ex(socket_address)
                select.select([s], [s], [s], self.timeout)

            s.close()

            if self.delay and i != last_index:
                time.sleep(self.delay)


if __name__ == '__main__':
    host = "you-shall-not-pass.ctf.insecurity-insa.fr"
    print("[?] Getting Data ...")
    json_file = open("file.tar", "r")
    data = json.load(json_file)
    open_sesame = data["open_sesame"]
    ports_proto = [str(i["port"])+":"+i["proto"] for i in data["sequence"]]
    print("[+] Knocking Ports now...")
    Knocker(ports_proto, delay=900, host=host).knock_it()
    time.sleep(1)
    print("[?] Asking for Flag ...")
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect((host, int(data["door"])))
    except Exception as e:
        print("[!] CONNECTION FAILED!!!, Port may not be opened")
    else:
        print("[+] DOOR OPENED")
        s.send(open_sesame.encode())
        try:
            flag = s.recv(2014).decode()
            print("Got flag: ", flag)
        except Exception as e:
            print("NOTHING")
    finally:
        s.close()

```






