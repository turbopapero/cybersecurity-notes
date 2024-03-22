## Port Scan

```shell
┌──(kali㉿kali)-[~]
└─$ nmap support.htb -Pn -sV          
Starting Nmap 7.93 ( https://nmap.org ) at 2022-12-07 05:27 EST
Stats: 0:00:19 elapsed; 0 hosts completed (1 up), 1 undergoing Connect Scan
Connect Scan Timing: About 9.00% done; ETC: 05:30 (0:03:22 remaining)
Stats: 0:00:45 elapsed; 0 hosts completed (1 up), 1 undergoing Connect Scan
Connect Scan Timing: About 21.50% done; ETC: 05:30 (0:02:44 remaining)
Nmap scan report for support.htb (10.129.227.255)
Host is up (0.048s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT    STATE SERVICE      VERSION
88/tcp  open  kerberos-sec Microsoft Windows Kerberos (server time: 2022-12-07 10:29:28Z)
464/tcp open  kpasswd5?
593/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 140.31 seconds                                                                   
```
