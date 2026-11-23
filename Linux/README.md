# Linux Administration Handbook

A practical, example-driven guide to Linux fundamentals and system administration. Examples target **Debian/Ubuntu** (apt, dpkg), but most commands work on any distribution.

> **Tip:** Practice in a VM, container, or a throwaway cloud instance. Commands marked `sudo` change the system.

## Table of Contents

1. [Linux Architecture](#1-linux-architecture)
2. [Terminal and Navigation](#2-terminal-and-navigation)
3. [Files and Directories](#3-files-and-directories)
4. [Text Processing](#4-text-processing)
5. [Permissions and Ownership](#5-permissions-and-ownership)
6. [Users and Groups](#6-users-and-groups)
7. [Processes and Services](#7-processes-and-services)
8. [Storage and Filesystems](#8-storage-and-filesystems)
9. [Package Management](#9-package-management)
10. [Scheduling and Logs](#10-scheduling-and-logs)
11. [SSH and Remote Administration](#11-ssh-and-remote-administration)
12. [Shell Environment](#12-shell-environment)
13. [Cheat Sheet and Practice Labs](#13-cheat-sheet-and-practice-labs)

---

## 1. Linux Architecture

### 1.1 Layers of a Linux system

```
+--------------------------------------------------+
|  User applications (nginx, python, vim, firefox)  |
+--------------------------------------------------+
|  Shell / GUI  (bash, zsh, GNOME)                  |
+--------------------------------------------------+
|  System libraries (glibc) and system calls        |
+--------------------------------------------------+
|  KERNEL (scheduler, memory, VFS, drivers, net)    |
+--------------------------------------------------+
|  Hardware (CPU, RAM, disks, NICs)                 |
+--------------------------------------------------+
```

### 1.2 Kernel

The kernel is the core of the OS. It:

- Schedules processes on CPUs
- Manages memory (virtual memory, paging, swap)
- Provides the virtual filesystem (VFS) and talks to drivers
- Handles networking and security (permissions, namespaces, cgroups)

```bash
uname -r            # kernel version, e.g. 6.8.0-45-generic
uname -a            # full system info
cat /proc/version   # kernel build info
lsmod               # loaded kernel modules
modinfo ext4        # info about a module
sudo modprobe loop  # load a module
```

### 1.3 Shell

The shell is a program that reads commands and asks the kernel (via programs) to run them. Common shells: `bash`, `zsh`, `fish`, `sh` (dash on Debian).

```bash
echo $SHELL         # your login shell
echo $0             # current shell
cat /etc/shells     # valid login shells
chsh -s /bin/zsh    # change your login shell
```

### 1.4 Distributions

A distribution = kernel + userland tools + package manager + defaults.

| Family  | Examples                    | Package format / tool |
|---------|-----------------------------|-----------------------|
| Debian  | Debian, Ubuntu, Linux Mint  | `.deb` / `apt`, `dpkg`|
| Red Hat | RHEL, Fedora, Rocky, Alma   | `.rpm` / `dnf`, `rpm` |
| Arch    | Arch, Manjaro               | `pacman`              |
| SUSE    | openSUSE, SLES              | `zypper`              |
| Alpine  | Alpine                      | `apk`                 |

```bash
cat /etc/os-release   # identify your distro
lsb_release -a        # Debian/Ubuntu-specific
hostnamectl           # host, OS, kernel summary
```

### 1.5 Filesystem Hierarchy Standard (FHS)

Everything is a file, under a single tree rooted at `/`.

| Path      | Purpose                                              |
|-----------|------------------------------------------------------|
| `/`       | Root of the tree                                     |
| `/bin`    | Essential user binaries (often symlink to `/usr/bin`)|
| `/sbin`   | Essential system binaries                            |
| `/boot`   | Kernel, initramfs, bootloader files                  |
| `/dev`    | Device files (`/dev/sda`, `/dev/null`, `/dev/tty`)   |
| `/etc`    | System-wide configuration                            |
| `/home`   | User home directories                                |
| `/root`   | Home directory of the root user                      |
| `/lib`    | Shared libraries                                     |
| `/media`  | Auto-mounted removable media                         |
| `/mnt`    | Temporary manual mounts                              |
| `/opt`    | Optional third-party software                        |
| `/proc`   | Virtual FS: process and kernel info                  |
| `/sys`    | Virtual FS: devices and kernel parameters            |
| `/run`    | Runtime data (PID files, sockets), cleared on boot   |
| `/srv`    | Data served by the system (web, ftp)                 |
| `/tmp`    | Temporary files (may be cleared on reboot)           |
| `/usr`    | Userland programs, libraries, docs                   |
| `/var`    | Variable data: logs, caches, spools, databases       |

```bash
ls /                       # see top level
cat /proc/cpuinfo | head   # CPU info from /proc
cat /proc/meminfo | head   # memory info
ls /dev | head
ls -l /dev/null /dev/zero /dev/urandom
```

### 1.6 Boot process

```
Power on
  -> Firmware (BIOS or UEFI) runs POST, finds boot device
  -> Bootloader (GRUB2) loads kernel + initramfs
  -> Kernel initializes hardware, mounts initramfs, then real root FS
  -> Kernel starts PID 1 (systemd)
  -> systemd starts services to reach the default target
  -> Login prompt / display manager
```

```bash
ls /boot                             # vmlinuz-*, initrd.img-*, grub/
cat /proc/cmdline                    # kernel boot parameters
systemd-analyze                      # total boot time
systemd-analyze blame | head         # slowest units at boot
journalctl -b                        # logs from this boot
journalctl -b -1                     # logs from the previous boot
sudo update-grub                     # regenerate GRUB config (Debian/Ubuntu)
```

### 1.7 systemd

systemd is the init system (PID 1) and service manager on most distributions. Its work is described by **units** (`.service`, `.timer`, `.socket`, `.mount`, `.target`).

```bash
ps -p 1 -o comm=                     # prints: systemd
systemctl get-default                # e.g. graphical.target or multi-user.target
sudo systemctl set-default multi-user.target
systemctl list-units --type=service --state=running
systemctl list-unit-files --type=service | head
```

Targets are similar to old "runlevels":

| Target             | Old runlevel | Meaning                  |
|--------------------|--------------|--------------------------|
| `poweroff.target`  | 0            | Shutdown                 |
| `rescue.target`    | 1            | Single-user / rescue     |
| `multi-user.target`| 3            | Multi-user, no GUI       |
| `graphical.target` | 5            | Multi-user with GUI      |
| `reboot.target`    | 6            | Reboot                   |

---

## 2. Terminal and Navigation

### 2.1 Anatomy of a prompt and command

```
user@host:~/projects$ ls -la /etc
  |    |     |         |   |    |
  |    |     |         |   |    +-- argument
  |    |     |         |   +------- option(s)
  |    |     |         +----------- command
  |    |     +--------------------- current directory (~ = home)
  |    +--------------------------- hostname
  +-------------------------------- username
```

### 2.2 `pwd`: print working directory

```bash
$ pwd
/home/alice/projects
$ pwd -P        # resolve symlinks to the physical path
```

### 2.3 `ls`: list directory contents

```bash
ls                 # names only
ls -l              # long format: perms, owner, size, date
ls -a              # include hidden files (dotfiles)
ls -lah            # long + all + human-readable sizes
ls -lt             # sort by modification time (newest first)
ls -ltr            # oldest first
ls -lS             # sort by size
ls -R              # recursive
ls -d */           # directories only
ls -i              # show inode numbers
ls -l --time-style=long-iso
```

Sample output of `ls -l`:

```
-rw-r--r-- 1 alice dev  1204 Nov 20 10:15 notes.txt
drwxr-xr-x 2 alice dev  4096 Nov 18 09:00 scripts
lrwxrwxrwx 1 alice dev    11 Nov 18 09:05 latest -> notes.txt
```

The first character is the type: `-` file, `d` directory, `l` symlink, `b` block device, `c` char device, `s` socket, `p` pipe.

### 2.4 `cd`: change directory

```bash
cd /var/log        # absolute path
cd projects        # relative path
cd ..              # up one level
cd ../..           # up two levels
cd ~               # home (same as plain `cd`)
cd -               # previous directory
cd ~bob            # bob's home directory
```

### 2.5 `tree`: directory tree view

```bash
sudo apt install tree
tree                       # current directory
tree -L 2                  # depth limit 2
tree -a -L 2               # include hidden files
tree -d                    # directories only
tree -I 'node_modules|.git'  # ignore patterns
tree -h --du               # human sizes, cumulative dir sizes
```

### 2.6 `file`: identify file type

```bash
file /bin/ls               # ELF 64-bit LSB pie executable...
file notes.txt             # ASCII text
file photo.jpg             # JPEG image data...
file -i notes.txt          # MIME type: text/plain; charset=us-ascii
```

### 2.7 `which`, `whereis`, `type`: find commands

```bash
which python3              # /usr/bin/python3 (first match in PATH)
which -a python3           # all matches in PATH
whereis ls                 # binary, source, man page locations
type ls                    # alias, builtin, or file?
type cd                    # cd is a shell builtin
command -v git             # portable way to test if a command exists
```

### 2.8 `man`, `help`, `info`, `apropos`: documentation

```bash
man ls                     # manual page (q to quit, / to search)
man 5 passwd               # section 5 = file formats
man -k "copy file"         # search descriptions (same as apropos)
apropos network
ls --help | less           # quick usage for most commands
help cd                    # help for shell builtins
info coreutils             # GNU info docs
tldr tar                   # community cheat sheets (apt install tldr)
```

Man sections: 1 user commands, 2 syscalls, 3 library calls, 4 devices, 5 file formats, 6 games, 7 misc, 8 admin commands.

### 2.9 `history`: command history

```bash
history                    # numbered list
history 20                 # last 20 commands
!!                         # repeat last command
sudo !!                    # repeat last command with sudo
!42                        # run history entry 42
!grep                      # run most recent command starting with grep
!$                         # last argument of previous command
Ctrl+R                     # reverse-search history interactively
history -c                 # clear session history
```

History settings (in `~/.bashrc`):

```bash
HISTSIZE=10000
HISTFILESIZE=20000
HISTCONTROL=ignoreboth     # ignore duplicates and lines starting with space
HISTTIMEFORMAT="%F %T "    # timestamps
```

### 2.10 Useful keyboard shortcuts

| Keys     | Action                        |
|----------|-------------------------------|
| `Tab`    | Autocomplete                  |
| `Ctrl+C` | Interrupt running command     |
| `Ctrl+D` | End of input / logout         |
| `Ctrl+Z` | Suspend job                   |
| `Ctrl+L` | Clear screen                  |
| `Ctrl+A` / `Ctrl+E` | Start / end of line|
| `Ctrl+U` / `Ctrl+K` | Cut to start / end |
| `Ctrl+W` | Delete previous word          |
| `Alt+.`  | Insert last argument          |

---

## 3. Files and Directories

### 3.1 `touch`: create empty files / update timestamps

```bash
touch a.txt                       # create or update mtime
touch file{1..5}.txt              # file1.txt ... file5.txt
touch -d "2026-01-01 12:00" a.txt # set a specific time
touch -r a.txt b.txt              # copy a.txt's timestamp to b.txt
```

### 3.2 `mkdir`: create directories

```bash
mkdir logs
mkdir -p project/{src,docs,tests}   # create nested/multiple dirs
mkdir -m 700 private                # set mode at creation
mkdir -v data                       # verbose
```

### 3.3 `cp`: copy

```bash
cp a.txt b.txt                # copy file
cp a.txt dir/                 # copy into directory
cp -r project project.bak     # recursive (directories)
cp -a project project.bak     # archive: preserves perms, times, symlinks
cp -i a.txt b.txt             # prompt before overwrite
cp -u src/* dest/             # copy only newer files
cp -v *.log backup/           # verbose
cp --backup=numbered a b      # keep numbered backups of overwritten files
```

### 3.4 `mv`: move / rename

```bash
mv old.txt new.txt            # rename
mv report.txt ~/Documents/    # move
mv -i a b                     # prompt before overwriting
mv -n a b                     # never overwrite
mv *.jpg photos/              # move many
```

### 3.5 `rm` and `rmdir`: delete

> **Warning:** There is no recycle bin. `rm` is permanent.

```bash
rm file.txt
rm -i file.txt                # ask first
rm -f file.txt                # force, no prompt, no error if missing
rm -r old_dir                 # recursive
rm -rf build/                 # recursive + force (be very careful!)
rmdir empty_dir               # only removes empty directories
rm -- -weird-name             # '--' ends options; deletes a file named -weird-name
```

Safer habit: preview with `ls` or `echo` first.

```bash
ls *.tmp          # check what the glob matches
rm *.tmp
```

Never run `rm -rf /` or `rm -rf /*`. Be careful with variables: `rm -rf "$DIR/"` with an empty `$DIR` becomes `rm -rf /`. Use `${DIR:?}` to abort if unset: `rm -rf "${DIR:?}/"`.

### 3.6 `ln`: hard and symbolic links

```bash
ln file.txt hard.txt          # hard link: same inode
ln -s /var/log/syslog logs    # symbolic link (pointer to a path)
ls -li file.txt hard.txt      # same inode number for hard links
readlink -f logs              # resolve to the final target
```

| Feature                  | Hard link | Symbolic link |
|--------------------------|-----------|---------------|
| Points to                | inode     | path          |
| Works across filesystems | No        | Yes           |
| Links to directories     | No        | Yes           |
| Breaks if target deleted | No (data remains while any link exists) | Yes (dangling) |

### 3.7 `find`: search the filesystem in real time

```bash
find . -name "*.log"                    # by name (case-sensitive)
find . -iname "readme*"                 # case-insensitive
find /var -type f -size +100M           # files larger than 100 MB
find . -type d -name node_modules       # directories
find . -mtime -1                        # modified in last 24 h
find . -mtime +30                       # modified more than 30 days ago
find . -mmin -10                        # modified in last 10 minutes
find . -user alice                      # owned by alice
find . -perm 777                        # exact permissions
find . -perm -4000                      # SUID files
find . -empty                           # empty files/dirs
find . -name "*.tmp" -delete            # delete matches
find . -name "*.sh" -exec chmod +x {} \;        # run command per file
find . -name "*.log" -exec gzip {} +            # batch execution (faster)
find . -type f -print0 | xargs -0 wc -l         # safe with spaces in names
find . -name "*.py" -not -path "./venv/*"       # exclude a path
find . -maxdepth 2 -type f                      # limit depth
```

### 3.8 `locate`: fast indexed search

```bash
sudo apt install plocate       # or mlocate
sudo updatedb                  # refresh the index
locate nginx.conf
locate -i "*.PDF"
locate -c ".conf"              # count matches
```

`locate` is much faster than `find`, but results may be out of date until `updatedb` runs.

### 3.9 `stat`: detailed file metadata

```bash
$ stat notes.txt
  File: notes.txt
  Size: 1204        Blocks: 8          IO Block: 4096   regular file
Device: 801h/2049d  Inode: 131090      Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1000/ alice)   Gid: ( 1000/ alice)
Access: 2026-11-20 10:20:01
Modify: 2026-11-20 10:15:44
Change: 2026-11-20 10:15:44
 Birth: 2026-11-18 09:00:12
```

```bash
stat -c '%n %s bytes, mode %a, owner %U' notes.txt
```

Three timestamps: **atime** (last read), **mtime** (content changed), **ctime** (metadata changed).

### 3.10 Globbing (wildcards)

```bash
ls *.txt             # any chars
ls file?.txt         # exactly one char
ls file[1-3].txt     # range
ls file[!0-9].txt    # negation
ls {a,b,c}.txt       # brace expansion -> a.txt b.txt c.txt
```

### 3.11 Archives and compression

```bash
tar -czf backup.tar.gz project/      # create gzip archive
tar -xzf backup.tar.gz               # extract
tar -xzf backup.tar.gz -C /tmp       # extract into /tmp
tar -tf backup.tar.gz                # list contents
tar -cJf backup.tar.xz project/      # xz compression
zip -r project.zip project/
unzip project.zip -d out/
gzip file.txt        # -> file.txt.gz
gunzip file.txt.gz
```

---

## 4. Text Processing

Sample file `data.csv` used in the examples:

```
id,name,dept,salary
1,alice,eng,5000
2,bob,ops,4200
3,carol,eng,5500
4,dave,hr,3900
5,erin,ops,4200
```

### 4.1 `cat`: print / concatenate

```bash
cat data.csv
cat -n data.csv                  # number lines
cat a.txt b.txt > combined.txt   # concatenate
cat >> notes.txt <<'EOF'
appended line
EOF
```

### 4.2 `less`: page through a file

```bash
less /var/log/syslog
```

Inside `less`: `Space` next page, `b` back, `/word` search, `n` next match, `G` end, `g` start, `F` follow (like tail -f), `q` quit. Use `less -N` for line numbers and `less -S` to disable wrapping.

### 4.3 `head` and `tail`

```bash
head data.csv              # first 10 lines
head -n 3 data.csv         # first 3 lines
head -c 100 file           # first 100 bytes
tail -n 2 data.csv         # last 2 lines
tail -n +2 data.csv        # everything from line 2 (skip header)
tail -f /var/log/syslog    # follow new lines live
tail -F app.log            # follow even if file is rotated
```

### 4.4 `grep`: search text with patterns

```bash
grep "eng" data.csv                   # lines containing "eng"
grep -i "ALICE" data.csv              # case-insensitive
grep -v "eng" data.csv                # invert match
grep -n "ops" data.csv                # show line numbers
grep -c "ops" data.csv                # count matching lines
grep -w "ops" data.csv                # whole word
grep -r "TODO" ./src                  # recursive
grep -rn --include="*.py" "import os" .   # recursive, only .py files
grep -l "error" *.log                 # list files that match
grep -A2 -B1 "ERROR" app.log          # 2 lines after, 1 before
grep -E "ops|hr" data.csv             # extended regex (alternation)
grep -E "^[0-9]+,a" data.csv          # lines starting with digits then ,a
grep -o "[0-9]\+" data.csv            # print only matched parts
grep -P '\d{4}' file                  # Perl regex
```

Regex quick reference: `.` any char, `*` zero+ , `+` one+ (ERE), `?` optional (ERE), `^` start, `$` end, `[abc]` set, `\b` word boundary, `(a|b)` group.

### 4.5 `cut`: extract columns

```bash
cut -d, -f2 data.csv           # name column
cut -d, -f2,4 data.csv         # name and salary
cut -d, -f2-4 data.csv         # fields 2 to 4
cut -c1-5 data.csv             # first 5 characters
cut -d: -f1,7 /etc/passwd      # username and shell
```

### 4.6 `sort`

```bash
sort names.txt
sort -r names.txt                      # reverse
sort -n numbers.txt                    # numeric
sort -h sizes.txt                      # human numeric (1K, 2M)
sort -u names.txt                      # unique
sort -t, -k4 -n data.csv               # sort CSV by 4th field numerically
sort -t, -k4 -nr data.csv              # descending
sort -t, -k3,3 -k4,4nr data.csv        # dept asc, then salary desc
du -sh * | sort -h                     # largest directories last
```

### 4.7 `uniq`: collapse adjacent duplicates (sort first!)

```bash
sort names.txt | uniq             # unique lines
sort names.txt | uniq -c          # count occurrences
sort names.txt | uniq -d          # only duplicated lines
sort names.txt | uniq -u          # only lines that appear once
```

Classic "top N" idiom:

```bash
cut -d, -f3 data.csv | tail -n +2 | sort | uniq -c | sort -nr
#   2 ops
#   2 eng
#   1 hr
```

### 4.8 `wc`: count

```bash
wc data.csv            # lines words bytes
wc -l data.csv         # lines
wc -w data.csv         # words
wc -c data.csv         # bytes
wc -m data.csv         # characters
ls | wc -l             # how many entries in the directory
```

### 4.9 `tr`: translate / delete characters

```bash
echo "hello" | tr 'a-z' 'A-Z'         # HELLO
echo "a,b,c" | tr ',' '\n'            # split to lines
echo "hello   world" | tr -s ' '      # squeeze repeated spaces
echo "abc123" | tr -d '0-9'           # delete digits -> abc
cat file | tr -d '\r'                 # strip Windows CR characters
```

### 4.10 `sed`: stream editor

```bash
sed 's/ops/operations/' data.csv         # replace first match per line
sed 's/e/E/g' data.csv                   # replace all matches
sed -n '2,4p' data.csv                   # print lines 2-4
sed '1d' data.csv                        # delete line 1 (header)
sed '/hr/d' data.csv                     # delete lines matching hr
sed -i 's/foo/bar/g' config.txt          # in-place edit
sed -i.bak 's/foo/bar/g' config.txt      # in-place edit with backup
sed 's/^/> /' file                       # prefix every line
sed -E 's/([0-9]+)/<\1>/g' file          # capture groups
sed '3i\inserted before line 3' file     # insert
sed '$a\appended at end' file            # append after last line
```

### 4.11 `awk`: field-oriented programming

```bash
awk -F, '{print $2}' data.csv                     # print name column
awk -F, 'NR>1 {print $2, $4}' data.csv            # skip header
awk -F, '$4 > 4500' data.csv                      # filter rows by condition
awk -F, '$3=="eng" {print $2}' data.csv           # filter by text
awk -F, 'NR>1 {sum+=$4} END {print "Total:", sum}' data.csv
awk -F, 'NR>1 {sum+=$4; n++} END {print "Avg:", sum/n}' data.csv
awk -F, 'NR>1 {c[$3]++} END {for (d in c) print d, c[d]}' data.csv
awk -F, 'BEGIN{OFS="\t"} {print $1,$2}' data.csv  # change output separator
awk '{print NR": "$0}' file                       # number lines
awk 'length($0) > 80' file                        # long lines
```

Built-ins: `NR` record number, `NF` number of fields, `FS` input separator, `OFS` output separator, `$0` whole line.

### 4.12 Pipelines and redirection

```bash
cmd > out.txt          # stdout to file (overwrite)
cmd >> out.txt         # append
cmd 2> err.txt         # stderr to file
cmd > all.txt 2>&1     # both to file
cmd &> all.txt         # shorthand (bash)
cmd < input.txt        # stdin from file
cmd1 | cmd2            # pipe stdout into stdin
cmd | tee out.txt      # display AND save
cmd > /dev/null 2>&1   # discard everything
```

Real-world example, top 5 IPs in an nginx access log:

```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -5
```

Count failed SSH logins per source IP:

```bash
sudo grep "Failed password" /var/log/auth.log \
  | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr | head
```

---

## 5. Permissions and Ownership

### 5.1 Reading permissions

```
-rwxr-xr--  1  alice  dev  4096  Nov 20  script.sh
 |||||||||
 |||||||++-- others: r--  (read)
 ||||||+---- group:  r-x  (read, execute)
 |||+++----- owner:  rwx  (read, write, execute)
 +---------- type
```

| Bit | Value | On a file       | On a directory                      |
|-----|-------|----------------|-------------------------------------|
| `r` | 4     | read contents  | list names (`ls`)                   |
| `w` | 2     | modify contents| create/delete/rename entries inside |
| `x` | 1     | execute        | enter (`cd`) and access entries     |

`rwxr-xr--` = 7 5 4 = **754**.

### 5.2 `chmod`: change mode

Symbolic:

```bash
chmod u+x script.sh          # add execute for owner
chmod g-w file               # remove write from group
chmod o= file                # remove all perms for others
chmod a+r file               # add read for all
chmod u=rwx,g=rx,o=r file    # set exactly
chmod -R g+rwX project/      # recursive; X = execute only on dirs/already-exec files
```

Numeric:

```bash
chmod 644 file       # rw-r--r--  (typical file)
chmod 755 script.sh  # rwxr-xr-x  (typical script/dir)
chmod 600 id_rsa     # rw-------  (private keys)
chmod 700 ~/private  # rwx------
chmod 400 secret     # r--------
```

### 5.3 `chown` and `chgrp`: change ownership

```bash
sudo chown alice file
sudo chown alice:dev file            # owner and group
sudo chown :dev file                 # group only
sudo chown -R www-data:www-data /var/www/site
sudo chgrp dev project/
sudo chown --reference=a.txt b.txt   # copy ownership from another file
```

### 5.4 `umask`: default permission mask

New files start at `666`, new directories at `777`, minus the umask.

```bash
umask            # e.g. 0022
umask -S         # symbolic: u=rwx,g=rx,o=rx
umask 077        # files -> 600, dirs -> 700 (private)
```

| umask | New file | New dir |
|-------|----------|---------|
| 022   | 644      | 755     |
| 027   | 640      | 750     |
| 077   | 600      | 700     |

Make it permanent in `~/.bashrc` or `/etc/profile`.

### 5.5 `sudo`: run as another user

```bash
sudo apt update                 # run as root
sudo -u postgres psql           # run as another user
sudo -i                         # root login shell
sudo -l                         # list what you may run
sudo !!                         # re-run last command with sudo
sudo visudo                     # safely edit /etc/sudoers
```

Example sudoers entries (always edit with `visudo`, or drop files in `/etc/sudoers.d/`):

```
alice   ALL=(ALL:ALL) ALL                       # full sudo
%admins ALL=(ALL) ALL                           # group admins
bob     ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx
```

### 5.6 Special permission bits

| Bit        | Octal | Effect                                                                 |
|------------|-------|------------------------------------------------------------------------|
| **SUID**   | 4000  | Executable runs with the *file owner's* privileges (`s` in owner x slot)|
| **SGID**   | 2000  | Executable runs with group's privileges; on a **directory**, new files inherit the directory's group |
| **Sticky** | 1000  | On a directory, only file owner (or dir owner/root) can delete/rename files (`t` in others x slot)|

```bash
ls -l /usr/bin/passwd      # -rwsr-xr-x  (SUID root: lets users change own password)
ls -ld /tmp                # drwxrwxrwt  (sticky bit)

chmod u+s program          # set SUID   (or chmod 4755 program)
chmod g+s shared_dir       # set SGID   (or chmod 2775 shared_dir)
chmod +t shared_dir        # set sticky (or chmod 1777 shared_dir)

find / -perm -4000 -type f 2>/dev/null     # audit SUID binaries
find / -perm -2000 -type f 2>/dev/null     # audit SGID binaries
```

Shared team directory example:

```bash
sudo groupadd devs
sudo mkdir /srv/team
sudo chgrp devs /srv/team
sudo chmod 2775 /srv/team      # SGID: new files inherit group "devs"
```

A capital `S` or `T` in `ls` output means the special bit is set but the underlying execute bit is not (usually a mistake).

### 5.7 ACLs: fine-grained access control

When owner/group/others is too coarse:

```bash
sudo apt install acl
getfacl report.txt                       # show ACLs
setfacl -m u:bob:rw report.txt           # give bob read/write
setfacl -m g:auditors:r report.txt       # give group read
setfacl -x u:bob report.txt              # remove bob's entry
setfacl -b report.txt                    # remove all ACLs
setfacl -R -m u:bob:rwX project/         # recursive
setfacl -d -m g:devs:rwx project/        # default ACL: inherited by new files
```

A `+` at the end of the permission string (`-rw-rw-r--+`) means ACLs exist. The filesystem must be mounted with ACL support (default on ext4/xfs).

### 5.8 Attributes (bonus)

```bash
sudo chattr +i important.conf    # immutable: cannot be modified or deleted, even by root
lsattr important.conf
sudo chattr -i important.conf
```

---

## 6. Users and Groups

### 6.1 Account files

| File           | Content                                      |
|----------------|----------------------------------------------|
| `/etc/passwd`  | User accounts (world-readable)               |
| `/etc/shadow`  | Password hashes and aging (root-only)        |
| `/etc/group`   | Groups and members                           |
| `/etc/gshadow` | Group passwords/admins                       |
| `/etc/login.defs` | Defaults (UID ranges, password aging)     |
| `/etc/skel`    | Template files copied into new home dirs     |

`/etc/passwd` format (7 fields separated by `:`):

```
alice:x:1000:1000:Alice Smith,,,:/home/alice:/bin/bash
  |   | |    |         |            |           |
  |   | |    |         |            |           +-- login shell
  |   | |    |         |            +-------------- home directory
  |   | |    |         +--------------------------- GECOS (full name/info)
  |   | |    +------------------------------------- primary GID
  |   | +------------------------------------------ UID
  |   +-------------------------------------------- 'x' = hash is in /etc/shadow
  +------------------------------------------------ username
```

`/etc/shadow` format (excerpt):

```
alice:$y$j9T$...hash...:19700:0:99999:7:::
  |        |             |    | |     |
  |        |             |    | |     +-- warning days
  |        |             |    | +-------- max age (days)
  |        |             |    +---------- min age (days)
  |        |             +--------------- last change (days since 1970-01-01)
  |        +----------------------------- hashed password ('!' or '*' = locked/no login)
  +-------------------------------------- username
```

UID ranges: `0` = root, `1-999` = system accounts, `1000+` = regular users.

### 6.2 `useradd` / `adduser`

```bash
sudo useradd -m -s /bin/bash -c "Bob Jones" bob     # -m creates home
sudo useradd -m -G sudo,dev -s /bin/bash carol      # with supplementary groups
sudo useradd -r -s /usr/sbin/nologin svc_app        # system (service) account
sudo useradd -u 1500 -g dev -d /srv/dave dave       # explicit UID/group/home
sudo adduser erin                                   # Debian: friendly interactive wrapper
```

### 6.3 `passwd`

```bash
passwd                      # change your own password
sudo passwd bob             # set bob's password
sudo passwd -l bob          # lock account
sudo passwd -u bob          # unlock
sudo passwd -e bob          # expire password (force change at next login)
sudo passwd -S bob          # status
```

### 6.4 `chage`: password aging

```bash
sudo chage -l bob                    # show aging info
sudo chage -M 90 -W 7 bob            # max 90 days, warn 7 days before
sudo chage -E 2026-12-31 bob         # account expires on a date
sudo chage -d 0 bob                  # force password change at next login
```

### 6.5 `usermod`

```bash
sudo usermod -aG docker alice        # ADD to group (always use -a with -G!)
sudo usermod -s /bin/zsh alice       # change shell
sudo usermod -l alicia alice         # rename login
sudo usermod -d /new/home -m alice   # move home directory
sudo usermod -L alice                # lock
sudo usermod -U alice                # unlock
sudo usermod -e 2026-12-31 alice     # set expiry
```

> **Warning:** `usermod -G docker alice` (without `-a`) *replaces* all supplementary groups.

### 6.6 `userdel`

```bash
sudo userdel bob            # remove account, keep home
sudo userdel -r bob         # remove account and home directory
sudo deluser --remove-home bob   # Debian equivalent
```

### 6.7 Groups: `groupadd`, `groupmod`, `groupdel`, `gpasswd`

```bash
sudo groupadd devs
sudo groupadd -g 2000 qa              # specific GID
sudo groupmod -n developers devs      # rename
sudo gpasswd -a alice devs            # add alice to devs
sudo gpasswd -d alice devs            # remove alice
sudo groupdel qa
```

### 6.8 `id`, `groups`, `whoami`, `who`, `w`, `last`

```bash
$ id
uid=1000(alice) gid=1000(alice) groups=1000(alice),27(sudo),999(docker)

id bob                 # info about another user
id -u                  # just the UID
groups alice           # groups for alice
whoami                 # current effective user
who                    # who is logged in
w                      # who is logged in and what they're doing
last -n 10             # recent logins
lastb                  # failed logins (root)
getent passwd alice    # lookup (works with LDAP/SSSD too)
getent group sudo
```

Group membership changes take effect on a **new login**; use `newgrp devs` or log out/in.

### 6.9 Switching users

```bash
su - bob               # become bob with bob's environment (login shell)
su bob                 # become bob, keep current environment
sudo -iu bob           # same as su - bob, via sudo
exit                   # return
```

### 6.10 Practical: create a team workspace

```bash
sudo groupadd webteam
sudo useradd -m -s /bin/bash -G webteam alice
sudo useradd -m -s /bin/bash -G webteam bob
sudo mkdir -p /srv/web
sudo chown root:webteam /srv/web
sudo chmod 2770 /srv/web         # group rwx, SGID, no access for others
sudo -u alice touch /srv/web/index.html
ls -l /srv/web                   # file inherits group webteam
```

---

## 7. Processes and Services

### 7.1 Concepts

- A **process** is a running instance of a program with a **PID**; every process has a parent (**PPID**).
- PID 1 is `systemd` (init). Orphans are re-parented to it.
- States: **R** running, **S** sleeping (interruptible), **D** uninterruptible I/O wait, **T** stopped, **Z** zombie.

### 7.2 `ps`: snapshot of processes

```bash
ps                           # processes in this terminal
ps aux                       # all processes, BSD style
ps -ef                       # all processes, full format (shows PPID)
ps -ef --forest              # tree view
ps -u alice                  # processes of a user
ps -p 1234 -o pid,ppid,user,%cpu,%mem,etime,cmd
ps aux --sort=-%mem | head   # top memory consumers
ps aux --sort=-%cpu | head   # top CPU consumers
pgrep -a nginx               # find PIDs by name
pidof sshd
```

### 7.3 `top` and `htop`: live view

```bash
top
```

Inside `top`: `P` sort by CPU, `M` by memory, `k` kill, `1` per-CPU view, `u` filter user, `q` quit.

Header example: `load average: 0.52, 0.48, 0.45`. These are 1/5/15 minute averages; compare to your CPU core count (`nproc`).

```bash
sudo apt install htop
htop                      # colored, mouse support; F6 sort, F9 kill, F5 tree
```

Others: `uptime`, `free -h`, `vmstat 1`, `iostat`, `pidstat`.

### 7.4 Signals: `kill`, `pkill`, `killall`

```bash
kill -l                    # list signals
kill 1234                  # SIGTERM (15): polite request to terminate
kill -9 1234               # SIGKILL: cannot be caught; last resort
kill -HUP 1234             # SIGHUP: often means "reload config"
kill -STOP 1234            # pause
kill -CONT 1234            # resume
pkill nginx                # by name
pkill -u alice             # all of alice's processes
pkill -f "python app.py"   # match full command line
killall firefox
```

| Signal    | Number | Meaning                          |
|-----------|--------|----------------------------------|
| SIGHUP    | 1      | Hangup / reload                  |
| SIGINT    | 2      | Interrupt (Ctrl+C)               |
| SIGQUIT   | 3      | Quit with core dump              |
| SIGKILL   | 9      | Force kill (uncatchable)         |
| SIGTERM   | 15     | Graceful terminate (default)     |
| SIGSTOP   | 19     | Pause (uncatchable)              |
| SIGCONT   | 18     | Continue                         |

Try SIGTERM first and give the process time to clean up before using `-9`.

### 7.5 Priority: `nice` and `renice`

Nice values range from **-20** (highest priority) to **19** (lowest). Default is 0. Only root can lower the value.

```bash
nice -n 10 ./backup.sh              # start with lower priority
sudo nice -n -5 ./important.sh      # higher priority
renice -n 15 -p 1234                # change a running process
sudo renice -n -5 -p 1234
ionice -c3 ./heavy_io_job           # idle I/O class
```

### 7.6 Job control: `&`, `jobs`, `bg`, `fg`

```bash
sleep 300 &              # run in background
jobs                     # [1]+ Running  sleep 300 &
fg %1                    # bring job 1 to foreground
# press Ctrl+Z           # suspend foreground job
bg %1                    # continue it in background
kill %1                  # kill job 1
disown %1                # detach from the shell (survives logout)
nohup ./long.sh > out.log 2>&1 &   # immune to hangup
```

For long-lived sessions, use `tmux` or `screen`:

```bash
tmux new -s work         # start; detach with Ctrl+B then D
tmux ls
tmux attach -t work
```

### 7.7 `systemctl`: manage services

```bash
systemctl status nginx              # state, recent logs, PID
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx        # stop + start
sudo systemctl reload nginx         # reload config without dropping connections
sudo systemctl enable nginx         # start at boot
sudo systemctl disable nginx
sudo systemctl enable --now nginx   # enable and start immediately
sudo systemctl mask nginx           # prevent starting at all
sudo systemctl unmask nginx
systemctl is-active nginx
systemctl is-enabled nginx
systemctl list-units --type=service --state=failed
sudo systemctl daemon-reload        # after editing unit files
systemctl cat nginx                 # show the unit file
sudo systemctl edit nginx           # create a drop-in override
```

### 7.8 Writing a custom service

`/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=My Python App
After=network.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/python3 /opt/myapp/app.py
Restart=on-failure
RestartSec=5
Environment=APP_ENV=production

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
systemctl status myapp
journalctl -u myapp -f
```

### 7.9 `journalctl`: read the systemd journal

```bash
journalctl                           # everything (oldest first)
journalctl -e                        # jump to end
journalctl -f                        # follow live
journalctl -u nginx                  # one unit
journalctl -u nginx -f --since "10 min ago"
journalctl --since "2026-11-23 08:00" --until "2026-11-23 09:00"
journalctl -p err                    # priority err and worse
journalctl -p warning..err
journalctl -b                        # this boot
journalctl -b -1                     # previous boot
journalctl -k                        # kernel messages
journalctl _UID=1000                 # by user id
journalctl -o json-pretty -n 1       # JSON output
journalctl --disk-usage
sudo journalctl --vacuum-time=7d     # keep only 7 days
sudo journalctl --vacuum-size=500M
```

### 7.10 Troubleshooting a service

```bash
systemctl status myapp               # 1. is it running? exit code?
journalctl -u myapp -n 50 --no-pager # 2. recent logs
sudo ss -tulpn | grep :8080          # 3. is the port in use / listening?
sudo lsof -i :8080                   # 4. which process owns it?
systemctl list-dependencies myapp    # 5. dependency issues
```

---

## 8. Storage and Filesystems

### 8.1 `df`: filesystem free space

```bash
df -h                  # human readable
df -hT                 # include filesystem type
df -i                  # inode usage
df -h /var             # specific path
```

### 8.2 `du`: directory usage

```bash
du -sh /var/log                    # total size of a directory
du -h --max-depth=1 /var | sort -h # size per subdirectory
du -sh * | sort -h                 # biggest items in cwd
du -ah /var/log | sort -rh | head  # largest files/dirs
sudo ncdu /                        # interactive (apt install ncdu)
```

Disk full? Find out why:

```bash
df -h
sudo du -xh / --max-depth=2 2>/dev/null | sort -rh | head -20
df -i                                   # maybe out of inodes instead
sudo lsof +L1                           # deleted files still held open
```

### 8.3 `lsblk`, `blkid`, `fdisk -l`

```bash
lsblk
# NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
# sda      8:0    0   50G  0 disk
# ├─sda1   8:1    0    1G  0 part /boot/efi
# └─sda2   8:2    0   49G  0 part /
# sdb      8:16   0  100G  0 disk

lsblk -f               # filesystems, labels, UUIDs
sudo blkid             # UUIDs and types
sudo fdisk -l          # partition tables
```

### 8.4 Partitioning and formatting a new disk

> **Warning:** these steps destroy data on the target disk. Double-check the device name (`/dev/sdb` below is an example).

```bash
sudo fdisk /dev/sdb
#  n  -> new partition (accept defaults)
#  w  -> write and exit
# (use gdisk or parted for GPT / >2 TB disks)

sudo mkfs.ext4 /dev/sdb1          # create ext4 filesystem
sudo mkfs.xfs  /dev/sdb1          # or XFS
sudo e2label /dev/sdb1 data       # optional label

sudo mkdir -p /mnt/data
sudo mount /dev/sdb1 /mnt/data
df -h /mnt/data
```

### 8.5 `mount` and `umount`

```bash
mount                                # list mounts
findmnt                              # tree view
sudo mount /dev/sdb1 /mnt/data
sudo mount -o ro /dev/sdb1 /mnt/data # read-only
sudo mount -o remount,rw /           # change options on the fly
sudo mount -t nfs server:/share /mnt/nfs
sudo mount -o loop disk.iso /mnt/iso # mount an ISO image
sudo umount /mnt/data                # unmount by mountpoint
sudo umount -l /mnt/data             # lazy unmount if busy
lsof +f -- /mnt/data                 # who is using it?
fuser -vm /mnt/data
```

### 8.6 `/etc/fstab`: persistent mounts

Format: `device  mountpoint  type  options  dump  fsck-order`

```
# <file system>                          <mount point> <type> <options>        <dump> <pass>
UUID=3f1c2a9e-1b7d-4c2a-9a0b-1234abcd5678  /           ext4   errors=remount-ro  0      1
UUID=9a8b7c6d-1111-2222-3333-444455556666  /mnt/data   ext4   defaults,noatime   0      2
server:/export/share                       /mnt/nfs    nfs    defaults,_netdev   0      0
tmpfs                                      /tmp        tmpfs  defaults,size=1G   0      0
```

- Use **UUIDs** (from `blkid`) rather than `/dev/sdX`, which can change.
- `pass`: 1 for root, 2 for others, 0 to skip fsck.
- `nofail` prevents a missing disk from blocking boot.

Always test before rebooting:

```bash
sudo mount -a          # mount everything in fstab; errors appear now, not at boot
findmnt --verify       # check fstab syntax
```

### 8.7 Inodes

An **inode** stores a file's metadata (owner, perms, timestamps, block pointers), but not its name. Directories map names to inode numbers.

```bash
ls -i file.txt
stat file.txt
df -i                  # inode usage per filesystem
```

If `df -h` shows free space but you get "No space left on device", you likely ran out of inodes (millions of tiny files). Find the culprit:

```bash
sudo find / -xdev -type f | cut -d/ -f2-3 | sort | uniq -c | sort -nr | head
```

### 8.8 Swap

```bash
swapon --show
free -h

# Create a 2 GB swap file
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

sudo swapoff /swapfile              # disable

cat /proc/sys/vm/swappiness         # default 60
sudo sysctl vm.swappiness=10        # prefer RAM more (runtime)
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf  # persistent
```

### 8.9 LVM basics

LVM adds flexible layers: **PV** (physical volume) -> **VG** (volume group) -> **LV** (logical volume).

```
/dev/sdb  /dev/sdc          (disks)
    \       /
     PV   PV                pvcreate
       VG  "vg_data"        vgcreate
     /     \
  LV data  LV logs          lvcreate  -> /dev/vg_data/data
```

```bash
sudo apt install lvm2

sudo pvcreate /dev/sdb /dev/sdc               # initialize disks
sudo vgcreate vg_data /dev/sdb /dev/sdc       # create volume group
sudo lvcreate -L 50G -n lv_data vg_data       # 50 GB logical volume
sudo lvcreate -l 100%FREE -n lv_logs vg_data  # use all remaining space

sudo mkfs.ext4 /dev/vg_data/lv_data
sudo mkdir -p /data
sudo mount /dev/vg_data/lv_data /data

# Inspect
sudo pvs ; sudo vgs ; sudo lvs
sudo pvdisplay ; sudo vgdisplay ; sudo lvdisplay

# Grow a volume and its filesystem online
sudo lvextend -L +10G /dev/vg_data/lv_data
sudo resize2fs /dev/vg_data/lv_data            # ext4
# or in one step:
sudo lvextend -r -L +10G /dev/vg_data/lv_data  # -r resizes the filesystem too
# XFS: sudo xfs_growfs /data

# Add a new disk to the pool
sudo pvcreate /dev/sdd
sudo vgextend vg_data /dev/sdd

# Snapshot
sudo lvcreate -s -n snap1 -L 5G /dev/vg_data/lv_data
```

### 8.9 Checking and repairing filesystems

```bash
sudo umount /dev/sdb1          # must be unmounted
sudo fsck /dev/sdb1            # check (and prompt to fix)
sudo fsck -y /dev/sdb1         # auto-answer yes
sudo e2fsck -f /dev/sdb1       # force check ext4
sudo smartctl -a /dev/sda      # disk health (apt install smartmontools)
```

---

## 9. Package Management

### 9.1 `apt`: high-level package manager (Debian/Ubuntu)

```bash
sudo apt update                       # refresh package index
sudo apt upgrade                      # upgrade installed packages
sudo apt full-upgrade                 # upgrade, allowing removals to resolve deps
sudo apt install nginx                # install
sudo apt install nginx=1.24.0-2ubuntu7   # specific version
sudo apt install -y curl git          # no prompt
sudo apt install --no-install-recommends vim
sudo apt remove nginx                 # remove, keep config
sudo apt purge nginx                  # remove including config
sudo apt autoremove                   # remove unused dependencies
sudo apt clean                        # clear downloaded .deb cache

apt search "web server"
apt show nginx                        # details, dependencies
apt list --installed | grep nginx
apt list --upgradable
apt policy nginx                      # installed vs candidate versions, sources
apt-cache depends nginx
apt-cache rdepends nginx              # reverse dependencies
sudo apt --fix-broken install         # repair broken dependencies
```

Typical workflow:

```bash
sudo apt update && sudo apt upgrade -y
```

### 9.2 `dpkg`: low-level package tool

```bash
sudo dpkg -i package.deb              # install a local .deb
sudo apt install ./package.deb        # better: resolves dependencies
dpkg -l                               # list installed packages
dpkg -l | grep nginx
dpkg -L nginx                         # files installed by a package
dpkg -S /usr/sbin/nginx               # which package owns a file
dpkg -s nginx                         # status
sudo dpkg -r nginx                    # remove
sudo dpkg -P nginx                    # purge
sudo dpkg --configure -a              # finish interrupted installs
dpkg --get-selections > pkgs.txt      # record installed packages
dpkg-deb -I package.deb               # inspect a .deb
```

### 9.3 Repositories

```bash
cat /etc/apt/sources.list
ls /etc/apt/sources.list.d/
```

Modern Ubuntu/Debian use deb822 `.sources` files, e.g. `/etc/apt/sources.list.d/ubuntu.sources`:

```
Types: deb
URIs: http://archive.ubuntu.com/ubuntu
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

Components: `main` (free, supported), `universe` (community), `restricted` (proprietary drivers), `multiverse` (non-free).

Adding a third-party repository securely (example pattern):

```bash
# 1. Download and store the signing key in a dedicated keyring
curl -fsSL https://example.com/repo/key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/example.gpg

# 2. Add the source, pinned to that key
echo "deb [signed-by=/usr/share/keyrings/example.gpg] https://example.com/repo stable main" \
  | sudo tee /etc/apt/sources.list.d/example.list

# 3. Refresh and install
sudo apt update
sudo apt install examplepkg
```

Avoid the deprecated `apt-key add`. PPAs on Ubuntu:

```bash
sudo add-apt-repository ppa:ondrej/php
sudo add-apt-repository --remove ppa:ondrej/php
```

### 9.4 Holding and pinning packages

Hold a package at its current version:

```bash
sudo apt-mark hold nginx
apt-mark showhold
sudo apt-mark unhold nginx
```

**Pinning** controls which source/version wins. Higher priority wins (default 500; 990+ preferred even over newer; negative blocks).

`/etc/apt/preferences.d/nginx-pin`:

```
Package: nginx*
Pin: version 1.24.*
Pin-Priority: 1001
```

Prefer one release but allow selected packages from another:

```
Package: *
Pin: release a=stable
Pin-Priority: 900

Package: *
Pin: release a=testing
Pin-Priority: 400
```

```bash
apt policy nginx        # verify which version/source apt will choose
sudo apt install -t testing somepkg   # one-off from another release
```

### 9.5 Automatic updates: `unattended-upgrades`

```bash
sudo apt install unattended-upgrades apt-listchanges
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

`/etc/apt/apt.conf.d/20auto-upgrades`:

```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";
```

Key options in `/etc/apt/apt.conf.d/50unattended-upgrades`:

```
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";
Unattended-Upgrade::Mail "admin@example.com";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
```

```bash
sudo unattended-upgrade --dry-run --debug     # test
cat /var/log/unattended-upgrades/unattended-upgrades.log
[ -f /var/run/reboot-required ] && echo "Reboot needed"
```

### 9.6 Other formats

```bash
sudo snap install code --classic ; snap list
flatpak install flathub org.gimp.GIMP
# Red Hat family equivalents
sudo dnf install nginx ; sudo dnf update ; rpm -qa | grep nginx
```

---

## 10. Scheduling and Logs

### 10.1 `cron`: recurring jobs

Crontab format:

```
┌──────── minute (0-59)
│ ┌────── hour (0-23)
│ │ ┌──── day of month (1-31)
│ │ │ ┌── month (1-12)
│ │ │ │ ┌ day of week (0-7, Sun = 0 or 7)
│ │ │ │ │
* * * * *  command
```

```bash
crontab -e            # edit your crontab
crontab -l            # list
crontab -r            # remove all (careful!)
sudo crontab -u bob -l
```

Examples:

```
*/5 * * * *    /opt/scripts/healthcheck.sh               # every 5 minutes
0 2 * * *      /opt/scripts/backup.sh                    # daily at 02:00
30 3 * * 0     /usr/bin/apt update                       # Sundays at 03:30
0 9-17 * * 1-5 /opt/scripts/report.sh                    # hourly 9-17, Mon-Fri
0 0 1 * *      /opt/scripts/monthly.sh                   # 1st of each month
@reboot        /opt/scripts/on_boot.sh                   # at startup
@daily         /opt/scripts/cleanup.sh
0 2 * * *      /opt/scripts/backup.sh >> /var/log/backup.log 2>&1   # capture output
```

Best practices:

- Use **absolute paths**; cron has a minimal `PATH`.
- Set variables at the top: `SHELL=/bin/bash`, `PATH=/usr/local/bin:/usr/bin:/bin`, `MAILTO=admin@example.com`.
- Redirect output, or you'll get local mail or silent failures.
- Use `flock` to avoid overlapping runs: `*/5 * * * * flock -n /tmp/job.lock /opt/job.sh`.
- Escape `%` as `\%` in crontab lines.

System-wide locations:

```
/etc/crontab                  # includes a user field
/etc/cron.d/                  # drop-in files (with user field)
/etc/cron.hourly|daily|weekly|monthly/   # scripts run by run-parts
/etc/cron.allow, /etc/cron.deny          # who may use crontab
```

Debug: `grep CRON /var/log/syslog` or `journalctl -u cron`.

### 10.2 `at`: one-time jobs

```bash
sudo apt install at
echo "/opt/scripts/deploy.sh" | at 23:30
at now + 2 hours <<< "echo hello > /tmp/hello.txt"
at 10:00 tomorrow
atq                  # list pending jobs
at -c 3              # view job 3
atrm 3               # remove job 3
```

### 10.3 systemd timers (modern alternative to cron)

`/etc/systemd/system/backup.service`:

```ini
[Unit]
Description=Nightly backup

[Service]
Type=oneshot
ExecStart=/opt/scripts/backup.sh
```

`/etc/systemd/system/backup.timer`:

```ini
[Unit]
Description=Run backup nightly

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
systemctl list-timers --all
systemd-analyze calendar "Mon..Fri 09:00"    # validate calendar expressions
journalctl -u backup.service
```

### 10.4 Where logs live

| Path / source             | Content                                     |
|---------------------------|---------------------------------------------|
| `/var/log/syslog`         | General system messages (Debian/Ubuntu)     |
| `/var/log/auth.log`       | Authentication, sudo, SSH                   |
| `/var/log/kern.log`       | Kernel messages                             |
| `/var/log/dpkg.log`, `/var/log/apt/` | Package operations               |
| `/var/log/nginx/`, `/var/log/apache2/` | Web server logs                |
| `/var/log/wtmp`, `/var/log/btmp` | Logins (`last`) / failed logins (`lastb`)|
| `/var/log/journal/`       | Persistent journald storage                 |

(On RHEL-family systems: `/var/log/messages` and `/var/log/secure`.)

### 10.5 syslog / rsyslog

Messages have a **facility** (auth, kern, daemon, mail, local0-7...) and **severity** (emerg, alert, crit, err, warning, notice, info, debug).

`/etc/rsyslog.d/50-default.conf` (excerpt):

```
auth,authpriv.*        /var/log/auth.log
*.*;auth,authpriv.none /var/log/syslog
kern.*                 -/var/log/kern.log
```

Send your own message and view it:

```bash
logger "Backup completed"
logger -p local0.err -t myapp "Disk almost full"
tail -n 3 /var/log/syslog
```

Forward logs to a central server (client side, `/etc/rsyslog.d/90-remote.conf`):

```
*.* @@logserver.example.com:514     # @@ = TCP, @ = UDP
```

```bash
sudo systemctl restart rsyslog
```

### 10.6 journald

Configuration: `/etc/systemd/journald.conf`

```
[Journal]
Storage=persistent        # keep logs across reboots (creates /var/log/journal)
SystemMaxUse=500M
MaxRetentionSec=1month
Compress=yes
```

```bash
sudo systemctl restart systemd-journald
journalctl -u ssh --since today
journalctl -p err -b
journalctl --disk-usage
```

See also section 7.9 for more `journalctl` examples.

### 10.7 `dmesg`: kernel ring buffer

```bash
dmesg                      # all kernel messages
sudo dmesg -T              # human-readable timestamps
sudo dmesg -w              # follow
sudo dmesg --level=err,warn
sudo dmesg | grep -i usb
sudo dmesg | grep -iE "error|fail|oom|segfault"
sudo dmesg -T | grep -i "out of memory"    # OOM killer events
```

Plug in a USB drive and run `dmesg -w` to see it detected (`sdb: sdb1`).

### 10.8 `logrotate`: keep logs from growing forever

Main config `/etc/logrotate.conf`; per-app files in `/etc/logrotate.d/`. Example `/etc/logrotate.d/myapp`:

```
/var/log/myapp/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 appuser adm
    sharedscripts
    postrotate
        systemctl reload myapp > /dev/null 2>&1 || true
    endscript
}
```

| Directive         | Meaning                                         |
|-------------------|-------------------------------------------------|
| `daily/weekly/monthly` | Rotation frequency                         |
| `size 100M`       | Rotate when larger than size                    |
| `rotate N`        | Keep N old logs                                 |
| `compress`        | gzip old logs (`delaycompress` waits one cycle) |
| `missingok`       | No error if the log is absent                   |
| `notifempty`      | Skip empty logs                                 |
| `copytruncate`    | Copy then truncate (for apps that can't reopen) |

```bash
sudo logrotate -d /etc/logrotate.d/myapp   # debug / dry run
sudo logrotate -f /etc/logrotate.d/myapp   # force rotation now
cat /var/lib/logrotate/status              # last rotation times
```

---

## 11. SSH and Remote Administration

### 11.1 Basic connection

```bash
ssh user@192.168.1.10
ssh -p 2222 user@host                 # non-default port
ssh user@host 'uptime; df -h'         # run a remote command
ssh -v user@host                      # verbose (-vvv for more) for debugging
ssh -i ~/.ssh/mykey user@host         # specific identity file
```

On first connect you'll be asked to verify the host fingerprint; it's stored in `~/.ssh/known_hosts`. A "REMOTE HOST IDENTIFICATION HAS CHANGED" warning means the key differs (reinstall or possible attack). If legitimate: `ssh-keygen -R host`.

### 11.2 SSH keys

```bash
ssh-keygen -t ed25519 -C "alice@laptop"        # recommended modern key type
ssh-keygen -t rsa -b 4096 -C "alice@laptop"    # if ed25519 unsupported
# creates ~/.ssh/id_ed25519 (private) and ~/.ssh/id_ed25519.pub (public)

ssh-copy-id user@host                           # install public key on server
ssh-copy-id -i ~/.ssh/mykey.pub user@host

# manual alternative
cat ~/.ssh/id_ed25519.pub | ssh user@host 'mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

Required permissions (SSH refuses to work otherwise):

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519 ~/.ssh/authorized_keys ~/.ssh/config
chmod 644 ~/.ssh/id_ed25519.pub
```

Never share your private key. Protect it with a passphrase.

```bash
ssh-keygen -p -f ~/.ssh/id_ed25519      # change passphrase
ssh-keygen -lf ~/.ssh/id_ed25519.pub    # show fingerprint
```

### 11.3 `ssh-agent`: avoid retyping passphrases

```bash
eval "$(ssh-agent -s)"            # start agent in this shell
ssh-add ~/.ssh/id_ed25519         # add key (asks passphrase once)
ssh-add -l                        # list loaded keys
ssh-add -d ~/.ssh/id_ed25519      # remove one
ssh-add -D                        # remove all
ssh -A user@bastion               # forward agent (use sparingly; prefer ProxyJump)
```

On many desktops, the agent starts automatically. Add `AddKeysToAgent yes` to your config to load keys on first use.

### 11.4 Client config: `~/.ssh/config`

```
# Defaults for all hosts
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    AddKeysToAgent yes
    IdentitiesOnly yes

Host web1
    HostName 203.0.113.10
    User deploy
    Port 2222
    IdentityFile ~/.ssh/id_ed25519

Host db-internal
    HostName 10.0.0.5
    User admin
    ProxyJump bastion

Host bastion
    HostName bastion.example.com
    User alice
    IdentityFile ~/.ssh/bastion_key
```

Now you can simply run:

```bash
ssh web1
ssh db-internal        # automatically hops through bastion
scp file.txt web1:/tmp/
```

### 11.5 Server config: `/etc/ssh/sshd_config`

Recommended hardening:

```
Port 22
PermitRootLogin no
PasswordAuthentication no        # key-only login (ensure your key works first!)
PubkeyAuthentication yes
PermitEmptyPasswords no
AllowUsers alice bob             # or AllowGroups sshusers
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
```

Always validate and keep an existing session open while testing:

```bash
sudo sshd -t                       # syntax check
sudo systemctl reload ssh          # service is "sshd" on RHEL-family
# In a NEW terminal, confirm you can still log in before closing the old one.
```

Extras: `fail2ban` to block brute force, change firewall rules with `ufw allow 22/tcp`, consider `ssh-audit`.

### 11.6 `scp`: simple copy over SSH

```bash
scp file.txt user@host:/tmp/                  # upload
scp user@host:/var/log/app.log .              # download
scp -r project/ user@host:/srv/               # directory
scp -P 2222 file user@host:~                  # port is -P (capital) for scp
scp -3 user@a:/file user@b:/dest              # copy between two remote hosts via local
scp -J bastion file.txt user@internal:/tmp    # jump host
```

### 11.7 `sftp`: interactive file transfer

```bash
sftp user@host
sftp> ls
sftp> cd /var/www
sftp> lcd ~/Downloads        # local cd
sftp> get backup.tar.gz
sftp> put index.html
sftp> mget *.log
sftp> mkdir uploads
sftp> bye
```

### 11.8 `rsync`: efficient synchronization

Only transfers differences, resumable, and the best tool for backups and deployments.

```bash
rsync -avz src/ user@host:/backup/src/           # archive, verbose, compress
rsync -avzP big.iso user@host:/data/             # progress + resume partial files
rsync -av --delete src/ dest/                    # mirror: remove files not in source
rsync -av --dry-run --delete src/ dest/          # ALWAYS preview deletions first
rsync -av --exclude='*.log' --exclude='.git/' src/ dest/
rsync -av -e "ssh -p 2222" src/ user@host:/dest/
rsync -av --backup --backup-dir=/backup/old_$(date +%F) src/ dest/
rsync -av --bwlimit=5000 src/ user@host:/dest/   # limit to ~5 MB/s
```

Trailing slash matters:

```bash
rsync -av src  dest/   # creates dest/src/...
rsync -av src/ dest/   # copies the CONTENTS of src into dest/
```

Cron-based remote backup example:

```
0 2 * * * rsync -az --delete -e "ssh -i /home/backup/.ssh/id_ed25519" /srv/data/ backup@backuphost:/backups/data/
```

### 11.9 Port forwarding (tunnels)

```bash
# Local forward: reach a remote-only service via a local port
ssh -L 8080:localhost:80 user@server           # then open http://localhost:8080
ssh -L 5433:db.internal:5432 user@bastion      # access db.internal through bastion

# Remote forward: expose a local service on the server
ssh -R 9000:localhost:3000 user@server

# Dynamic forward: SOCKS5 proxy
ssh -D 1080 user@server                        # configure browser to use localhost:1080

# Background tunnel without a shell
ssh -fNL 8080:localhost:80 user@server
```

### 11.10 Bastion (jump host) basics

A **bastion host** is a hardened, single entry point to a private network. Internal servers accept SSH only from the bastion.

```
Internet --> [Bastion: public IP] --> [Private servers: 10.0.x.x]
```

```bash
ssh -J alice@bastion.example.com admin@10.0.0.5     # ProxyJump (preferred)
ssh -J bastion1,bastion2 admin@10.0.0.5             # multi-hop
scp -J bastion file.txt admin@10.0.0.5:/tmp/
rsync -av -e "ssh -J bastion" src/ admin@10.0.0.5:/dest/
```

Best practices:

- Prefer `ProxyJump` over agent forwarding (`-A`), which exposes your agent to the bastion.
- Disable password auth; require keys (optionally with MFA or short-lived certificates).
- Keep the bastion minimal, patched, logged, and firewall-restricted to known source IPs.
- Use per-user accounts, not shared logins, so logs show who did what.

### 11.11 Multiplexing for speed

```
Host *
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
```

Subsequent `ssh`/`scp` to the same host reuse one connection and are near-instant.

### 11.12 Troubleshooting SSH

```bash
ssh -vvv user@host                       # client-side detail
sudo journalctl -u ssh -f                # server logs (unit "sshd" on RHEL)
sudo tail -f /var/log/auth.log
sudo ss -tlnp | grep :22                 # is sshd listening?
sudo ufw status                          # firewall
ls -ld ~ ~/.ssh ~/.ssh/authorized_keys   # home must not be group/world-writable
```

Common causes of "Permission denied (publickey)": wrong file permissions, public key not in `authorized_keys`, wrong user, `PasswordAuthentication no` with no key loaded, or SELinux contexts (RHEL: `restorecon -R ~/.ssh`).

---

## 12. Shell Environment

### 12.1 Variables

```bash
name="Alice"              # shell variable (no spaces around =)
echo "$name"
echo "${name}_backup"     # braces to delimit
unset name

export EDITOR=vim         # environment variable: inherited by child processes
export PATH="$PATH:$HOME/bin"
env                       # list environment variables
printenv HOME
set                       # all variables and functions
```

Default values and checks:

```bash
echo "${USER_NAME:-guest}"      # use "guest" if unset/empty
echo "${CONFIG:=/etc/app.conf}" # assign default if unset
echo "${REQUIRED:?must be set}" # abort with error if unset
echo "${file%.txt}"             # strip suffix:  a.txt -> a
echo "${file#*/}"               # strip shortest prefix pattern
echo "${#name}"                 # string length
echo "${name^^}"                # uppercase (bash 4+)
```

Special variables:

| Variable | Meaning                            |
|----------|------------------------------------|
| `$?`     | Exit status of last command (0 = success) |
| `$$`     | PID of current shell               |
| `$!`     | PID of last background job         |
| `$0`     | Script name                        |
| `$1..$9` | Positional arguments               |
| `$#`     | Number of arguments                |
| `$@`     | All arguments (each quoted separately with `"$@"`) |
| `$*`     | All arguments as one string        |

Common environment variables: `HOME`, `USER`, `SHELL`, `PWD`, `OLDPWD`, `PATH`, `EDITOR`, `LANG`, `TERM`, `PS1`, `TZ`.

### 12.2 `PATH`

`PATH` is a colon-separated list of directories searched, in order, for commands.

```bash
echo $PATH
# /usr/local/bin:/usr/bin:/bin:/usr/local/games

export PATH="$HOME/.local/bin:$PATH"     # prepend: your tools take priority
export PATH="$PATH:/opt/tools/bin"       # append: lowest priority

which -a python3                         # see order resolution
hash -r                                  # clear the shell's command location cache
```

Run something in the current directory explicitly with `./script.sh` (the current directory is intentionally *not* in `PATH` for security).

### 12.3 Startup files: profiles and rc files

| File                 | Read when                                        |
|----------------------|--------------------------------------------------|
| `/etc/profile`, `/etc/profile.d/*.sh` | Login shells, system-wide       |
| `~/.profile` or `~/.bash_profile` | Login shells (SSH login, `su -`, console) |
| `~/.bashrc`          | Interactive non-login shells (new terminal tab)  |
| `/etc/bash.bashrc`   | Interactive shells, system-wide (Debian)         |
| `~/.bash_logout`     | On logout                                        |
| `/etc/environment`   | Simple `KEY=value` lines (not a shell script), read by PAM |

Debian's `~/.profile` normally sources `~/.bashrc`, so put most things in `~/.bashrc`. Apply changes without logging out:

```bash
source ~/.bashrc      # or:  . ~/.bashrc
```

Typical `~/.bashrc` additions:

```bash
export EDITOR=vim
export PATH="$HOME/.local/bin:$PATH"
HISTSIZE=10000
HISTCONTROL=ignoreboth
shopt -s histappend checkwinsize

# Prompt: user@host:dir$ with colors
PS1='\[\e[32m\]\u@\h\[\e[0m\]:\[\e[34m\]\w\[\e[0m\]\$ '
```

### 12.4 Aliases and functions

```bash
alias ll='ls -lah'
alias ..='cd ..'
alias gs='git status'
alias update='sudo apt update && sudo apt upgrade -y'
alias grep='grep --color=auto'
alias rm='rm -i'               # safer deletes
alias                          # list aliases
unalias ll
\ls                            # bypass an alias for one run
```

Put permanent aliases in `~/.bashrc` (or `~/.bash_aliases`). Aliases can't take arguments; use functions:

```bash
mkcd() { mkdir -p "$1" && cd "$1"; }
backup() { cp -a "$1" "$1.$(date +%F).bak"; }
extract() {
  case "$1" in
    *.tar.gz|*.tgz) tar -xzf "$1" ;;
    *.tar.bz2)      tar -xjf "$1" ;;
    *.zip)          unzip "$1" ;;
    *) echo "unknown archive: $1" >&2; return 1 ;;
  esac
}
```

### 12.5 Shell expansion (order matters)

Bash processes each command line through these expansions:

1. **Brace expansion**
2. **Tilde expansion**
3. **Parameter/variable expansion**
4. **Command substitution**
5. **Arithmetic expansion**
6. **Word splitting**
7. **Pathname expansion (globbing)**

```bash
echo {a,b,c}.txt            # a.txt b.txt c.txt
echo {1..5}                 # 1 2 3 4 5
echo {01..10..3}            # 01 04 07 10
echo file_{jan,feb}_{2025,2026}.csv
mkdir -p proj/{src,docs}

echo ~                      # /home/alice
echo ~bob                   # /home/bob

echo "$HOME"                # variable expansion

echo "Today is $(date +%F)" # command substitution (preferred over backticks)
files=$(ls | wc -l)

echo $((3 + 4 * 2))         # arithmetic: 11
x=5; echo $((x ** 2))       # 25
((x++)); echo $x            # 6

ls *.txt                    # globbing
echo "*.txt"                # in quotes: literal *.txt, no globbing

diff <(sort a.txt) <(sort b.txt)   # process substitution
```

Extended globbing:

```bash
shopt -s extglob globstar
ls !(*.log)                 # everything except .log
ls **/*.py                  # recursive match (globstar)
```

### 12.6 Quoting

| Form       | Behavior                                                           |
|------------|--------------------------------------------------------------------|
| `'single'` | Everything is literal; no expansion at all                         |
| `"double"` | Prevents word splitting & globbing; allows `$var`, `$(cmd)`, `\`   |
| `\char`    | Escapes a single character                                          |
| `$'...'`   | ANSI-C quoting: `$'line1\nline2\tTab'`                              |

```bash
name="Alice Smith"
echo '$name'             # $name
echo "$name"             # Alice Smith
echo "Today: $(date +%F)"
echo 'It'\''s fine'      # It's fine (close, escaped quote, reopen)
echo "Price: \$5"        # Price: $5

# The classic bug: unquoted variables with spaces
file="my report.txt"
touch $file              # creates TWO files: "my" and "report.txt"
touch "$file"            # creates one file: "my report.txt"
rm $file                 # tries to delete "my" and "report.txt"!
```

**Rule of thumb:** always quote variable expansions: `"$var"`, `"$@"`, `"$(cmd)"`.

### 12.7 Exit codes and control operators

```bash
ls /etc && echo "ok"            # run second only if first succeeds
ls /nonexistent || echo "failed" # run second only if first fails
cmd1 ; cmd2                     # run sequentially regardless
cmd &                           # background
echo $?                         # exit status of the last command

mkdir build && cd build && cmake .. || echo "build setup failed"

if [ -f /etc/passwd ]; then echo "exists"; fi
[[ -d /tmp && -w /tmp ]] && echo "writable dir"
```

### 12.8 A small script putting it together

`~/bin/sysreport.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail           # exit on error, unset vars, and pipe failures

OUT="${1:-/tmp/sysreport_$(date +%F).txt}"

{
  echo "== Host =="
  hostnamectl | sed -n '1,6p'
  echo
  echo "== Uptime / Load =="
  uptime
  echo
  echo "== Disk (>80% used) =="
  df -hP | awk 'NR>1 && $5+0 > 80 {print $6, $5}'
  echo
  echo "== Top 5 memory processes =="
  ps -eo pid,comm,%mem --sort=-%mem | head -6
  echo
  echo "== Failed services =="
  systemctl --failed --no-legend || true
} > "$OUT"

echo "Report written to $OUT"
```

```bash
chmod +x ~/bin/sysreport.sh
sysreport.sh                 # works because ~/bin is on PATH
```

---

## 13. Cheat Sheet and Practice Labs

### 13.1 One-liners worth memorizing

```bash
# Find the 10 largest files under /var
sudo find /var -type f -printf '%s %p\n' 2>/dev/null | sort -nr | head -10

# What's listening on which port?
sudo ss -tulpn

# Which process is using the most memory?
ps aux --sort=-%mem | head -5

# Watch a command repeatedly
watch -n 2 'df -h | grep -v tmpfs'

# Replace text across many files (preview with grep first)
grep -rl "oldtext" . | xargs sed -i 's/oldtext/newtext/g'

# Count lines of code by extension
find . -name '*.py' -print0 | xargs -0 cat | wc -l

# Show last 20 failed SSH logins
sudo journalctl -u ssh | grep "Failed password" | tail -20

# Generate a random password
openssl rand -base64 18

# Quickly serve the current directory over HTTP
python3 -m http.server 8000

# Backup with timestamp
tar -czf "backup_$(date +%F_%H%M).tar.gz" /etc
```

### 13.2 Practice labs

**Lab 1: Navigation and files**
1. Create `~/lab/{logs,data,scripts}` with a single `mkdir`.
2. Create 5 files `data/file1.txt` ... `file5.txt` using brace expansion.
3. Use `find` to list all `.txt` files modified in the last 10 minutes.
4. Make a symlink `~/lab/latest` pointing to `data/file5.txt` and inspect it with `ls -l` and `readlink -f`.

**Lab 2: Text processing**
1. Save the sample `data.csv` from section 4.
2. Print the average salary with `awk`.
3. Show how many employees are in each department with `cut | sort | uniq -c`.
4. Replace `ops` with `operations` using `sed` in a copy of the file.

**Lab 3: Permissions**
1. Create users `alice` and `bob`, plus a group `team`.
2. Create `/srv/team` owned by `root:team` with mode `2770`.
3. Confirm that files created by alice are group-owned by `team`.
4. Use `setfacl` to give a third user read-only access.

**Lab 4: Processes and services**
1. Write the `myapp.service` from section 7.8 for any simple script.
2. Enable it, kill it with `kill -9`, and confirm `Restart=on-failure` brings it back.
3. Read its output with `journalctl -u myapp -f`.

**Lab 5: Storage**
1. Create a 1 GB loop device: `dd if=/dev/zero of=disk.img bs=1M count=1024`, then `sudo losetup -f --show disk.img`.
2. Format it with ext4, mount it, add an `fstab` entry with `nofail`, and test with `sudo mount -a`.

**Lab 6: Scheduling and logs**
1. Add a cron job that appends `date` to `~/cron_test.log` every minute; verify, then remove it.
2. Write a `logrotate` config for a custom log and test with `logrotate -d`.

**Lab 7: SSH**
1. Generate an ed25519 key and install it on a VM with `ssh-copy-id`.
2. Create a `~/.ssh/config` entry so `ssh myvm` works.
3. Disable password authentication on the VM (keeping a second session open!) and test.
4. Use `rsync -av --dry-run` to preview a sync, then run it for real.

### 13.3 Safety checklist for administrators

- [ ] Take a backup/snapshot before major changes.
- [ ] Keep a second SSH session open when editing `sshd_config`, firewall rules, or `sudoers`.
- [ ] Use `visudo`, `sshd -t`, `mount -a`, and `logrotate -d` to validate configs before applying.
- [ ] Preview destructive commands (`ls` before `rm`, `--dry-run` before `rsync --delete`).
- [ ] Quote your variables.
- [ ] Prefer least privilege: use `sudo` for single commands rather than staying root.
- [ ] Keep systems patched (`unattended-upgrades` for security updates).
- [ ] Monitor disk space, inode usage, failed services, and authentication logs.

### 13.4 Further reading

- `man hier`, `man systemd.service`, `man sshd_config`, `man crontab`, `man 7 signal`
- The Linux Documentation Project: <https://tldp.org>
- Ubuntu Server documentation: <https://documentation.ubuntu.com/server/>
- Debian Administrator's Handbook: <https://debian-handbook.info>
- Arch Wiki (excellent general reference): <https://wiki.archlinux.org>