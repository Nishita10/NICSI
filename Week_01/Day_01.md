# Day 1 — Linux Fundamentals

A complete reference covering Linux installation, the file system, permissions, users & groups, package management, and the two core tools you'll live in every day: **bash** and **tmux**.

---

## Table of Contents
1. [Introduction to Linux](#1-introduction-to-linux)
2. [Linux Installation (Virtual Machine)](#2-linux-installation-virtual-machine)
3. [The Linux File System](#3-the-linux-file-system)
4. [Permissions](#4-permissions)
5. [Users & Groups](#5-users--groups)
6. [Package Management](#6-package-management)
7. [Tool: Bash](#7-tool-bash)
8. [Tool: tmux](#8-tool-tmux)
9. [Quick Command Cheat Sheet](#9-quick-command-cheat-sheet)

---

## 1. Introduction to Linux

Linux is a free, open-source, Unix-like operating system **kernel**. What people call "Linux" is really a **distribution (distro)** — the kernel plus a collection of software (shell, package manager, libraries, desktop environment) bundled together.

**Popular distros and their lineage:**

| Family | Distros | Package Manager |
|---|---|---|
| Debian-based | Ubuntu, Kali Linux, Linux Mint | `apt` / `dpkg` |
| Red Hat-based | RHEL, CentOS, Fedora, Rocky Linux | `dnf` / `yum` / `rpm` |
| Arch-based | Arch Linux, Manjaro | `pacman` |
| SUSE-based | openSUSE | `zypper` |

**Why Linux matters in the field (DevOps, security, SysAdmin, cloud):**
- Powers the vast majority of servers, cloud instances (AWS/Azure/GCP), containers (Docker/Kubernetes), and embedded systems.
- Nearly all penetration testing / cybersecurity tooling (Kali, Parrot OS) runs on Linux.
- Open source, scriptable, and highly configurable — ideal for automation.

**Core philosophy:** "Everything is a file," small tools that do one thing well, and composability via pipes (`|`).

---

## 2. Linux Installation (Virtual Machine)

Running Linux inside a VM lets you experiment safely without touching your host OS.

### 2.1 Choosing a Hypervisor

| Hypervisor | Platform | Notes |
|---|---|---|
| **VirtualBox** | Windows/macOS/Linux | Free, beginner-friendly |
| **VMware Workstation/Player** | Windows/Linux | Free (Player), more performance |
| **Hyper-V** | Windows Pro/Enterprise | Built into Windows |
| **UTM** | macOS (Apple Silicon) | Uses QEMU under the hood |

### 2.2 Installation Steps (Generic — e.g., Ubuntu on VirtualBox)

1. **Download the hypervisor** (e.g., VirtualBox from virtualbox.org) and install it on your host OS.
2. **Download the distro ISO** (e.g., Ubuntu Desktop/Server `.iso` from ubuntu.com).
3. **Create a new VM:**
   - Name it, select OS type "Linux," version matching your distro.
   - Allocate RAM (≥2 GB, 4 GB+ recommended).
   - Create a Virtual Hard Disk (VDI, dynamically allocated, ≥20–25 GB).
4. **Attach the ISO** to the VM's virtual optical drive (Settings → Storage → Controller: IDE → choose ISO file).
5. **Configure networking** — default "NAT" works for internet access; use "Bridged Adapter" if the VM needs its own LAN IP.
6. **Boot the VM** — it boots from the ISO into the installer.
7. **Follow the installer:**
   - Choose language/keyboard layout.
   - Choose "Erase disk and install Linux" (safe — this only affects the *virtual* disk).
   - Create your username, hostname, and password.
   - Wait for installation to complete, then reboot and **remove/eject the ISO**.
8. **Install Guest Additions / VMware Tools** (optional but recommended) — enables shared clipboard, better screen resolution, shared folders.
   ```bash
   sudo apt update && sudo apt install build-essential dkms linux-headers-$(uname -r)
   # Then: Devices menu → Insert Guest Additions CD image → run autorun.sh
   ```

### 2.3 Post-Installation Checklist
```bash
sudo apt update && sudo apt upgrade -y   # update package lists & upgrade installed packages
hostnamectl                              # verify hostname & OS info
ip a                                     # check network/IP configuration
whoami                                   # confirm current user
```

### 2.4 Snapshots
Take a VM **snapshot** right after a clean install so you can always roll back if you break something while practicing (VirtualBox: Machine → Take Snapshot).

---

## 3. The Linux File System

### 3.1 The Filesystem Hierarchy Standard (FHS)

Linux has a single unified directory tree starting at `/` (root) — unlike Windows' drive letters (C:\, D:\).

| Directory | Purpose |
|---|---|
| `/` | Root of the entire filesystem |
| `/bin` | Essential user command binaries (`ls`, `cp`, `mv`) |
| `/sbin` | Essential system binaries (admin tools, e.g. `fdisk`) |
| `/etc` | System-wide configuration files |
| `/home` | Personal directories for each user (`/home/john`) |
| `/root` | Home directory of the root (superuser) |
| `/var` | Variable data — logs, mail, spools (`/var/log`) |
| `/tmp` | Temporary files, cleared on reboot |
| `/usr` | User-installed programs & libraries (`/usr/bin`, `/usr/lib`) |
| `/lib`, `/lib64` | Shared libraries needed by binaries in `/bin`, `/sbin` |
| `/opt` | Optional/third-party software packages |
| `/dev` | Device files (`/dev/sda`, `/dev/null`) |
| `/proc` | Virtual filesystem exposing kernel & process info |
| `/mnt`, `/media` | Mount points for external/temporary filesystems |
| `/boot` | Bootloader files, kernel images |
| `/srv` | Data for services (web/ftp servers) |

### 3.2 File Types
Linux treats almost everything as a file. Check type with `ls -l` (first character):

| Symbol | Type |
|---|---|
| `-` | Regular file |
| `d` | Directory |
| `l` | Symbolic link |
| `b` | Block device (e.g., disk) |
| `c` | Character device (e.g., keyboard) |
| `s` | Socket |
| `p` | Named pipe |

### 3.3 Navigation & File Management Commands

```bash
pwd                     # print working (current) directory
ls                      # list directory contents
ls -la                  # long format, including hidden files (dotfiles)
ls -lh                  # human-readable file sizes
cd /path/to/dir         # change directory
cd ..                   # go up one level
cd ~ or cd              # go to home directory
cd -                    # go to previous directory

mkdir myfolder          # create a directory
mkdir -p a/b/c          # create nested directories at once

touch file.txt          # create empty file / update timestamp

cp file.txt backup.txt          # copy file
cp -r dir1/ dir2/               # copy directory recursively

mv old.txt new.txt              # rename/move file
mv file.txt /home/user/docs/    # move file to another directory

rm file.txt              # delete a file
rm -r dir/                # delete directory recursively
rm -rf dir/                # force delete, no prompts (use carefully!)

find /home -name "*.log"        # search for files by name
locate filename.txt              # fast search using indexed DB (updatedb first)

cat file.txt              # display file contents
less file.txt              # view file page-by-page (scrollable)
head -n 10 file.txt         # first 10 lines
tail -n 10 file.txt          # last 10 lines
tail -f /var/log/syslog       # follow file in real time (great for logs)

file myfile               # identify file type
stat file.txt               # detailed metadata (size, timestamps, inode)

ln -s /path/target linkname   # create a symbolic (soft) link
ln /path/target linkname       # create a hard link

du -sh folder/              # size of a folder (human-readable)
df -h                         # disk space usage of mounted filesystems
tree                           # visual directory tree (may need install)
```

### 3.4 Wildcards & Redirection
```bash
ls *.txt              # all .txt files
ls file?.txt           # file1.txt, file2.txt, etc.

command > file.txt      # redirect output, overwrite file
command >> file.txt      # redirect output, append to file
command < file.txt        # use file as input
command1 | command2        # pipe: output of cmd1 becomes input of cmd2
```

---

## 4. Permissions

Every file/directory has an **owner (user)**, a **group**, and **permission bits** for three categories: **Owner (u)**, **Group (g)**, **Others (o)**.

### 4.1 Reading `ls -l` Output

```
-rwxr-xr--  1 john devs  4096 Jul 2 10:00 script.sh
```

| Segment | Meaning |
|---|---|
| `-` | File type (`-`=file, `d`=directory) |
| `rwx` | Owner permissions: read, write, execute |
| `r-x` | Group permissions: read, execute (no write) |
| `r--` | Others permissions: read only |
| `john` | Owner |
| `devs` | Group |

### 4.2 Permission Values

| Symbol | Meaning | Numeric Value |
|---|---|---|
| `r` | Read | 4 |
| `w` | Write | 2 |
| `x` | Execute | 1 |
| `-` | No permission | 0 |

Add values per category → e.g. `rwx` = 4+2+1 = **7**, `r-x` = 4+0+1 = **5**, `r--` = 4 = **4**.
So `rwxr-xr--` = **754**.

### 4.3 Changing Permissions — `chmod`

```bash
chmod 755 script.sh          # numeric mode: rwx r-x r-x
chmod u+x script.sh           # symbolic: add execute for owner
chmod g-w file.txt             # remove write from group
chmod o=r file.txt              # set others to read-only
chmod a+x script.sh              # add execute for all (owner, group, others)
chmod -R 644 folder/               # apply recursively to all files inside
```

**Symbolic operators:** `u`=user/owner, `g`=group, `o`=others, `a`=all | `+`=add, `-`=remove, `=`=set exactly.

### 4.4 Changing Ownership — `chown` / `chgrp`

```bash
chown john file.txt              # change owner to john
chown john:devs file.txt          # change owner AND group
chown -R john:devs folder/          # recursive ownership change
chgrp devs file.txt                  # change only the group
```

### 4.5 Special Permissions

| Permission | Symbol | Numeric | Effect |
|---|---|---|---|
| SUID (Set User ID) | `s` (owner x position) | 4000 | Execute file as the file's owner (e.g., `passwd`) |
| SGID (Set Group ID) | `s` (group x position) | 2000 | Execute as file's group; on dirs, new files inherit group |
| Sticky Bit | `t` (others x position) | 1000 | On a shared directory, only the file owner can delete/rename their own files (e.g. `/tmp`) |

```bash
chmod 4755 file        # set SUID
chmod 2755 dir          # set SGID
chmod 1777 dir           # set sticky bit (like /tmp)
chmod u+s file             # SUID symbolically
```

### 4.6 Default Permissions — `umask`
`umask` defines which permission bits are **removed** from the default (666 for files, 777 for dirs) when new files/dirs are created.
```bash
umask               # show current umask (commonly 0022)
umask 027             # set a new umask for the session
```
Default file: 666 − 022 = 644. Default dir: 777 − 022 = 755.

---

## 5. Users & Groups

### 5.1 Key Files

| File | Contents |
|---|---|
| `/etc/passwd` | User account info: username, UID, GID, home dir, shell |
| `/etc/shadow` | Encrypted passwords & password aging (root-readable only) |
| `/etc/group` | Group definitions and members |
| `/etc/sudoers` | Who can run commands as root via `sudo` (edit with `visudo`) |

Example `/etc/passwd` line:
```
john:x:1001:1001:John Doe:/home/john:/bin/bash
```
`username : password-placeholder : UID : GID : comment : home-dir : login-shell`

### 5.2 User Management Commands

```bash
whoami                      # current username
id                            # current UID, GID, and group memberships
id john                        # info for a specific user

sudo useradd john                # create a new user (basic)
sudo useradd -m -s /bin/bash john  # create user with home dir & bash shell
sudo passwd john                    # set/change john's password

sudo usermod -aG sudo john           # add john to the 'sudo' group (admin rights)
sudo usermod -l newname oldname       # rename a user
sudo usermod -d /new/home -m john       # change & move home directory

sudo userdel john                  # delete a user (keep home dir)
sudo userdel -r john                 # delete user AND home directory

su john                    # switch to user john (needs john's password)
su -                         # switch to root, loading root's environment
sudo command                  # run a single command as root
sudo -i                         # start a root interactive shell
```

### 5.3 Group Management Commands

```bash
groups                    # groups the current user belongs to
groups john                  # groups john belongs to

sudo groupadd devs             # create a new group
sudo groupdel devs               # delete a group

sudo usermod -aG devs john         # add john to group 'devs' (append, don't overwrite!)
sudo gpasswd -d john devs            # remove john from group 'devs'

cat /etc/group                # view all groups and their members
```

⚠️ **Common pitfall:** always use `-aG` (append) with `usermod`, not just `-G`, or you will overwrite the user's existing group memberships.

### 5.4 sudo vs root
- **root** is the all-powerful superuser (UID 0).
- **sudo** ("superuser do") lets permitted users run specific/all commands as root, tracked in logs — the recommended, safer practice over logging in as root directly.
- Edit sudo permissions safely with:
```bash
sudo visudo
```

---

## 6. Package Management

Package managers install, update, and remove software along with dependency resolution.

### 6.1 Debian/Ubuntu — `apt` / `dpkg`

```bash
sudo apt update                  # refresh package index (does NOT install anything)
sudo apt upgrade                  # upgrade all installed packages
sudo apt full-upgrade               # upgrade, also handling dependency changes

sudo apt install nginx                # install a package
sudo apt install pkg1 pkg2               # install multiple packages
sudo apt remove nginx                      # remove package, keep config files
sudo apt purge nginx                         # remove package AND config files
sudo apt autoremove                            # remove unused dependencies

apt search nginx                    # search for a package
apt show nginx                        # show package details
apt list --installed                    # list installed packages

sudo dpkg -i package.deb              # install a local .deb file
dpkg -l                                  # list all installed packages (low-level)
dpkg -L nginx                              # list files installed by a package
```

### 6.2 Red Hat/Fedora/CentOS/Rocky — `dnf` / `yum` / `rpm`

```bash
sudo dnf update                    # update all packages (dnf = modern successor to yum)
sudo dnf install httpd                # install a package
sudo dnf remove httpd                   # remove a package
dnf search httpd                          # search for a package
dnf info httpd                              # show package details
dnf list installed                            # list installed packages

sudo rpm -ivh package.rpm             # install a local .rpm file (i=install, v=verbose, h=hash progress)
rpm -qa                                  # query all installed packages
rpm -qi package                            # query package info
```

### 6.3 Arch — `pacman`

```bash
sudo pacman -Syu             # sync repos & upgrade everything
sudo pacman -S package         # install a package
sudo pacman -R package           # remove a package
pacman -Ss keyword                 # search for a package
pacman -Q                            # list installed packages
```

### 6.4 Universal/Cross-distro Formats
```bash
sudo snap install code           # Snap packages (sandboxed, Canonical)
flatpak install app                 # Flatpak packages (sandboxed, cross-distro)
```

### 6.5 Compiling From Source (when no package exists)
```bash
./configure
make
sudo make install
```

---

## 7. Tool: Bash

Bash (**B**ourne **A**gain **SH**ell) is the default command-line shell/interpreter on most Linux distros.

### 7.1 Shell Basics
```bash
echo $SHELL              # show current shell
echo $0                    # name of current shell/script
bash                          # start a new bash session
exit                            # exit the shell

history                    # list command history
!45                          # re-run command number 45 from history
!!                             # re-run the last command
Ctrl + R                         # reverse search through history
```

### 7.2 Environment Variables
```bash
echo $HOME               # print value of HOME variable
env                         # list all environment variables
export MYVAR="hello"          # create/export a variable for child processes
unset MYVAR                     # remove a variable

echo $PATH                       # directories bash searches for executables
export PATH=$PATH:/new/dir          # add a directory to PATH
```

### 7.3 Bash Scripting Essentials

```bash
#!/bin/bash
# ^ shebang: tells the OS which interpreter to use

# Variables (no spaces around =)
name="John"
echo "Hello, $name"

# User input
read -p "Enter your name: " user_name

# Conditionals
if [ "$name" == "John" ]; then
    echo "Match!"
elif [ "$name" == "Jane" ]; then
    echo "Different match"
else
    echo "No match"
fi

# Loops
for i in 1 2 3; do
    echo "Number: $i"
done

count=0
while [ $count -lt 5 ]; do
    echo "Count: $count"
    count=$((count+1))
done

# Functions
greet() {
    echo "Hello, $1"   # $1 = first argument passed to function
}
greet "World"

# Script arguments
echo "Script name: $0"
echo "First arg: $1"
echo "All args: $@"
echo "Arg count: $#"
```

Run a script:
```bash
chmod +x script.sh     # make it executable
./script.sh              # run it
bash script.sh              # or run without execute permission
```

### 7.4 Useful Bash Command Combos
```bash
grep "error" logfile.txt          # search for a pattern in a file
grep -r "TODO" ./src                 # recursive search in a directory
grep -i "hello" file.txt               # case-insensitive search

sed 's/foo/bar/g' file.txt              # find & replace text
awk '{print $1}' file.txt                 # print the first column of each line

chmod +x, ps aux, kill -9 <pid>             # process management
ps aux                                # list all running processes
top / htop                              # live resource monitor
kill -9 1234                              # force-kill process with PID 1234

alias ll='ls -la'                    # create a shortcut command
which python3                          # locate an executable's path
man ls                                   # manual page / help for a command
```

### 7.5 Command Chaining
```bash
cmd1 && cmd2      # run cmd2 only if cmd1 succeeds
cmd1 || cmd2       # run cmd2 only if cmd1 fails
cmd1 ; cmd2          # run cmd2 regardless of cmd1's result
cmd1 | cmd2            # pipe output of cmd1 into cmd2
```

---

## 8. Tool: tmux

**tmux** (terminal multiplexer) lets you run multiple terminal sessions inside one window, detach from them, and reattach later — essential for remote/server work where SSH connections can drop.

### 8.1 Core Concepts
- **Session** — a collection of windows (like a workspace); persists even if you disconnect.
- **Window** — like a tab within a session.
- **Pane** — a split section within a window.

### 8.2 Session Management
```bash
tmux                          # start a new unnamed session
tmux new -s mysession           # start a new named session

tmux ls                          # list all sessions
tmux attach -t mysession           # reattach to a named session
tmux attach -t mysession -d          # detach it from any other client first, then attach

tmux kill-session -t mysession        # kill a specific session
tmux kill-server                        # kill ALL tmux sessions
```

**Detach from inside a session:** `Ctrl+b` then `d`

### 8.3 Inside tmux — Key Bindings
All tmux commands start with the **prefix**: `Ctrl+b`, then release, then press the key.

| Action | Keys (after `Ctrl+b`) |
|---|---|
| Detach session | `d` |
| New window | `c` |
| Next window | `n` |
| Previous window | `p` |
| Switch to window # | `0`–`9` |
| Rename current window | `,` |
| List windows | `w` |
| Split pane vertically | `%` |
| Split pane horizontally | `"` |
| Move between panes | Arrow keys |
| Close current pane | `x` |
| Toggle pane zoom (fullscreen) | `z` |
| Show pane numbers | `q` |
| Scroll/copy mode | `[` (then arrows/PgUp; `q` to exit) |

### 8.4 Practical Workflow Example
```bash
tmux new -s deploy          # start a session named "deploy"
# Ctrl+b %                    → split pane vertically
# Ctrl+b "                     → split pane horizontally
# run a long process, e.g.:
ping google.com
# Ctrl+b d                     → detach, process keeps running in background
tmux attach -t deploy          # come back later and reattach
```

### 8.5 Config File
Persistent settings go in `~/.tmux.conf`, e.g.:
```bash
set -g mouse on                 # enable mouse support (click to switch panes/resize)
set -g history-limit 5000          # scrollback buffer size
```
Reload config without restarting tmux:
```
Ctrl+b :   then type   source-file ~/.tmux.conf
```

---

## 9. Quick Command Cheat Sheet

| Category | Command | Purpose |
|---|---|---|
| Navigation | `pwd`, `cd`, `ls -la` | Where am I / move / list |
| Files | `cp`, `mv`, `rm`, `touch`, `mkdir` | Basic file ops |
| Viewing | `cat`, `less`, `head`, `tail -f` | Read file contents |
| Search | `find`, `grep -r`, `locate` | Find files / text |
| Permissions | `chmod 755`, `chown user:group` | Set access rights |
| Users | `useradd -m`, `passwd`, `usermod -aG` | Manage accounts |
| Groups | `groupadd`, `groups`, `gpasswd -d` | Manage groups |
| Packages (Debian) | `apt update && apt install` | Install software |
| Packages (RHEL) | `dnf install` | Install software |
| Processes | `ps aux`, `top`, `kill -9` | Manage running programs |
| Disk | `df -h`, `du -sh` | Check disk/space usage |
| Networking | `ip a`, `ping`, `curl` | Check connectivity |
| tmux | `tmux new -s`, `Ctrl+b d`, `tmux attach -t` | Persistent terminal sessions |

---

### Practice Suggestions for Day 1
1. Install a Linux VM (Ubuntu Server or Desktop) and take a fresh snapshot.
2. Navigate the filesystem using only the terminal — no GUI file manager.
3. Create a test user, add them to a custom group, and verify with `id`.
4. Practice `chmod`/`chown` on a set of test files until the numeric permissions feel intuitive.
5. Install and remove a package with `apt` (or your distro's manager), then check logs in `/var/log`.
6. Write a 10-line bash script that loops through a directory and prints file names.
7. Open a `tmux` session, split it into 3 panes, run a command in each, detach, then reattach.
