# Placement Study Notes — Part 8: Linux in Depth

*Extends the short Linux section in Part 2 (D4). Written for developer and backend interviews, online practical tests and day-to-day work on servers. Commands are for a typical Debian/Ubuntu or RHEL-family system with GNU tools; macOS differs in places (flagged).*

## 0. How to use these notes

Each topic has: **idea**, **commands with examples**, **interview answer**, **loopholes** (the mistakes and trick questions). The final sections give a **troubleshooting playbook**, **40 rapid-fire questions** and a **cheat sheet**. Practise every command on a real shell (a VM, WSL, a Docker container or a cloud free-tier instance); reading alone does not stick.

**Contents:** L1 Architecture, boot and the shell · L2 Filesystem, links, devices, mounts · L3 Users, groups and permissions · L4 Processes, signals, job control · L5 Pipes, redirection and text processing · L6 Shell scripting · L7 Packages, systemd, cron, logs · L8 Networking and SSH · L9 Disk, memory, performance and troubleshooting playbook · L10 Security and hardening basics · L11 Linux behind containers · L12 Interview Q&A, scenarios and cheat sheet.

---

## L1. Architecture, boot process and the shell

**Layers:** hardware → **kernel** (process scheduling, memory management, drivers, filesystems, networking, system calls; Part 2 A1, A8b) → **userland** (libc, shell, utilities, services) → applications. A **distribution** (Ubuntu, Debian, Fedora, RHEL, Alpine, Arch) = kernel + package manager + default tools. The **shell** (bash, zsh, sh, dash) is a user program that reads commands and starts processes with `fork` + `exec`.

**Boot sequence:** firmware (BIOS/UEFI) runs self-tests and loads the **bootloader** (GRUB) → bootloader loads the **kernel** and an **initramfs** (a temporary root with drivers) → the kernel mounts the real root filesystem and starts **PID 1** (`systemd` on most distributions) → systemd reaches a **target** (`multi-user.target` for servers, `graphical.target` for desktops) by starting services in dependency order → login prompt/SSH.

**Everything is a file:** regular files, directories, devices (`/dev/sda`), pipes, sockets and kernel information (`/proc`, `/sys`) are accessed through file-like interfaces, which is why `cat`, redirection and pipes work on almost everything.

**Getting help:** `man ls` (manual; `/pattern` to search, `q` to quit), `ls --help`, `info`, `apropos copy` (search man page summaries), `type cmd`/`which cmd` (what runs), `tldr` (community examples).

**Shell environment**

- **Variables:** `NAME=value` (no spaces around `=`); `export NAME` makes it visible to child processes; `echo "$NAME"`; `env` / `printenv` list the environment; `unset NAME`.
- **`PATH`:** a colon-separated list of directories searched for executables (`echo $PATH`); `./script.sh` runs a file in the current directory because `.` is usually not in `PATH`.
- **Startup files:** login shells read `~/.profile` / `~/.bash_profile`; interactive non-login bash reads `~/.bashrc`; put aliases and prompt settings in `~/.bashrc`, exported environment in `~/.profile`.
- **Aliases and history:** `alias ll='ls -alF'`; `history`, `!!` (previous command), `!$` (last argument), `Ctrl+R` (reverse search), `Ctrl+C` (interrupt), `Ctrl+D` (end of input), `Ctrl+Z` (suspend), `Ctrl+L` (clear), tab completion.
- **Wildcards (globbing, expanded by the shell before the command runs):** `*` (any string), `?` (one character), `[abc]`, `{a,b}.txt` (brace expansion).
- **Quoting:** `'single'` = literal; `"double"` = variables and `$(...)` expand; backslash escapes one character. Quote variables: `"$file"`.
- **Exit status:** every command returns a number in `$?`; **0 = success**, non-zero = failure. `cmd1 && cmd2` runs cmd2 only if cmd1 succeeded; `cmd1 || cmd2` only if it failed; `;` always continues.

**Interview answer:** "The kernel manages hardware and gives programs system calls; the shell is just a program that launches other programs. At boot the firmware loads GRUB, which loads the kernel and initramfs; the kernel starts systemd as PID 1, which starts services until the target is reached."

**Loopholes:** the shell is not the kernel; `export` affects child processes only (not the parent shell); wildcards are expanded by the **shell**, not by `ls`/`rm`, so an unquoted `*` in a directory with a file named `-rf` is a classic trap; `sh` may be `dash` (no bashisms like `[[ ]]`); spaces in `VAR = value` make bash run a command named `VAR`.

---

## L2. Filesystem hierarchy, links, devices and mounts

**Filesystem Hierarchy Standard (FHS)**

| Path | Purpose |
| --- | --- |
| `/` | root of the single directory tree (there are no drive letters) |
| `/home/user` | users' files; `/root` is root's home |
| `/etc` | system-wide configuration (text files: `/etc/passwd`, `/etc/hosts`, `/etc/fstab`, `/etc/ssh/sshd_config`) |
| `/var` | variable data: logs (`/var/log`), caches, spools, databases, mail |
| `/tmp` | temporary files (cleared on reboot; sticky bit); `/var/tmp` survives reboots |
| `/usr` | installed programs and libraries (`/usr/bin`, `/usr/lib`, `/usr/local` for manual installs) |
| `/bin`, `/sbin`, `/lib` | essential binaries and libraries (often symlinks into `/usr` on modern systems) |
| `/opt` | optional/third-party software |
| `/dev` | device files (`/dev/sda`, `/dev/null`, `/dev/zero`, `/dev/urandom`, `/dev/tty`) |
| `/proc` | virtual files exposing processes and kernel state (`/proc/cpuinfo`, `/proc/meminfo`, `/proc/PID/`) |
| `/sys` | virtual interface to kernel objects and devices |
| `/mnt`, `/media` | mount points for temporary and removable filesystems |
| `/boot` | kernel, initramfs, bootloader files |
| `/run` | runtime data since boot (PID files, sockets) |

**Navigation and inspection**

```bash
pwd                       # print working directory          cd /var/log   cd ..   cd -   cd ~
ls -lah                   # long listing, hidden files, human sizes   ls -ld dir  (the directory itself)
tree -L 2                 # directory tree two levels deep
stat file                 # size, inode, permissions, timestamps (atime/mtime/ctime)
file photo.jpg            # detect file type from content
find /var/log -name '*.log' -size +50M -mtime -7      # by name, size, modified within 7 days
find . -type f -name '*.tmp' -delete                   # careful: test without -delete first
find . -type f -exec grep -l 'TODO' {} +                # run a command on matches
locate name               # fast, uses a prebuilt database (updatedb)
du -sh * | sort -h        # which entries use the most space
df -h                     # free/used space per filesystem      df -i  (inode usage)
lsblk                     # block devices and mount points      mount | column -t
```

**Inodes and links**

- An **inode** stores a file's metadata (owner, permissions, size, timestamps, pointers to data blocks) but **not its name**; a directory is a table mapping names to inode numbers. A filesystem can run out of **inodes** (millions of tiny files) while `df -h` still shows free space: check `df -i`.
- **Hard link** (`ln file link`): another directory entry for the **same inode**; deleting one name does not remove the data until the last link (link count) is gone; cannot cross filesystems or link directories.
- **Symbolic (soft) link** (`ln -s target link`): a small file containing a **path**; can cross filesystems and point to directories; breaks (dangles) if the target is removed or moved.
- **Deleting a file** (`rm`) removes a name; space is freed only when no names and no open file descriptors remain: a process holding a deleted log file open keeps the space used (`lsof +L1` finds it; restart or truncate the process).

**Mounting and filesystems**

