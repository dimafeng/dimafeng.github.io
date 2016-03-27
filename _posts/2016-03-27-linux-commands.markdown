---
layout: devops
title:  "Linux command line cheat sheet"
categories: linux
---
Last few month, I've been dealing with Linux servers. I had experience with Linux, I even was using Linux distributives as a primary operation system for a few year in the past. But I feel the lack of skill of administration. Here, I'm going to collect all basic commands and command combinations which I use in the day-to-day routine. It's really difficult to remember all the commands, parameters and keys when you don't do it every day like a full-time system administrator. This article may be used as a Linux commands cheat sheet.

## General shortcuts

* <kbd>Ctrl</kbd> + <kbd>r</kbd> commands execution history search.
* <kbd>Ctrl</kbd> + <kbd>x</kbd> <kbd>Ctrl</kbd> + <kbd>e</kbd> opens the current command in editor.
* <kbd>Ctrl</kbd> + <kbd>a</kbd> moves cursor to the beginning, <kbd>Ctrl</kbd> + <kbd>e</kbd> to the end.
* <kbd>Alt</kbd> + <kbd>◀</kbd> moves cursor backward one word.
* <kbd>Ctrl</kbd> + <kbd>u</kbd> deletes to the beginning of line.
* <kbd>Ctrl</kbd> + <kbd>l</kbd> clears the screen
* <kbd>Ctrl</kbd> + <kbd>D</kbd> = `exit`

## General tips

* `cd -` - changes directory to the previous one
* `sudo !!` - runs previous command with sudo
* `![command]` - runs previous command that starts with [command]
* `[cmd1] ; [cmd2]` - executes both commands ignoring result of [cmd1]
* `[cmd1] && [cmd2]` executes [cmd2] if [cmd1] is executed without errors
* `[cmd1] || [cmd2]` executes [cmd2] if [cmd1] is executed with errors

## Output manipulations

* `grep [keyword]` - leaves only lines contain keyword, `grep -E "[regex]"` works with regular expressions
* `awk "{print $1}"` - prints the first column of the output
* `awk "{[program]}"` - executes Perl-like script
* `sort -bf` - sorts output line by line. `b` - ignore leading blanks, `f` - ignore case, `h` - human sort, `r` - reverse
* `tail [file]` - shows the last part of a file. `f` - monitors a file for updates, `-n [N]`, `-[N]` - prints last N lines
* `head` - opposite to `tail`

## System information

* `lscpu` - system specs.
* `lsb_release -a` - Ubuntu version information
* `df -h` - overall information about filesystems
* `du -hs [path] | sort -hr | head` - top 10 biggest directories/files within the path sorted by directory/file size `s` - displays only total for each directory
* `free -m` - current memory usage. `m` show megabytes, `h` - human readable.
* `top` - displays processes. Sort by: %MEM - <kbd>Shift</kbd> + <kbd>m</kbd>, %CPU - <kbd>Shift</kbd> + <kbd>p</kbd>. <kbd>Shift</kbd> + <kbd>e</kbd> changes units. <kbd>c</kbd> toggles between command line and name for processes
* `top -p [PID]` - displays information for the specific PID/PIDs
* `ps aux` - static version of the `top`

## Archiving

* `tar -zxvf file.tar.gz` - extracts tar.gz into current directory. `z` - gzip, `x` - extract, `v` - verbose output, `f` - work with file (not stdout/stdin)
* `tar -cvzf test.tar.gz ./*` - creates tar.gz. `c` - create a file

## SSH

* `ssh-keygen -t rsa` - generates key pair
* `cat ~/.ssh/id_rsa.pub | ssh [user@host] "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"` - adds public key to a remote machine. A proper way for linux is `ssh-copy-id`, but it doesn't present on MacOS
* `scp [name]@[domain]:[remote path] [local path]` or `scp [local path] [name]@[domain]:[remote path]` - copies file from remote machine to local or vice versa

## Network

* `dig [domain]` - dns related information
* `nslookup [domain]` - translates a domain name to an IP address, `nslookup [ip]`  - vice versa
* `telnet [host] [port]` - connects to host:port by TCP
* `iptables -L -n -v` - displays status of the firewall. `L` - list rules, `v` - verbose, `n` - shows ip addresses in numeric format

### Netstat

**Netstat** is a command line utility that can show information about network connections in the system.

* `netstat -plunt` - show processes that are using ports on the server. `u` - udp, `t` - tcp, `p` - PID and program name, `l` - only listening, `n` - numerical addresses
* `netstat -r` - shows routing table. See [this article](http://www.techrepublic.com/article/understand-the-basics-of-linux-routing/) to understand the output.
* `netstat -s` - statistics

### Curl

* `curl -O [url]` - downloads file to the current directory
* `curl [url] -F file=@[file path]` - uploads file from [file path]
* `curl [url] -X [method] -H [header] -d [data] |  python -m json.tool` - sends [method] (GET/POST/PUT...) reqest to [url] with headers and [data] body. `python -m json.tool` formats json (useful tip: `alias prettyjson='python -m json.tool'`)

## Misc

### Mysql

* `mysqldump -u [uname] -p[pass] [dbname] | gzip -9 > [db.sql.gz]` - dumps database to a gzip archive
* `gunzip < db.sql.gz | mysql -u [uname] -p[pass] db` - restores database dump from the archive

### APT Package management

* `apt-cache search [keyword]` - searches package
* `apt-cache show [package name]` - retrieves a package description

### Docker

* `docker stop $(docker ps -a -q) & docker rm $(docker ps -a -q)` - stops and removes all containers, sometimes may require `-f` to force deletion.
* `docker volume rm $(docker volume ls -qf dangling=true)` - removes unused volumes
* `docker rmi $(docker images | grep "^<none>" | awk "{print $3}")` - removes untagged images
