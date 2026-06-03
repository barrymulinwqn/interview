# Unix/Linux — Interview Fundamentals

## Core Architecture

```
Hardware
  └── Kernel (process mgmt, memory mgmt, device drivers, filesystem)
        └── Shell (bash, zsh, sh)
              └── User Space (applications, utilities)
```

### Key Kernel Subsystems
| Subsystem | Role |
|-----------|------|
| Process Scheduler | CPU time allocation |
| Memory Manager | Virtual memory, paging, swapping |
| VFS | Abstraction over filesystems (ext4, xfs, nfs) |
| Network Stack | TCP/IP, sockets |
| Device Drivers | Hardware abstraction |

---

## Filesystem

### Standard Directory Layout (FHS)
| Path | Purpose |
|------|---------|
| `/bin` | Essential user binaries |
| `/sbin` | Essential system binaries |
| `/etc` | Configuration files |
| `/var` | Variable data (logs, spools) |
| `/tmp` | Temporary files |
| `/home` | User home directories |
| `/opt` | Optional/third-party software |
| `/proc` | Virtual FS: kernel & process info |
| `/sys` | Virtual FS: device/kernel info |
| `/dev` | Device files |
| `/usr` | Secondary hierarchy (non-essential binaries) |
| `/lib` | Shared libraries |
| `/mnt`, `/media` | Mount points |

### File Types
```
-  regular file
d  directory
l  symbolic link
c  character device
b  block device
p  named pipe (FIFO)
s  socket
```

### Permissions
```
rwxr-xr--
│││ │││ │││
│││ │││ └── other: read only
│││ └────── group: read+execute
└────────── owner: read+write+execute

Octal: 755 = rwxr-xr-x, 644 = rw-r--r--
```

```bash
chmod 755 script.sh
chmod u+x,g-w file
chown user:group file
chown -R www-data:www-data /var/www

# Special bits
chmod u+s file       # SUID: execute as file owner
chmod g+s dir        # SGID: new files inherit group
chmod +t /tmp        # Sticky: only owner can delete
```

### ACLs (Access Control Lists)
```bash
getfacl file
setfacl -m u:username:rwx file
setfacl -m g:groupname:rx file
setfacl -x u:username file       # remove ACL entry
```

---

## Process Management

### Process States
- **R** — Running/Runnable
- **S** — Sleeping (interruptible)
- **D** — Sleeping (uninterruptible, usually I/O)
- **Z** — Zombie (terminated, not yet reaped)
- **T** — Stopped

### Process Commands
```bash
ps aux                          # all processes
ps -ef --forest                 # tree view
pgrep -a nginx                  # find by name
kill -9 <pid>                   # SIGKILL (force)
kill -15 <pid>                  # SIGTERM (graceful)
killall nginx
pkill -f "python app.py"

# Background / foreground
command &                       # run in background
jobs                            # list background jobs
fg %1                           # bring job 1 to foreground
bg %1                           # resume job 1 in background
nohup command &                 # immune to SIGHUP
disown %1                       # detach from shell
```

### System Monitoring
```bash
top                             # real-time process monitor
htop                            # enhanced top
vmstat 1                        # virtual memory stats
iostat -x 1                     # I/O stats
sar -u 1 5                      # CPU utilization (5 samples)
mpstat -P ALL 1                 # per-CPU stats
free -h                         # memory usage
uptime                          # load averages (1, 5, 15 min)
```

### Load Average
Three values: 1-min, 5-min, 15-min average number of runnable + uninterruptible processes. Compare to CPU count: load of 4.0 on 4-core system = 100% utilization.

---

## Memory Management

```bash
free -h
cat /proc/meminfo
vmstat -s

# Page cache & swap
swapon -s                       # swap usage
swapoff -a / swapon -a          # disable/enable swap
echo 3 > /proc/sys/vm/drop_caches  # clear page cache (use carefully)
```

### OOM Killer
When memory is exhausted, the kernel OOM killer terminates processes. Check:
```bash
dmesg | grep -i "oom\|killed process"
grep -i oom /var/log/syslog
```

---

## Networking

### Essential Commands
```bash
ip addr show                    # interface addresses
ip route show                   # routing table
ss -tulnp                       # listening ports (modern netstat)
netstat -tulnp                  # legacy
ping -c 4 host
traceroute host / tracepath host
mtr host                        # combined ping+traceroute
nslookup / dig host             # DNS queries
curl -v http://url              # HTTP testing
wget -O- http://url
```

### Firewall
```bash
# iptables
iptables -L -n -v               # list rules
iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# firewalld
firewall-cmd --list-all
firewall-cmd --add-port=8080/tcp --permanent
firewall-cmd --reload

# UFW (Ubuntu)
ufw status
ufw allow 22/tcp
```

### Network Config Files
| File | Purpose |
|------|---------|
| `/etc/hosts` | Static hostname resolution |
| `/etc/resolv.conf` | DNS resolver config |
| `/etc/network/interfaces` | Debian network config |
| `/etc/sysconfig/network-scripts/` | RHEL/CentOS network config |
| `/etc/nsswitch.conf` | Name service switch order |

---

## Disk & Storage