- Storage devices are attached to a directory with `mount /dev/sdb1 /mnt/data` and detached with `umount /mnt/data` (fails with "target is busy" if a process uses it: `lsof +f -- /mnt/data`). `/etc/fstab` lists persistent mounts (`device  mountpoint  type  options  dump  pass`); use **UUIDs** (`blkid`) instead of `/dev/sdX` names, and test with `mount -a` before rebooting (a bad fstab line can prevent boot).
- Common types: **ext4** (default), **xfs** (large files, RHEL default), **btrfs/zfs** (snapshots), **tmpfs** (RAM), **nfs** (network), **overlayfs** (containers), **vfat/ntfs** (compatibility).
- Special devices: `/dev/null` (discards writes: `cmd > /dev/null 2>&1`), `/dev/zero` (endless zero bytes), `/dev/urandom` (random bytes), `/dev/full`.
- Extra tools: **LVM** (flexible volumes), **RAID** (redundancy), `fdisk`/`parted` (partitioning), `mkfs.ext4` (format), `fsck` (repair an **unmounted** filesystem).

**Interview answer:** "A directory maps names to inodes; an inode holds metadata and data pointers. A hard link is another name for the same inode, a symlink is a file containing a path. Space is freed only when the link count and open file handles both reach zero."

**Loopholes:** `rm` does not shred data; hard links cannot span filesystems; a symlink's permissions are irrelevant (the target's matter); relative symlinks resolve relative to the link's directory; `cd` into a symlinked directory can show a logical path different from `pwd -P`; **never** run `fsck` on a mounted filesystem; `df` and `du` can disagree (deleted-but-open files, mount points hiding data, reserved blocks).

---

## L3. Users, groups and permissions

**Identity files:** `/etc/passwd` (`name:x:UID:GID:comment:home:shell`, world-readable), `/etc/shadow` (password hashes, root-only), `/etc/group` (group membership). **UID 0 is root** (all-powerful); system accounts have low UIDs and `/usr/sbin/nologin`; regular users start at 1000 on most distributions.

```bash
id                         # your uid, gid and groups        whoami   groups   who   w   last
sudo useradd -m -s /bin/bash asha      sudo passwd asha      sudo usermod -aG docker asha     # -a = append (without it groups are REPLACED)
sudo groupadd devs         sudo gpasswd -d asha devs        sudo userdel -r asha
sudo -u postgres psql      sudo -i  (root shell)      su - asha      visudo  (edit sudoers safely)
```

**Reading permissions**

```
-rwxr-x---  1  asha  devs  4096  Oct 28 10:15  deploy.sh
^|-||-||-|
| |  |  |  type: - file, d directory, l symlink, c/b device, s socket, p pipe
|  owner(u)=rwx  group(g)=r-x  other(o)=---
```

| Bit | On a file | On a directory |
| --- | --- | --- |
| **r (4)** | read contents | **list** names (`ls`) |
| **w (2)** | modify contents | **create, delete, rename** entries inside (needs `x` too) |
| **x (1)** | execute as a program | **enter/traverse** (`cd`, access files by path) |

**Octal and symbolic modes:** `rwx = 7`, `rw- = 6`, `r-x = 5`, `r-- = 4`. `chmod 755 f` = `rwxr-xr-x`; `chmod 640 f` = `rw-r-----`; `chmod u+x,g-w,o= f`; `chmod -R 750 dir` (recursive; use `find ... -type d` / `-type f` to give directories and files different modes: `chmod -R u=rwX,g=rX,o= dir` where capital `X` sets execute only on directories and already-executable files).

**Ownership:** `chown asha:devs f`, `chown -R app:app /opt/app`, `chgrp devs f`. Only root may change a file's owner.

**How the kernel decides (first match wins, no fallthrough):** if you are the **owner**, only the owner bits apply; else if you are in the **group**, the group bits apply; else the **other** bits. So a file `---rwxrwx` locks out its owner while allowing everyone else (but the owner can still `chmod` it). Root bypasses read/write checks (execute still needs at least one `x` bit for regular files).

**umask (default permission mask):** new files start from 666 and new directories from 777, then the umask bits are removed. `umask 022` → files **644**, directories **755**; `umask 027` → files **640**, directories **750**; `umask 077` → private (**600/700**). Set per shell or in `~/.profile`.

**Special bits**

| Bit | Octal | Effect | Example |
| --- | --- | --- | --- |
| **setuid** | 4000 | an executable runs with the **file owner's** privileges | `/usr/bin/passwd` (shown as `-rwsr-xr-x`) |
| **setgid** | 2000 | executable runs with the group's privileges; on a **directory**, new files inherit the directory's group | shared project directory |
| **sticky** | 1000 | in a world-writable directory only the file owner (or root) may delete/rename entries | `/tmp` (`drwxrwxrwt`) |

`chmod 4755 prog`, `chmod u+s prog`, `chmod 2775 shared/`, `chmod +t dir`. Audit setuid files: `find / -perm -4000 -type f 2>/dev/null`.

**ACLs:** finer control than owner/group/other: `setfacl -m u:asha:rw file`, `getfacl file` (a `+` after the mode string means an ACL exists). **Capabilities:** split root's power into pieces (`setcap cap_net_bind_service=+ep prog` lets a program bind port 80 without being root).

**Worked examples**

1. A deploy script should be runnable by the owner and group but not others: `chmod 750 deploy.sh`.
2. Make a web directory readable by the web server user `www-data` through group access: `chgrp -R www-data /var/www/site && chmod -R g+rX /var/www/site`.
3. `umask 022`, create `a.txt` and `d/`: modes `644` and `755`.
4. SSH keys must be private or SSH ignores them: `chmod 700 ~/.ssh && chmod 600 ~/.ssh/id_ed25519 ~/.ssh/authorized_keys`.

**Interview answer:** "Permissions are checked against owner, then group, then other, and the first applicable class decides. On directories, `x` means traverse and `w` means create/delete entries. The umask removes bits from the default 666/777, and setuid, setgid and sticky bits handle privilege and shared-directory cases."

**Loopholes:** `chmod 777` is almost never the right fix (find the owner/group problem); `usermod -G` without `-a` **replaces** the groups; a new group membership applies only after re-login (or `newgrp`); deleting a file needs write permission on the **directory**, not on the file; `chmod -R` on a tree gives files the execute bit unless you use `X`; setuid on shell scripts is ignored on Linux and dangerous elsewhere; `sudo` logs commands, `su -` switches the whole environment.

---

## L4. Processes, signals and job control

(Process model, states and `fork`/`exec`: Part 2 A2, A3.)

```bash
ps aux                       # all processes (BSD syntax: user, PID, %CPU, %MEM, STAT, command)       ps -ef  (UNIX syntax, shows PPID)
ps -eo pid,ppid,stat,pcpu,pmem,etime,cmd --sort=-pcpu | head      # custom columns, top CPU users
pstree -p                    # parent-child tree          top / htop   (interactive: P sort CPU, M sort memory, k kill, q quit)
pgrep -fa python             # find PIDs by name/command line      pidof nginx      cat /proc/1234/status   ls -l /proc/1234/fd
lsof -i :8080                # who listens on a port           lsof -p 1234   (files opened by a process)
strace -p 1234 -f -tt        # trace system calls of a running process (Part 2 A8b)      uptime   # load averages
```

**Process states (the `STAT` column):** `R` running/runnable, `S` interruptible sleep (waiting for an event), **`D` uninterruptible sleep** (usually waiting for disk/NFS I/O; cannot be killed until the I/O returns), `T` stopped (Ctrl+Z or SIGSTOP), `Z` **zombie** (finished, parent has not called `wait()`; consumes only a table entry; kill the **parent** or fix it so it reaps children; `init` adopts orphans). Extra letters: `s` session leader, `+` foreground, `<` high priority, `N` low priority.

**Load average:** the three numbers (1, 5, 15 minute averages) count processes **running or waiting for CPU or in uninterruptible I/O wait**. Compare with the number of cores (`nproc`): on a 4-core machine a load of 4.0 means fully busy, 8.0 means a queue twice the capacity. A high load with low CPU use points to **I/O wait** (disk/network/NFS), not CPU.

**Foreground, background and job control**

```bash
long_task &                  # start in the background (still tied to this terminal/session)     jobs   fg %1   bg %1
# Ctrl+Z suspends the foreground job, then bg continues it in the background, fg brings it back
nohup long_task > out.log 2>&1 &     # ignore SIGHUP so it survives closing the terminal      disown -h %1   # detach a running job
tmux new -s work      # tmux/screen: persistent terminal sessions (detach with Ctrl+B then D, reattach with tmux attach -t work)
setsid cmd &          # run in its own session
```