```bash
df -h                           # filesystem disk usage
du -sh /var/log/                # directory disk usage
du -sh * | sort -rh | head -20  # find large directories
lsblk                           # list block devices
fdisk -l                        # partition table
blkid                           # block device UUIDs
mount /dev/sdb1 /mnt/data
umount /mnt/data
/etc/fstab                      # persistent mounts

# Filesystem check
fsck /dev/sdb1                  # check filesystem (unmounted)
```

### LVM (Logical Volume Manager)
```bash
pvs / pvdisplay                 # physical volumes
vgs / vgdisplay                 # volume groups
lvs / lvdisplay                 # logical volumes
lvextend -L +10G /dev/vg0/lv0   # extend LV
resize2fs /dev/vg0/lv0          # resize ext4
xfs_growfs /mountpoint          # resize xfs
```

---

## System Services (systemd)

```bash
systemctl status nginx
systemctl start | stop | restart | reload nginx
systemctl enable | disable nginx     # persist across reboots
systemctl list-units --type=service
journalctl -u nginx -f               # follow service logs
journalctl --since "1 hour ago"
journalctl -p err..emerg             # filter by priority
systemctl daemon-reload              # reload unit files
```

### Creating a systemd Service
```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

---

## Logging

```bash
# Log locations
/var/log/syslog         # general system messages (Debian)
/var/log/messages       # general system messages (RHEL)
/var/log/auth.log       # authentication logs
/var/log/secure         # authentication (RHEL)
/var/log/kern.log       # kernel logs
/var/log/dmesg          # boot messages

tail -f /var/log/syslog
grep -i error /var/log/syslog
zcat /var/log/syslog.*.gz | grep pattern   # search rotated logs

# logrotate
cat /etc/logrotate.conf
cat /etc/logrotate.d/nginx
```

---

## User & Group Management

```bash
useradd -m -s /bin/bash -G sudo username
usermod -aG docker username
userdel -r username             # remove user + home dir
groupadd devteam
passwd username

id username                     # show uid, gid, groups
whoami
who / w                         # logged-in users
last                            # login history
```

### sudoers
```bash
visudo                          # safe edit sudoers
# /etc/sudoers.d/myapp
appuser ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp
```

---

## Performance Tuning

### Kernel Parameters (sysctl)
```bash
sysctl -a                       # list all params
sysctl net.core.somaxconn       # check value
sysctl -w net.core.somaxconn=65535   # set temporarily
# Persist in /etc/sysctl.d/99-custom.conf
```

### Common Tuning Parameters
```
# Network
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_keepalive_time = 60

# File handles
fs.file-max = 1000000
# Per-process limit: ulimit -n / /etc/security/limits.conf
```

---

## Common Interview Questions

### Q1: What happens when you type `ls -l` and press Enter?
1. Shell parses the command
2. Shell forks a child process (`fork()`)
3. Child calls `execve()` to replace itself with `ls`
4. Kernel loads `/bin/ls` binary
5. `ls` reads directory entries via `getdents()` syscall
6. Output written to stdout (fd 1), displayed on terminal
7. `ls` exits, shell reaps the child (`wait()`)

### Q2: Difference between process and thread?
- **Process**: independent execution unit with its own memory space, file descriptors, PID
- **Thread**: lightweight execution unit within a process, sharing memory space and resources
- Context switch between processes is heavier than between threads

### Q3: What is a zombie process?
A process that has completed execution but whose parent hasn't called `wait()` to read its exit status. Visible as `Z` in `ps`. Freed when parent calls `wait()` or when parent terminates. Long-lived zombies indicate a bug in the parent.

### Q4: What is the difference between hard link and symbolic link?
| | Hard Link | Symbolic Link |
|---|---|---|
| Points to | inode (data blocks) | path (filename) |
| Cross-filesystem | No | Yes |
| Broken if target deleted | No (data remains) | Yes (dangling link) |
| Works on dirs | No (usually) | Yes |

### Q5: How does `fork()` + `exec()` work?
`fork()` creates an exact copy of the calling process (copy-on-write). The child then calls `exec()` to replace its memory image with a new program. This is how shells execute commands.

### Q6: What is inode?
A data structure storing file metadata (permissions, owner, timestamps, size, data block pointers) but NOT the filename. Filenames are stored in directory entries that map names to inodes.

### Q7: Explain /proc filesystem
Virtual filesystem created by the kernel on-the-fly. Contains runtime system info:
- `/proc/<pid>/` — per-process info (maps, fd, status)
- `/proc/meminfo` — memory stats
- `/proc/cpuinfo` — CPU info
- `/proc/sys/` — tunable kernel parameters

### Q8: What are signals? Name common ones.
Asynchronous notifications sent to processes:
| Signal | Number | Meaning |
|--------|--------|---------|
| SIGHUP | 1 | Hangup / reload config |
| SIGINT | 2 | Interrupt (Ctrl+C) |
| SIGKILL | 9 | Force kill (cannot be caught) |
| SIGTERM | 15 | Graceful termination |
| SIGSTOP | 19 | Stop process (cannot be caught) |
| SIGCHLD | 17 | Child status changed |

### Q9: What is the sticky bit?
When set on a directory (like `/tmp`), only the file owner, directory owner, or root can delete/rename files within it — even if the directory is world-writable.

### Q10: How would you find which process is using a port?
```bash
ss -tulnp | grep :8080
lsof -i :8080
fuser 8080/tcp
```