**Signals** (software interrupts delivered to a process; `kill -l` lists them)

| Signal | No. | Default / use |
| --- | --- | --- |
| `SIGHUP` | 1 | terminal closed; many daemons **reload config** on it |
| `SIGINT` | 2 | Ctrl+C |
| `SIGQUIT` | 3 | Ctrl+\\ (terminate + core dump) |
| `SIGKILL` | 9 | **cannot be caught or ignored**; immediate kill, no cleanup |
| `SIGSEGV` | 11 | invalid memory access (segmentation fault) |
| `SIGTERM` | 15 | **polite termination request** (the default of `kill`); can be handled for graceful shutdown |
| `SIGSTOP` / `SIGCONT` | 19 / 18 | pause (cannot be caught) / resume |
| `SIGCHLD` | 17 | a child exited |
| `SIGUSR1/2` | 10 / 12 | application-defined |

```bash
kill 1234            # SIGTERM          kill -9 1234   # SIGKILL (last resort)     kill -HUP $(pidof nginx)
pkill -f 'celery worker'      killall python3      kill -STOP 1234; kill -CONT 1234
```

**Always try SIGTERM first**, wait, then escalate to SIGKILL: SIGKILL skips cleanup (temp files, locks, database flushes, child processes). Containers run your app as PID 1 and send SIGTERM on `docker stop`: handle it for graceful shutdown (Part 2 Docker notes).

**Priority:** `nice -n 10 cmd` starts a process with lower priority; `renice -n 5 -p 1234`; niceness ranges **-20 (highest priority) to 19 (lowest)**; only root may lower the number. `ionice` for disk I/O priority. `ulimit -a` shows per-process resource limits (open files, processes, memory).

**Kill by resource trouble:** the **OOM killer** terminates a process when memory runs out (look in `dmesg | grep -i 'killed process'` or `journalctl -k`); each process has an `oom_score`.

**Interview answer:** "I try SIGTERM first so the process can clean up, then SIGKILL if it hangs. A zombie is a dead process whose parent has not reaped it; I fix the parent. A process in D state is stuck in I/O and cannot be killed until it returns. Load average counts runnable and I/O-blocked tasks, so I compare it to the core count and check I/O wait."

**Loopholes:** `kill -9` is not the normal way to stop a service; you cannot kill a zombie (it is already dead); `&` alone does not protect from the terminal closing (use `nohup`, `setsid`, `tmux` or systemd); `kill` is a shell builtin and a program (builtin used first); `Ctrl+C` sends SIGINT to the foreground **process group**; `top`'s CPU percent can exceed 100% on multi-core machines.

---

## L5. Pipes, redirection and text-processing tools

**Standard streams:** file descriptors **0 = stdin**, **1 = stdout**, **2 = stderr**. A **pipe** `a | b` connects a's stdout to b's stdin (processes run concurrently; the pipeline's exit status is that of the **last** command unless `set -o pipefail`).

```bash
cmd > file          # stdout to file (overwrite)         cmd >> file        # append
cmd 2> err.log      # stderr only                         cmd > out.log 2>&1   # both to one file (order matters!)      cmd &> all.log   # bash shortcut
cmd < input.txt     # stdin from a file                    cmd > /dev/null 2>&1    # discard everything
cmd | tee out.log   # print AND save                       cmd | tee -a out.log     # append
sort << EOF         # here-document: multi-line input inline
b
a
EOF
grep foo <<< "$text"                          # here-string
diff <(sort a.txt) <(sort b.txt)              # process substitution: a command's output as a file
echo "today is $(date +%F)"                   # command substitution
find . -name '*.log' -print0 | xargs -0 -n1 gzip     # xargs builds commands from stdin; -print0/-0 handles spaces safely
```

**The toolbox**

| Tool | Common use and options |
| --- | --- |
| `cat`, `less`, `head -n 20`, `tail -n 50`, **`tail -f`** (follow live), `tail -F` (follow across rotation) | view files |
| **`grep`** | `-i` ignore case, `-n` line numbers, `-r` recursive, `-v` invert, `-c` count, `-l` file names only, `-w` whole word, `-E` extended regex, `-A3/-B3/-C3` context lines, `-o` only the match |
| **`sed`** | stream editor: `sed 's/old/new/g' f` (substitute), `sed -n '10,20p'` (print lines), `sed '/^#/d'` (delete matching lines), `sed -i 's/a/b/g' f` (in place; **macOS needs `-i ''`**) |
| **`awk`** | field-based processing: `awk '{print $1, $3}'`, `-F,` custom separator, `awk '$9 >= 500'` (condition), `awk '{s += $3} END {print s}'` (sum), `NR` (line number), `NF` (field count) |
| `cut -d, -f1,3` | extract columns; `paste`, `join` merge files |
| **`sort`** | `-n` numeric, `-r` reverse, `-k2` by column 2, `-u` unique, `-h` human sizes |
| **`uniq -c`** | collapse adjacent duplicates and count (input must be **sorted first**) |
| `wc -l/-w/-c` | count lines, words, bytes |
| `tr 'a-z' 'A-Z'`, `tr -d '\r'` | translate/delete characters |
| `diff -u a b`, `cmp`, `comm` | compare files |
| `jq '.items[].name'` | query JSON (\`curl -s url |
| `column -t`, `rev`, `nl`, `shuf`, `split`, `xxd` | formatting and utilities |

**Regex quick reference (grep -E / sed -E):** `.` any char, `*` zero or more, `+` one or more, `?` optional, `^` start, `$` end, `[0-9]` class, `[^a]` negated class, `(a|b)` alternation, `{n,m}` repeat, `\b` word boundary; escape metacharacters (`\.`) to match them literally.

**Worked example on a web server access log** (`access.log` lines look like `10.0.0.5 - - [28/Oct/2026:10:15:02 +0000] "GET /api/items HTTP/1.1" 200 512`; field 1 is the IP, field 7 the path, field 9 the status code, field 10 the bytes)

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head -10          # top 10 client IPs by request count
awk '{print $9}' access.log | sort | uniq -c | sort -nr                       # distribution of HTTP status codes
awk '$9 >= 500 {print $7}' access.log | sort | uniq -c | sort -nr | head     # which paths produce 5xx errors
awk '{s += $10} END {printf "%.1f MB\n", s/1048576}' access.log               # total bytes served
grep ' 404 ' access.log | awk '{print $7}' | sort -u                           # distinct missing URLs
grep -E 'ERROR|CRITICAL' app.log | tail -n 100                                 # latest errors
awk '{print substr($4, 2, 14)}' access.log | uniq -c                            # requests per minute (timestamp up to the minute)
find /var/log -type f -size +100M -exec ls -lh {} + 2>/dev/null                 # biggest logs
grep -rl 'old_api_url' src/ | xargs sed -i 's/old_api_url/new_api_url/g'        # replace across files (review with git diff!)
ps aux --sort=-%mem | head -5                                                    # top memory consumers
```

**Interview answer:** "stdin, stdout and stderr are file descriptors 0, 1 and 2. Pipes connect them between processes; `2>&1` after `> file` sends both streams to the file. For logs I combine grep to filter, awk to pick fields, sort and uniq -c to aggregate, and head to limit."

**Loopholes:** `cmd 2>&1 > file` sends stderr to the **terminal** (order matters: redirect stdout first, then `2>&1`); `uniq` only removes **adjacent** duplicates (sort first); `sort` is lexicographic unless `-n` (`10` sorts before `9`); `grep` returns exit status 1 when nothing matches (breaks `set -e` scripts); `cat file | grep` is a useless use of `cat`; `> file` **truncates** the file immediately, even when the command later fails (`sort f > f` empties `f`); `xargs` without `-0` mishandles spaces; `sed -i` has different syntax on macOS and BSD; never parse `ls` output.

---

## L6. Shell scripting

**Anatomy of a script**

```bash
#!/usr/bin/env bash                  # shebang: which interpreter runs the file (chmod +x script.sh, then ./script.sh)
set -euo pipefail                    # -e exit on error, -u error on unset variables, -o pipefail a pipeline fails if ANY stage fails
IFS=$'\n\t'                          # safer word splitting (optional)

name="${1:-world}"                   # positional argument with a default
count=$(ls -1 | wc -l)               # command substitution
echo "Hello, $name. This folder has $count entries. Script: $0, args: $#, last status: $?"
```

**Special variables:** `$0` script name, `$1..$9` arguments, `$#` argument count, `"$@"` all arguments (each preserved as a separate word: always use this form), `$*` all as one string, `$?` last exit status, `$$` current PID, `$!` PID of the last background job.

**Variables and expansion:** `${var:-default}` (use default if unset or empty), `${var:?message}` (abort with a message if unset), `${var#prefix}` / `${var%suffix}` (strip), `${var//old/new}` (replace all), `${#var}` (length), `$((a + b))` (integer arithmetic), arrays `arr=(a b c)`, `${arr[1]}`, `"${arr[@]}"`, `${#arr[@]}`. **Quote every expansion** (`"$var"`).

**Conditionals**

```bash
if [[ -f "$file" && -r "$file" ]]; then echo "readable file"
elif [[ -d "$file" ]]; then echo "directory"
else echo "missing" >&2; exit 1; fi

if [[ "$n" -gt 10 ]]; then ...; fi          # numeric tests: -eq -ne -lt -le -gt -ge   (or: if (( n > 10 )); then ...)
if [[ "$s" == "yes" ]]; then ...; fi        # string tests: == != -z (empty) -n (non-empty) and patterns: [[ $f == *.txt ]], regex: [[ $s =~ ^[0-9]+$ ]]
case "$1" in start) run;; stop) halt;; *) echo "usage: $0 start|stop" >&2; exit 2;; esac
```

**File tests:** `-e` exists, `-f` regular file, `-d` directory, `-L` symlink, `-r/-w/-x` permissions, `-s` non-empty, `-nt/-ot` newer/older than.

**Loops, functions, input**

```bash
for f in *.log; do gzip "$f"; done                         # glob, NOT `for f in $(ls)` (breaks on spaces)
for i in {1..5}; do echo "$i"; done          for ((i = 0; i < 3; i++)); do echo "$i"; done
while IFS= read -r line; do echo "got: $line"; done < input.txt        # read a file line by line (-r keeps backslashes)
until curl -fsS http://localhost:8000/healthz > /dev/null; do sleep 2; done   # wait until a service is up

log() { printf '%s %s\n' "$(date -Is)" "$*" >&2; }        # functions: arguments are $1.. inside; return sets the status (0-255)
need() { command -v "$1" > /dev/null || { echo "missing: $1" >&2; exit 1; }; }
read -rp "Continue? [y/N] " ans                            # prompt for input
```

**Error handling and cleanup**

```bash
tmp=$(mktemp -d)                                  # safe temporary directory
cleanup() { rm -rf "$tmp"; }
trap cleanup EXIT                                 # runs on normal exit AND on errors/signals (INT, TERM)
trap 'echo "failed at line $LINENO" >&2' ERR     # report where a command failed
lockfile=/tmp/job.lock
exec 9>"$lockfile"; flock -n 9 || { echo "already running" >&2; exit 1; }   # prevent overlapping runs (cron)
```

**Example 1: safe backup with retention**

```bash
#!/usr/bin/env bash
set -euo pipefail
SRC="${SRC:-/opt/app}"; DEST="${DEST:-/var/backups/app}"; KEEP_DAYS="${KEEP_DAYS:-7}"
[[ -d "$SRC" ]] || { echo "source $SRC not found" >&2; exit 1; }
mkdir -p "$DEST"
ts=$(date +%F_%H%M%S)
tar -czf "$DEST/app_${ts}.tgz" -C "$(dirname "$SRC")" "$(basename "$SRC")"
sha256sum "$DEST/app_${ts}.tgz" > "$DEST/app_${ts}.tgz.sha256"           # integrity record
find "$DEST" -name 'app_*.tgz*' -mtime +"$KEEP_DAYS" -delete              # retention: remove old backups
echo "backup ok: app_${ts}.tgz"
```

**Example 2: wait for health and fail the deploy if the service does not come up**

```bash
url="${1:?usage: $0 <health-url>}"
for attempt in $(seq 1 30); do
  if curl -fsS --max-time 2 "$url" > /dev/null; then echo "healthy after $attempt tries"; exit 0; fi
  sleep 2
done
echo "service did not become healthy" >&2; exit 1
```

**Example 3: report the five most common error messages in a log**

```bash
grep -E 'ERROR|CRITICAL' "${1:?log file}" | sed -E 's/^[^ ]+ [^ ]+ //' | sort | uniq -c | sort -nr | head -5
```

**Best practices:** start every serious script with `set -euo pipefail`; quote variables; check inputs and required commands; use `mktemp` and `trap` for cleanup; send errors to stderr with non-zero exit codes; use absolute paths and set `PATH` in cron jobs; make scripts **idempotent** (safe to run twice); log with timestamps; run `shellcheck script.sh` (static analysis catches most bugs); for anything complex (data structures, JSON, HTTP) prefer Python.

**Interview answer:** "I start scripts with `set -euo pipefail`, quote all expansions, validate arguments, clean up with a `trap`, and return meaningful exit codes. `[[ ]]` is the safer bash test; `-eq` compares numbers while `==` compares strings."

**Loopholes:** `[ $x = a ]` breaks when `$x` is empty or has spaces (use `[[ ]]` or quotes); `-eq` on non-numbers errors, `==` on numbers compares text (`"10" == "010"` is false); `set -e` does **not** trigger inside `if`/`&&`/`||` conditions or on failed commands in a pipeline unless `pipefail`; `grep` finding nothing returns 1 and exits a `set -e` script (use `|| true` deliberately); `cd dir && rm -rf *` is safe but `cd dir; rm -rf *` deletes the current directory when `cd` fails; **`rm -rf "$DIR/"` with an empty `$DIR` is `rm -rf /`** (use `${DIR:?}`); variables set in a pipeline's subshell (`cmd | while read`) vanish after the loop; `for f in $(ls)` and unquoted `$@` break on spaces; scripts run by cron have a minimal environment; Windows line endings (`\r\n`) break the shebang (`^M: bad interpreter`: fix with `dos2unix`).

---

## L7. Packages, systemd services, cron and logs

**Package management**

| Family | Tools | Typical commands |
| --- | --- | --- |
| Debian/Ubuntu | `apt`, `dpkg` | `sudo apt update` (refresh the index) · `sudo apt install nginx` · `apt search x` · `apt show x` · `sudo apt upgrade` · `sudo apt remove x` (keeps config) / `purge` (removes config) · \`dpkg -l |
| RHEL/Fedora/CentOS | `dnf`/`yum`, `rpm` | `sudo dnf install httpd` · `dnf update` · \`rpm -qa |
| Alpine (containers) | `apk` | `apk add --no-cache curl` |
| Universal | snap, flatpak, language managers (`pip`, `npm`) | prefer virtual environments for language packages |

Package managers resolve **dependencies**, verify **signatures**, and track installed files so upgrades and removals are clean. **Repositories** are listed in `/etc/apt/sources.list*` or `/etc/yum.repos.d/`; add third-party repos only from trusted sources with their signing key. Manual installs go in `/usr/local` or `/opt`; `./configure && make && sudo make install` bypasses the package manager (hard to uninstall: prefer packages or containers).

**systemd: the service manager (PID 1)**

```bash
sudo systemctl start|stop|restart|reload nginx        # act now          systemctl status nginx    # state, recent logs, PID, memory
sudo systemctl enable --now nginx                     # start at boot AND now       systemctl disable nginx       systemctl is-active nginx
systemctl list-units --type=service --state=failed    # what is broken           systemctl list-unit-files | grep enabled
sudo systemctl daemon-reload                          # REQUIRED after editing a unit file
journalctl -u nginx -f                                # follow a service's logs        journalctl -u nginx --since '1 hour ago' -p err
journalctl -b -p warning                              # this boot, warnings and worse    journalctl -k   # kernel messages (like dmesg)
systemd-analyze blame                                 # what slowed the boot        systemctl get-default / set-default multi-user.target
```

**A unit file for your own app** (`/etc/systemd/system/myapi.service`)

```ini
[Unit]
Description=My API service
After=network-online.target postgresql.service     # start ordering
Wants=network-online.target

[Service]
User=app                                           # never run as root
WorkingDirectory=/opt/api
EnvironmentFile=/etc/myapi.env                     # secrets and config (chmod 600, owned by root)
ExecStart=/opt/api/venv/bin/uvicorn app.main:app --host 127.0.0.1 --port 8000
Restart=on-failure                                 # auto-restart if it crashes
RestartSec=5
LimitNOFILE=65535                                  # raise the open-files limit
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target                         # enabled => started in the normal server target
```

Then `sudo systemctl daemon-reload && sudo systemctl enable --now myapi && journalctl -u myapi -f`. Stdout/stderr of the service go to the **journal** automatically (so log to stdout). `Type=simple` (default) means the `ExecStart` process is the service; `Type=forking` is for old daemons that background themselves; `Type=oneshot` for run-to-completion tasks.

**Scheduling: cron and systemd timers**

```
# crontab -e    (crontab -l lists, crontab -r DELETES everything)
# min  hour  day-of-month  month  day-of-week   command
*/5   *     *              *      *             /opt/scripts/health.sh >> /var/log/health.log 2>&1      # every 5 minutes
0     2     *              *      *             /opt/scripts/backup.sh                                   # daily at 02:00
30    3     *              *      1             /opt/scripts/weekly.sh                                   # Mondays at 03:30
0     0     1              *      *             /opt/scripts/monthly.sh                                  # first day of the month
@reboot                                         /opt/scripts/on_boot.sh                                  # at startup
```

- Cron runs with a **minimal environment** (small `PATH`, no login profile): use **absolute paths**, set `PATH=` and `SHELL=` at the top, redirect output (`>> log 2>&1`) because unhandled output is mailed or lost, make jobs **idempotent** and protect against overlap with `flock`.
- Server **time zone** matters (`timedatectl`); `%` has a special meaning in crontab lines (escape as `\%`); missed runs (machine off) are not replayed (use anacron or systemd timers with `Persistent=true`).
- **systemd timers** (`.timer` + `.service`) give logging via `journalctl`, dependency handling and catch-up; `at` runs a one-off job (`echo cmd | at 10:30`).

**Logs**

- Traditional: `/var/log/syslog` or `/var/log/messages` (system), `/var/log/auth.log` or `/var/log/secure` (logins, sudo, SSH), `/var/log/kern.log`, `/var/log/dpkg.log`, `/var/log/nginx/access.log` and `error.log`. Modern: the **systemd journal** (`journalctl`). **`logrotate`** (`/etc/logrotate.d/`) rotates, compresses and prunes logs so disks do not fill; applications must reopen files after rotation (or use `copytruncate`).
- Useful: `dmesg -T | tail` (kernel ring buffer: hardware errors, OOM kills), `last` / `lastb` (logins / failed logins), `tail -F` (follow across rotation), `journalctl --disk-usage` and `--vacuum-time=7d`.

**Interview answer:** "systemd starts services in dependency order, restarts them on failure, and captures their output in the journal. I write a unit with `User`, `EnvironmentFile`, `Restart=on-failure`, then `daemon-reload` and `enable --now`. For scheduled work I use cron with absolute paths and logging, or a systemd timer."

**Loopholes:** forgetting `daemon-reload` after editing a unit; `enable` does not start the service now (and `start` does not enable it at boot); cron jobs that work in your shell but fail in cron (environment/paths); `crontab -r` wipes all entries (typo next to `-e`); `apt update` and `apt upgrade` are different; running `pip install` system-wide can break OS tools (use a venv); a service using stdout logging with no journald limits can fill the disk; two cron instances overlapping.

---

## L8. Networking and SSH

(Protocol theory: Part 2, C1 to C12.)

```bash
ip addr                       # interfaces and IP addresses (replaces ifconfig)         ip -br a      ip link      ip route   # routing table, default gateway
ip route get 8.8.8.8          # which route/interface would be used
ss -tulpn                     # listening TCP/UDP sockets with owning processes (replaces netstat -tulpn)     ss -s   # summary     ss -tan state established | wc -l
ping -c 4 host                # ICMP reachability and latency       traceroute host / mtr host   # path and per-hop loss
dig +short example.com        # DNS lookup       dig example.com MX     dig @1.1.1.1 example.com     dig -x 8.8.8.8   # reverse
cat /etc/resolv.conf          # DNS servers      cat /etc/hosts   # local name overrides      resolvectl status
curl -v https://example.com/healthz                       # headers, TLS, timing details
curl -sS -o /dev/null -w '%{http_code} %{time_total}s\n' https://example.com      # status + total time
curl -X POST -H 'Content-Type: application/json' -d '{"a":1}' http://localhost:8000/v1/items
curl --resolve example.com:443:10.0.0.5 https://example.com        # test a specific backend IP without changing DNS
nc -zv host 5432              # is a TCP port reachable? (netcat)       nc -l 9000   # listen on a port
sudo tcpdump -i eth0 -nn port 80 -c 20                    # capture packets (Wireshark for analysis)
wget -c https://host/file.iso                              # download with resume
```

**Firewalls:** modern systems use **nftables** (iptables is the older interface); simple front ends: **ufw** (Ubuntu) and **firewalld** (RHEL).

```bash
sudo ufw default deny incoming && sudo ufw default allow outgoing
sudo ufw allow 22/tcp && sudo ufw allow 80,443/tcp && sudo ufw enable && sudo ufw status verbose
sudo iptables -L -n -v          sudo nft list ruleset
```

**Always allow SSH before enabling a firewall on a remote machine**, or you lock yourself out.

**SSH (Secure Shell)**

```bash
ssh-keygen -t ed25519 -C "asha@laptop"            # generate a key pair (private key stays secret; protect it with a passphrase)
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host    # install your PUBLIC key into the server's ~/.ssh/authorized_keys
ssh user@host      ssh -p 2222 user@host      ssh -i key.pem user@host      ssh user@host 'df -h'      # run one command remotely
scp file user@host:/tmp/           rsync -avz --progress src/ user@host:/dest/       # rsync: -a archive, -v verbose, -z compress (--delete mirrors deletions: dry-run with -n first)
ssh -L 5432:db.internal:5432 user@bastion         # LOCAL forward: localhost:5432 reaches db.internal:5432 through the bastion
ssh -R 8080:localhost:3000 user@server            # REMOTE forward: server:8080 reaches your local port 3000
ssh -D 1080 user@host                              # dynamic SOCKS proxy         ssh -J bastion user@private-host   # jump host (ProxyJump)
```

```
# ~/.ssh/config : shortcuts and per-host settings
Host prod
    HostName 203.0.113.10
    User deploy
    IdentityFile ~/.ssh/id_ed25519
    ProxyJump bastion
```

**How key authentication works:** the server stores your public key; at login it sends a challenge that only the holder of the private key can sign; the private key never leaves your machine. The first connection shows the host fingerprint (stored in `~/.ssh/known_hosts`); a later **"REMOTE HOST IDENTIFICATION HAS CHANGED"** warning means the server was rebuilt or someone is intercepting traffic: verify before removing the old entry.

**Server hardening (`/etc/ssh/sshd_config`, then `sudo systemctl reload ssh`):** `PasswordAuthentication no`, `PermitRootLogin no`, `PubkeyAuthentication yes`, optionally `AllowUsers deploy` and a non-default port (obscurity only), **fail2ban** to ban brute-force sources, keep SSH updated, use MFA or certificates for fleets. Keep a second session open while testing changes.

**Diagnosing connection errors**

| Symptom | Meaning and checks |
| --- | --- |
| **Connection refused** | the host answered but **nothing listens** on that port (service down, wrong port, bound to 127.0.0.1) or a firewall actively rejects: `ss -tulpn` on the server |
| **Connection timed out** | packets are **dropped or unroutable** (firewall/security group silently drops, wrong IP, host down): `ping`, `traceroute`, cloud security group rules |
| **Permission denied (publickey)** | wrong user, key not in `authorized_keys`, wrong permissions (`~/.ssh` 700, files 600, home not group-writable), wrong key offered (`ssh -v` / `-i`) |
| **Could not resolve hostname** | DNS problem: `dig`, `/etc/resolv.conf` |
| **Works by IP but not by name** | DNS or `/etc/hosts` |
| **Service reachable locally but not remotely** | bound to `127.0.0.1` instead of `0.0.0.0`, or firewall/security group blocks it |
| **Intermittent slow connections** | DNS delays (reverse lookup), packet loss (`mtr`), MTU issues |

**Interview answer:** "Refused means something answered with a reset (no listener or an explicit reject); timed out means packets were dropped. I use `ss -tulpn` to see what is listening and on which address, `curl -v` and `dig` to test the application and DNS, and `tcpdump` when I need to see packets."

**Loopholes:** `netstat` and `ifconfig` are deprecated (`ss` and `ip` replace them); `ping` blocked by a firewall does not mean the host is down; a service bound to `127.0.0.1` is invisible from other machines; `rsync src dest` vs `rsync src/ dest` (trailing slash changes whether the directory itself or its contents are copied); `rsync --delete` can erase the destination; never share or commit a private key; do not disable `StrictHostKeyChecking` for convenience on production paths.

---

## L9. Disk, memory, performance and the troubleshooting playbook

**Memory**

```bash
free -h                       # total, used, free, shared, buff/cache, AVAILABLE   (look at AVAILABLE, not 'free')
vmstat 1 5                    # r (run queue), b (blocked), si/so (swap in/out: should be ~0), us/sy/id/wa/st columns
top -o %MEM                   # sort by memory (htop is friendlier)       cat /proc/meminfo       ps aux --sort=-%mem | head
swapon --show                 # swap devices       dmesg -T | grep -i 'out of memory'
```

**Linux uses spare RAM as disk cache** (`buff/cache`), so "free" memory looks small on a healthy system; the cache is reclaimed when applications need it. **`available`** estimates memory usable without swapping. Real trouble signs: sustained **swap activity** (`si/so` > 0), **OOM kills** in `dmesg`, and rising `wa`. A memory leak shows as a process's RSS growing over time (`ps -o rss,vsz,cmd -p PID`, `pmap`, `smem`). **VSZ** (virtual size) is not RAM used; **RSS** is.

**CPU and load:** `top` CPU line: `us` user, `sy` kernel, `ni`, `id` idle, **`wa` I/O wait**, `hi/si` interrupts, **`st` steal** (virtual machine CPU taken by the hypervisor: noisy neighbour or an undersized instance). `mpstat -P ALL 1` (per core), `pidstat 1` (per process), `uptime` (load), `perf top` / `perf record` (profiling), flame graphs. **High load + low CPU** = tasks stuck in I/O wait; **high CPU + low load** = a single-threaded hot loop on one core.

**Disk and I/O**

```bash
df -h                          # filesystem usage       df -i   # inodes       du -xh --max-depth=1 / 2>/dev/null | sort -h | tail     # biggest directories (-x stays on one filesystem)
iostat -xz 1                   # per-device utilisation (%util), latency (await), queue (aqu-sz)         iotop -oPa   # per-process I/O
lsof +L1                       # deleted files still held open by a process (space not yet freed)         ncdu /var   # interactive disk usage
smartctl -a /dev/sda           # disk health (SMART)           lsblk -f   blkid   # filesystems and UUIDs
```

**Limits:** `ulimit -n` (open files per process, default often 1024), `ulimit -u` (processes); permanent limits in `/etc/security/limits.conf` or, for services, `LimitNOFILE=` in the systemd unit. System-wide: `cat /proc/sys/fs/file-max`; tune kernel parameters with `sysctl` (`sysctl net.core.somaxconn`; persist in `/etc/sysctl.d/`).

**Troubleshooting playbook: work from symptom to cause with evidence**

| Symptom | Investigate (in order) | Typical fixes |
| --- | --- | --- |
| **Server is slow** | `uptime` and `top` (load vs cores, `wa`, `st`) → `free -h`/`vmstat` (swapping?) → `iostat -xz 1` (disk saturated?) → `ss -s` (connections) → app logs and slow queries | kill/fix the runaway process, add RAM, move heavy work off the box, tune the DB, add caching |
| **Disk full** (`No space left on device`) | `df -h` → \`du -xh --max-depth=1 /var | sort -h`→`lsof +L1`(deleted-but-open logs) →`df -i`  (inodes) → Docker ( `docker system df\`), package caches, journal size |
| **Service will not start** | `systemctl status svc` → `journalctl -xeu svc` → run the `ExecStart` command manually as the service user → check config syntax (`nginx -t`) → file permissions/SELinux → port in use | fix config/permissions/env file, free the port |
| **Port already in use** (`Address already in use`) | \`ss -tulpn | grep :8080`or`lsof -i :8080\` |
| **Permission denied** | `ls -l` the file **and every parent directory** (`namei -l /path`) → which user runs the process (`ps -o user= -p PID`) → SELinux/AppArmor (`ls -Z`, `getenforce`, audit log) | fix owner/group/mode, add the user to the group, fix the label; do not `chmod 777` |
| **High CPU** | `top`/`pidstat 1` to find the process → `strace -c -p PID` or `perf top` → thread dump/profiler of the app | fix the loop/query, scale out, throttle |
| **Out of memory / process killed** | \`dmesg -T | grep -i kill`→`ps aux --sort=-%mem\` → trend RSS over time |
| **Too many open files** | `ulimit -n`, \`ls /proc/PID/fd | wc -l` ,  `lsof -p PID\` |
| **Cannot resolve names** | `dig`, `cat /etc/resolv.conf`, `resolvectl status`, test with IP | fix resolver, DNS records, search domains |
| **Cannot reach a service remotely** | `ss -tulpn` (listening address) → local `curl` → firewall (`ufw status`, cloud security groups) → routing | bind `0.0.0.0`, open the port, fix rules |
| **Cron job did not run** | `grep CRON /var/log/syslog` or `journalctl -u cron` → environment/paths → script permissions and shebang → output redirected? | absolute paths, `flock`, log output |
| **Clock/time problems** (cert errors, token expiry) | `timedatectl`, `chronyc tracking` | enable NTP (chrony/systemd-timesyncd), fix the time zone |
| **Server unreachable after reboot** | console access → `/etc/fstab` errors, network config, failed units (`systemctl --failed`) | fix fstab (use `nofail`), network settings |

**The USE method per resource (Part 5, H29):** for CPU, memory, disk and network ask: **U**tilization, **S**aturation (queues/waiting: run queue, swap, `aqu-sz`), **E**rrors (`dmesg`, interface error counters `ip -s link`). Change **one thing at a time**, record what you tried, and capture evidence (logs, `top` snapshots) **before** restarting, because restarts destroy the clues.

**Loopholes:** reading `free` as "almost no memory left" (check `available`); killing the highest-CPU process without understanding it; `df` full but `du` small = deleted-open files; `du` on `/` crossing into other mounts (use `-x`); increasing `ulimit` in the shell but not in the systemd unit; fixing symptoms by rebooting and never finding the cause; deleting logs that a running process still holds open (space is not freed: truncate with `: > file` or restart).

---

## L10. Security and hardening basics

**Principles:** **least privilege** (users and services get only the access they need), **defence in depth** (several layers), **minimize the attack surface** (fewer packages, fewer open ports), **keep it patched**, **log and monitor**.

**Checklist for a new server**

1. **Accounts and sudo:** create a normal admin user, use `sudo` instead of logging in as root, edit sudoers only with `visudo`, grant specific commands where possible (avoid blanket `NOPASSWD: ALL`), lock unused accounts (`usermod -L user`, shell `/usr/sbin/nologin` for service accounts). Membership of the **`docker` group is effectively root** access: grant it deliberately.
2. **SSH:** key-only login, `PermitRootLogin no`, `PasswordAuthentication no`, `fail2ban` or rate limiting, restrict with `AllowUsers`/firewall source ranges (L8).
3. **Firewall:** default deny inbound, open only needed ports (`ufw` or cloud security groups); verify with `ss -tulpn` that nothing unexpected listens, and with an outside scan from another host.
4. **Updates:** apply security patches regularly (`unattended-upgrades` on Debian/Ubuntu, `dnf-automatic` on RHEL); reboot for kernel updates; remove unused packages and services (`systemctl list-unit-files --state=enabled`).
5. **Services run unprivileged:** dedicated user per service, systemd hardening options (`NoNewPrivileges=true`, `ProtectSystem=strict`, `PrivateTmp=true`, `ReadWritePaths=`), bind to `127.0.0.1` and expose through a reverse proxy where possible.
6. **Permissions and secrets:** no world-writable files in sensitive places (`find / -xdev -type f -perm -0002 2>/dev/null`), audit setuid binaries (`find / -perm -4000 -type f 2>/dev/null`), config with secrets `chmod 600` and owned by the service user, **never pass secrets as command-line arguments** (visible in `ps` and shell history; use environment files, stdin or a secret manager), set `umask 027` for services where appropriate.
7. **Logging and auditing:** central log shipping, review `auth.log`/`journalctl -u ssh`, `last` and `lastb`, `sudo` logs, `auditd` for sensitive file access; alert on failed-login spikes and new users/keys.
8. **Encryption:** TLS for all services (Let's Encrypt with `certbot`, automatic renewal), disk encryption (LUKS) for laptops and sensitive volumes, encrypted backups.
9. **Integrity:** verify downloads with checksums and signatures (`sha256sum -c file.sha256`, `gpg --verify file.sig file`), install from signed repositories only, avoid `curl ... | sudo bash` from untrusted sources.
10. **Audit tools and baselines:** `lynis audit system`, CIS benchmarks, container image scanning, regular vulnerability scans.

**Mandatory access control: SELinux (RHEL family) and AppArmor (Ubuntu/SUSE)**

- Besides owner/group/other permissions, the kernel can enforce **policies on processes and labels**: a web server process may read only files labelled for web content even if Unix permissions allow more. Modes: **enforcing**, **permissive** (logs but does not block; for diagnosis), disabled.
- Commands: `getenforce`, `sudo setenforce 0` (temporary permissive for testing), `ls -Z` / `ps -eZ` (labels), `restorecon -Rv /var/www` (reset labels), `semanage fcontext`, `setsebool -P httpd_can_network_connect on` (policy switches), `ausearch -m avc -ts recent` / `audit2why` (why access was denied); AppArmor: `aa-status`, profiles in `/etc/apparmor.d/`.
- **Do not permanently disable SELinux to "fix" a permission problem**: find the denial in the audit log and correct the label or boolean.

**Interview answer:** "I give services their own unprivileged users, allow only key-based SSH, default-deny the firewall, patch regularly, keep secrets out of arguments and repositories, and review logs. If a service fails on RHEL even though Unix permissions look right, I check SELinux denials instead of disabling it."

**Loopholes:** `chmod 777` and running as root are the usual "quick fixes" that become vulnerabilities; changing the SSH port is not security; a closed firewall port does not protect a service bound to all interfaces inside a compromised network; passwords on the command line leak through `ps` and history (`HISTCONTROL=ignorespace`, or avoid); `sudo` with an editor or pager can escape to a shell (limit what can be run); backups on the same host are not backups; ignoring cloud-level controls (security groups, IAM) while hardening only the OS.

---

## L11. Linux behind containers

(Docker usage is in Part 6, B2; this is the Linux mechanism underneath, a favourite follow-up question.)

**A container is a normal Linux process (or group of processes) with a restricted view and limited resources.**

| Mechanism | What it provides |
| --- | --- |
| **Namespaces** | isolation of what the process can *see*: `pid` (own process tree, the app is PID 1), `net` (own interfaces, routing, ports), `mnt` (own mount tree), `uts` (hostname), `ipc`, `user` (UID mapping: root inside, unprivileged outside), `cgroup` |
| **cgroups** (control groups, v2) | limit and account for what the process can *use*: memory (`memory.max`), CPU (`cpu.max`, shares), I/O, process count (`pids.max`) |
| **Union/overlay filesystem** | layered, copy-on-write image layers plus a writable top layer (ephemeral) |
| **Capabilities and seccomp** | drop privileged powers and filter system calls (Docker's default profile blocks dangerous ones) |
| **chroot / pivot\_root** | change the root directory the process sees |

```bash
unshare --fork --pid --mount-proc bash          # a shell in a new PID namespace: `ps aux` shows only its own processes
sudo nsenter -t $(docker inspect -f '{{.State.Pid}}' mycontainer) -n ss -tulpn      # run a host tool inside a container's network namespace
systemd-run --scope -p MemoryMax=200M -p CPUQuota=50% ./heavy_job                   # cgroup limits without Docker
cat /sys/fs/cgroup/memory.max     cat /proc/self/cgroup     ps aux | grep 'node server.js'   # a container's processes are visible from the host
```

**Practical consequences**

- **Shared kernel:** a container is not a VM; a kernel bug or a privileged container can affect the host. Do not run untrusted code with `--privileged` or with the Docker socket mounted.
- **PID 1 inside the container** does not get default signal handling: a process that ignores SIGTERM will be killed by SIGKILL after the timeout, and orphaned children become zombies unless PID 1 reaps them (use `docker run --init` or `tini`, and handle SIGTERM in your app).
- **Exit codes:** `137` = 128 + 9 (**SIGKILL**, often the OOM killer when the memory limit was exceeded: check `docker inspect` -> `OOMKilled`), `143` = 128 + 15 (SIGTERM, a normal stop), `139` = segmentation fault, `126` = found but not executable, `127` = command not found, `130` = Ctrl+C.
- **Resource limits matter:** a JVM or Node process that reads the host's memory instead of the cgroup limit may use more than allowed and get killed: use container-aware flags/versions.
- **Debugging:** `docker exec -it c sh`, `docker logs`, `docker stats`, `docker top c`, and for minimal images `docker run --pid=container:c --net=container:c nicolaka/netshoot` (shares namespaces with the target).

**Loopholes:** "containers are lightweight VMs" is wrong; files written to the container layer vanish with the container (use volumes); `localhost` inside a container is the container itself; UID mismatches cause bind-mount permission errors; the writable layer is slow for heavy I/O.

---

## L12. Interview Q&A, practical scenarios and cheat sheet

### 40 rapid-fire questions (short answers)

1. **What happens when you run `ls`?** the shell parses the line, searches `PATH`, `fork`s, the child `exec`s `/bin/ls`, the shell `wait`s and shows the prompt.
2. **Hard link vs symbolic link?** hard link = another name for the same inode (same filesystem only); symlink = a file holding a path (can dangle, can cross filesystems).
3. **What is an inode?** the metadata record of a file (owner, mode, size, timestamps, block pointers); names live in directories.
4. **`df` says 100% but you deleted big files. Why?** a process still holds them open (`lsof +L1`), or inodes are exhausted (`df -i`).
5. **What does `chmod 755` mean?** owner `rwx`, group and others `r-x`.
6. **What does `x` mean on a directory?** permission to enter it and access items by path (`r` only lists names).
7. **What is umask 022?** removes write for group and others: new files 644, directories 755.
8. **What are setuid, setgid, sticky?** run as the file's owner / inherit group on a directory / only the owner can delete entries in a shared directory (`/tmp`).
9. **Difference between `su -` and `sudo`?** `su -` switches to another user with their password and environment; `sudo` runs one command with your own credentials according to sudoers.
10. **How do you stop a process gracefully?** `kill -TERM PID` (or `systemctl stop`), wait, then `kill -KILL` if needed.
11. **What is a zombie process?** an exited child whose parent has not collected its status; kill or fix the parent.
12. **What does load average mean?** average number of runnable plus I/O-blocked tasks over 1, 5 and 15 minutes; compare with core count.
13. **`kill -9` vs `kill -15`?** 9 = SIGKILL (uncatchable, no cleanup); 15 = SIGTERM (graceful).
14. **Foreground vs background; keep a job running after logout?** `&`, `jobs`, `bg`, `fg`; use `nohup`, `tmux`, or a systemd service.
15. **What are stdin, stdout, stderr?** file descriptors 0, 1, 2; redirect with `<`, `>`, `2>`; `2>&1` merges.
16. **`>` vs `>>` vs `|`?** overwrite, append, pipe stdout into the next command.
17. **Show the 10 most frequent IPs in an access log.** `awk '{print $1}' log | sort | uniq -c | sort -nr | head -10`.
18. **Find files bigger than 100 MB.** `find / -xdev -type f -size +100M 2>/dev/null`.
19. **Replace text in many files.** `grep -rl old . | xargs sed -i 's/old/new/g'` (review with version control).
20. **`grep` vs `egrep`/`-E`?** `-E` enables extended regex (`+`, `?`, `|`, `()` without backslashes).
21. **`sed` vs `awk`?** `sed` edits streams by pattern/line; `awk` processes fields and does calculations.
22. **Why quote variables in shell?** to prevent word splitting and globbing (`"$file"`).
23. **`$@` vs `$*`?** `"$@"` keeps each argument separate; `"$*"` joins them into one string.
24. **What does `set -euo pipefail` do?** stops on errors, on unset variables and on failures inside pipelines.
25. **`[ ]` vs `[[ ]]`?** `[ ]` is the POSIX test command; `[[ ]]` is a safer bash keyword (no word splitting, supports patterns and regex).
26. **What is the `PATH` variable?** ordered list of directories searched for executables.
27. **cron job not running, why?** minimal environment (paths), wrong permissions/shebang, no output redirect, wrong time zone, syntax; check `grep CRON /var/log/syslog`.
28. **How do you run a program at boot?** a systemd unit (`enable`), or `@reboot` in cron for simple cases.
29. **`systemctl enable` vs `start`?** enable = start at boot; start = start now; `enable --now` does both.
30. **Where are logs?** `/var/log/*` and the journal (`journalctl -u svc -f`).
31. **How do you check which process uses port 8080?** `ss -tulpn | grep :8080` or `lsof -i :8080`.
32. **Connection refused vs timed out?** refused = reached the host, nothing listening (or reject); timed out = packets dropped/unreachable.
33. **How does SSH key authentication work?** the server holds your public key and verifies a signature that only your private key can create.
34. **`scp` vs `rsync`?** `rsync` transfers only differences, can resume and mirror (`-a`, `--delete`); `scp` copies whole files.
35. **How do you check memory usage properly?** `free -h` and read **available**; `vmstat` for swap activity; `ps --sort=-%mem` for culprits.
36. **Why is free memory low on a healthy server?** Linux uses spare RAM as page cache; it is reclaimed on demand.
37. **What is the OOM killer?** the kernel kills a process when memory is exhausted (`dmesg | grep -i kill`).
38. **What are namespaces and cgroups?** isolation of what a process sees / limits on what it can use; the basis of containers.
39. **How would you find out why a service failed to start?** `systemctl status`, `journalctl -xeu svc`, run `ExecStart` manually, check config syntax, permissions, ports.
40. **What is the difference between a process and a thread (Linux view)?** both are tasks created by `clone`; threads share the address space (Part 2, A2).

### Practical scenarios (practise on a real machine)

1. **Disk is 98% full on `/var`.** `df -h`; `du -xh --max-depth=1 /var | sort -h | tail`; compress or delete old logs (`journalctl --vacuum-time=7d`, `logrotate -f`); `lsof +L1` for deleted-but-open files; restart the service that holds them.
2. **Deploy a Python API as a service.** create user `app`, install under `/opt/api`, venv, `/etc/myapi.env` (mode 600), unit file with `Restart=on-failure`, `daemon-reload`, `enable --now`, check `journalctl -u myapi`, put Nginx in front on 443 (L7, Part 6 B8).
3. **Find who is hammering the server.** `awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head`; check `ss -tan state established | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head`; block with `ufw deny from <ip>` or rate limit at the proxy.
4. **A script works in the shell but not in cron.** use absolute paths, set `PATH`, `cd` explicitly, log stdout/stderr to a file, source required environment, check permissions and line endings.
5. **A user cannot read a shared file.** `namei -l /path/to/file` (shows permissions of **every** directory in the path), check group membership (`id user`, re-login needed), ACLs (`getfacl`), SELinux labels (`ls -Z`).
6. **Locate the process that makes the server slow.** `top` (sort by CPU/memory), `vmstat 1`, `iostat -xz 1`, `pidstat -d 1` (I/O per process), `strace -c -p PID`.
7. **Create a backup cron job safely.** script from L6 Example 1, `flock`, logging, `crontab -e` entry `0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1`, test a restore.
8. **Harden a fresh VM in 10 minutes.** admin user + key, disable root/password SSH, `ufw` allow 22/80/443, `unattended-upgrades`, `fail2ban`, remove unused services (L10).

### Command cheat sheet

| Goal | Commands |
| --- | --- |
| Navigate and inspect | `pwd`, `ls -lah`, `cd`, `tree`, `stat`, `file`, `find`, `du -sh`, `df -h` |
| View/search text | `cat`, `less`, `head`, `tail -f`, `grep -rn`, `awk`, `sed`, `cut`, `sort`, `uniq -c`, `wc -l`, `jq` |
| Files | `cp -a`, `mv`, `rm -r` (careful), `mkdir -p`, `ln -s`, `touch`, `tar -czf/-xzf`, `rsync -avz`, `zip/unzip` |
| Permissions | `chmod`, `chown`, `chgrp`, `umask`, `getfacl/setfacl`, `id`, `sudo`, `visudo` |
| Processes | `ps aux`, `top/htop`, `pgrep`, `kill -TERM/-KILL`, `jobs/bg/fg`, `nohup`, `tmux`, `lsof`, `strace`, `nice/renice` |
| Services and schedule | `systemctl status/start/enable/daemon-reload`, `journalctl -u -f`, `crontab -e`, `logrotate` |
| Packages | `apt update/install/remove`, `dpkg -l`, `dnf install`, `rpm -qa` |
| Network | `ip a`, `ip route`, `ss -tulpn`, `ping`, `mtr`, `dig`, `curl -v`, `nc -zv`, `tcpdump`, `ufw` |
| Remote | `ssh`, `ssh-keygen`, `ssh-copy-id`, `scp`, `rsync`, `ssh -L/-R/-D`, `~/.ssh/config` |
| Performance | `uptime`, `free -h`, `vmstat 1`, `iostat -xz 1`, `pidstat`, `dmesg -T`, `ulimit -a` |
| Scripting | `set -euo pipefail`, `trap`, `shellcheck`, `mktemp`, `flock`, `xargs -0`, `read -r` |

### Can you do these without looking anything up? (self-test)

- [ ] Read a permission string and convert it to octal; apply `umask`
- [ ] Explain and demonstrate hard vs symbolic links and what frees disk space
- [ ] Find the top 10 IPs/URLs/status codes in a log with a one-line pipeline
- [ ] Stop a process gracefully, and explain zombie and D states
- [ ] Write a script with arguments, a loop, error handling and a `trap`
- [ ] Write a systemd unit and read its logs; write a cron entry and explain the pitfalls
- [ ] Diagnose "connection refused" vs "timed out" and find what listens on a port
- [ ] Set up key-based SSH, a port forward, and secure `sshd_config`
- [ ] Free disk space safely (including deleted-but-open files and inodes)
- [ ] Explain how containers use namespaces and cgroups

**Loopholes to remember for every Linux answer:** quote your variables; test destructive commands (`rm`, `find -delete`, `rsync --delete`, `sed -i`) with a dry run first; do not fix permission problems with `777` or by disabling SELinux; check evidence (`journalctl`, `dmesg`, `ss`) before changing anything; and know that behaviour differs slightly between distributions and between GNU and BSD/macOS tools.

---

*End of Part 8. This completes the series: Parts 1 to 6 (core), Part 7 (aptitude) and Part 8 (Linux). The master index lists Parts 1 to 6; Parts 7 and 8 are listed in my final message.*
