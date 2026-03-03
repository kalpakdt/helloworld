# 🌐 Topic 3: Networking — Zura Style

No CCNA lecture bullshit. Just what a Linux admin needs to know to keep servers talking.

---

## ⚡ The Networking Stack — One Line

```
TCP/IP → how data moves   |   DNS → how names resolve
NFS → how files are shared over network   |   SSH → how you get in securely
```

Know all four. Miss one and you're the guy who can't ping his own server.

---

## ⚡ Full Flow — Packet Life

```
YOU TYPE: ssh user@myserver.com
        │
        ▼
DNS RESOLVES: myserver.com → 192.168.1.10
        │
        ▼
TCP/IP: 3-way handshake → SYN → SYN-ACK → ACK
        │
        ▼
SSH: Encrypted tunnel established
        │
        ▼
YOU'RE IN 🔥
```

---

## 🔴 Stage 1 — TCP/IP (The Backbone)

**Everything runs on TCP/IP. If you don't know this, you don't know networking.**

### IP Addressing
```bash
# Check IP addresses
ip addr show                           # modern way
ip a                                   # short form
ifconfig                               # old way (still works)
ifconfig -a                            # all interfaces including down ones

# Check routing table
ip route show
ip r                                   # short form
route -n                               # old way

# Default gateway
ip route | grep default
netstat -rn                            # routing table (old way)

# Check interface stats
ip -s link show eth0                   # TX/RX bytes, errors
cat /proc/net/dev                      # raw interface stats
```

### Connectivity Testing
```bash
# Basic ping
ping 8.8.8.8                          # ICMP test
ping -c 4 8.8.8.8                     # 4 packets only
ping -i 0.2 8.8.8.8                   # faster interval

# Trace route (where packets go)
traceroute 8.8.8.8
tracepath 8.8.8.8                      # no root needed
mtr 8.8.8.8                           # live traceroute (best tool)

# Check open ports
ss -tulnp                              # modern netstat (MEMORIZE THIS)
ss -tulnp | grep 22                   # check if SSH is listening
netstat -tulnp                         # old way
lsof -i :80                           # what's using port 80
lsof -i TCP                           # all TCP connections

# Check if a remote port is open
nc -zv 192.168.1.10 22               # netcat port check
nc -zv -w3 192.168.1.10 80           # with 3s timeout
telnet 192.168.1.10 22               # old way (still useful)
curl -v telnet://192.168.1.10:22     # curl way

# Bandwidth test
iperf3 -s                             # start server
iperf3 -c 192.168.1.10               # run client test
```

### Network Config (Persistent)
```bash
# ── RHEL/CentOS 7 ──
# Edit interface file
vim /etc/sysconfig/network-scripts/ifcfg-eth0
# Key fields:
# BOOTPROTO=static (or dhcp)
# IPADDR=192.168.1.10
# NETMASK=255.255.255.0
# GATEWAY=192.168.1.1
# DNS1=8.8.8.8
# ONBOOT=yes

systemctl restart network             # apply (RHEL 7)
nmcli connection reload               # NetworkManager way

# ── RHEL/CentOS 8+ / Rocky / Alma ──
nmcli device status                   # list interfaces
nmcli connection show                 # list connections
nmcli connection modify eth0 ipv4.addresses 192.168.1.10/24
nmcli connection modify eth0 ipv4.gateway 192.168.1.1
nmcli connection modify eth0 ipv4.dns 8.8.8.8
nmcli connection modify eth0 ipv4.method manual
nmcli connection up eth0

# ── Ubuntu (Netplan) ──
vim /etc/netplan/00-installer-config.yaml
# network:
#   ethernets:
#     eth0:
#       dhcp4: no
#       addresses: [192.168.1.10/24]
#       gateway4: 192.168.1.1
#       nameservers:
#         addresses: [8.8.8.8, 8.8.4.4]
#   version: 2

netplan apply                         # apply changes
netplan try                           # test with auto-revert

# ── Hostname ──
hostnamectl set-hostname myserver
hostnamectl status
cat /etc/hostname
```

### TCP/IP Key Ports (Know These Cold)
```
22   → SSH
21   → FTP
23   → Telnet (don't use this shit)
25   → SMTP
53   → DNS
80   → HTTP
443  → HTTPS
111  → RPC (needed for NFS)
2049 → NFS
3306 → MySQL
5432 → PostgreSQL
6443 → Kubernetes API
8080 → HTTP alternate
```

---

## 🟢 Stage 2 — DNS (Domain Name System)

**Servers talk to each other by IP. Humans use names. DNS is the translator.**

```bash
# Basic DNS lookup
nslookup google.com                   # basic lookup
nslookup google.com 8.8.8.8          # use specific DNS server
dig google.com                        # detailed lookup
dig google.com A                      # A record (IPv4)
dig google.com AAAA                   # AAAA record (IPv6)
dig google.com MX                     # mail records
dig google.com NS                     # nameserver records
dig google.com TXT                    # text records
dig +short google.com                 # just the IP

# Reverse DNS (IP → name)
dig -x 8.8.8.8
nslookup 8.8.8.8
host 8.8.8.8

# Trace full DNS resolution path
dig +trace google.com

# Check which DNS server you're using
cat /etc/resolv.conf
systemd-resolve --status | grep DNS   # systemd systems

# Test DNS from specific server
dig @8.8.8.8 google.com               # query Google's DNS
dig @192.168.1.1 google.com           # query local DNS
```

### DNS Config Files
```bash
# DNS resolver config
cat /etc/resolv.conf
# nameserver 8.8.8.8
# nameserver 8.8.4.4
# search mydomain.local

# Local hostname resolution (checked BEFORE DNS)
cat /etc/hosts
# 192.168.1.10   myserver myserver.local
# Add entries here for quick local resolution

# Resolution order (hosts vs DNS vs NIS)
cat /etc/nsswitch.conf
# hosts: files dns    ← checks /etc/hosts first, then DNS
```

### DNS Troubleshooting
```bash
# DNS not resolving?
cat /etc/resolv.conf                  # check nameserver config
ping 8.8.8.8                         # if ping works but DNS fails → DNS issue
dig google.com                        # detailed DNS debug
systemctl status systemd-resolved     # check DNS service (Ubuntu)
systemctl restart systemd-resolved    # restart DNS resolver

# Flush DNS cache
systemd-resolve --flush-caches        # Ubuntu/systemd
nscd -I hosts                         # if nscd is running
service nscd restart                  # restart name service cache

# Check /etc/hosts overriding DNS
getent hosts myserver                 # what does the system resolve to?
```

---

## 🔵 Stage 3 — NFS (Network File System)

**Mount a remote directory like it's local. The original network share.**

### NFS Server Setup
```bash
# Install NFS server
dnf install nfs-utils -y              # RHEL/CentOS
apt install nfs-kernel-server -y      # Ubuntu

# Define what to share (/etc/exports)
vim /etc/exports
# Syntax: /path  client(options)
# Examples:
# /data  192.168.1.0/24(rw,sync,no_root_squash)
# /backup  *(ro,sync)                 ← read-only to everyone
# /home  192.168.1.10(rw,sync,root_squash)

# Export options explained:
# rw           = read-write
# ro           = read-only
# sync         = write to disk before responding (safe)
# async        = respond before writing (faster but risky)
# no_root_squash = remote root = local root (dangerous but sometimes needed)
# root_squash  = remote root mapped to nobody (safer default)
# no_subtree_check = don't verify subtree (better performance)

# Apply exports
exportfs -ra                          # reload exports
exportfs -v                           # show current exports
exportfs -a                           # export all

# Start and enable NFS
systemctl enable --now nfs-server     # RHEL
systemctl enable --now nfs-kernel-server  # Ubuntu
systemctl status nfs-server

# Firewall rules for NFS
firewall-cmd --add-service=nfs --permanent
firewall-cmd --add-service=rpc-bind --permanent
firewall-cmd --add-service=mountd --permanent
firewall-cmd --reload
```

### NFS Client Setup
```bash
# Install NFS client
dnf install nfs-utils -y
apt install nfs-common -y

# Show what server is sharing
showmount -e 192.168.1.10             # list exports from server
showmount -a 192.168.1.10            # show all mounts

# Mount NFS share
mkdir -p /mnt/nfs-data
mount -t nfs 192.168.1.10:/data /mnt/nfs-data
mount -t nfs4 192.168.1.10:/data /mnt/nfs-data   # NFS v4

# Mount with options
mount -t nfs -o rw,soft,intr 192.168.1.10:/data /mnt/nfs-data

# Verify mount
df -hT | grep nfs
mount | grep nfs

# Persistent NFS mount (/etc/fstab)
echo "192.168.1.10:/data  /mnt/nfs-data  nfs  defaults,_netdev  0  0" >> /etc/fstab
mount -a                              # test

# Unmount
umount /mnt/nfs-data
umount -l /mnt/nfs-data              # lazy unmount if busy
```

### NFS Troubleshooting
```bash
# Check NFS services running
systemctl status nfs-server
rpcinfo -p                            # list RPC services (111 and 2049 must be there)
rpcinfo -p 192.168.1.10              # check remote server RPC

# Permission denied on mount?
showmount -e 192.168.1.10            # check if share is exported
cat /etc/exports                      # check server config
exportfs -v                           # verify what's exported

# Stale NFS mount (server gone)
umount -l /mnt/nfs-data              # lazy unmount
umount -f /mnt/nfs-data              # force unmount

# NFS performance
nfsstat                               # NFS statistics
nfsstat -c                            # client stats
nfsstat -s                            # server stats
```

---

## 🟠 Stage 4 — SSH (Secure Shell)

**Your gateway to every server. Lock this down or get hacked.**

### Basic SSH Usage
```bash
# Connect to server
ssh user@192.168.1.10
ssh -p 2222 user@192.168.1.10        # non-standard port
ssh -i ~/.ssh/mykey user@server       # use specific key
ssh -v user@server                    # verbose (debug connection)
ssh -vvv user@server                  # very verbose

# Run command without interactive shell
ssh user@server "uptime"
ssh user@server "df -hT"

# Copy files (SCP)
scp file.txt user@server:/tmp/        # local → remote
scp user@server:/tmp/file.txt .       # remote → local
scp -r /localdir user@server:/remote/ # recursive directory copy

# Sync files (rsync over SSH)
rsync -avz /local/ user@server:/remote/
rsync -avz --delete /local/ user@server:/remote/   # mirror (delete extras)

# SSH tunnel (port forwarding)
ssh -L 8080:localhost:80 user@server  # local port forward
ssh -R 8080:localhost:80 user@server  # remote port forward
ssh -D 1080 user@server              # SOCKS proxy
```

### SSH Key Management
```bash
# Generate SSH key pair
ssh-keygen -t rsa -b 4096            # RSA 4096-bit
ssh-keygen -t ed25519                 # modern, shorter, more secure

# Copy public key to server
ssh-copy-id user@server               # easiest way
ssh-copy-id -i ~/.ssh/mykey.pub user@server

# Manual way
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys   # on remote server
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# SSH agent (avoid typing passphrase repeatedly)
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa
ssh-add -l                            # list loaded keys
```

### SSH Server Config (Hardening)
```bash
vim /etc/ssh/sshd_config

# KEY SETTINGS TO KNOW:
# Port 22                             ← change to non-standard (security)
# PermitRootLogin no                  ← NEVER allow root SSH
# PasswordAuthentication no           ← key-only auth
# PubkeyAuthentication yes            ← enable key auth
# MaxAuthTries 3                      ← limit brute force
# ClientAliveInterval 300             ← timeout idle sessions
# AllowUsers user1 user2              ← whitelist users
# AllowGroups sshusers                ← whitelist groups
# X11Forwarding no                    ← disable GUI forwarding
# Banner /etc/ssh/banner              ← legal warning banner

# Apply changes
systemctl reload sshd                 # reload config (no disconnect)
systemctl restart sshd                # full restart
sshd -t                               # test config before applying

# Check SSH logs
tail -f /var/log/secure               # RHEL — live SSH logs
tail -f /var/log/auth.log             # Ubuntu
journalctl -u sshd -f                 # systemd way
grep "Failed password" /var/log/secure | tail -20   # brute force attempts
grep "Accepted" /var/log/secure                      # successful logins
```

### SSH Troubleshooting
```bash
# Connection refused?
systemctl status sshd                 # is sshd running?
ss -tulnp | grep 22                  # is port 22 listening?
firewall-cmd --list-all               # is firewall blocking?

# Permission denied?
ls -la ~/.ssh/                        # check permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
ls -la /home/user/                    # home dir should not be world-writable

# Debug connection
ssh -vvv user@server 2>&1 | head -50  # verbose output
tail -50 /var/log/secure              # server-side logs

# Too many auth failures (key agent has too many keys)
ssh -o IdentitiesOnly=yes -i ~/.ssh/specific_key user@server
```

---

## 📊 Stage 5 — Key Commands Reference (Interview Speed Round)

| Task | Command |
|---|---|
| Show IP addresses | `ip a` |
| Show routing table | `ip r` |
| Show open ports | `ss -tulnp` |
| DNS lookup | `dig domain.com` |
| Trace route | `mtr 8.8.8.8` |
| Test port open | `nc -zv host port` |
| Show NFS exports | `showmount -e server` |
| Mount NFS | `mount -t nfs server:/path /mnt` |
| SSH with key | `ssh -i key user@host` |
| Copy SSH key | `ssh-copy-id user@host` |
| Check SSH logs | `tail -f /var/log/secure` |
| Reload SSHD | `systemctl reload sshd` |

---

## 🚨 Stage 6 — Troubleshooting Flow

```
Server unreachable?
        │
        ├─ Can you ping the IP?
        │       ├─ No → IP/routing issue → check ip addr, ip route
        │       └─ Yes → Go next
        │
        ├─ Can you resolve DNS?
        │       ├─ No → DNS issue → check /etc/resolv.conf, dig
        │       └─ Yes → Go next
        │
        ├─ Is the port open?
        │       ├─ No → Service down OR firewall blocking → ss -tulnp, firewall-cmd
        │       └─ Yes → Go next
        │
        └─ SSH still failing?
                ├─ Permission denied → key/auth issue → check authorized_keys perms
                └─ Connection reset → sshd config issue → check /var/log/secure
```

---

## 🧩 Key Files Cheatsheet

```
/etc/hosts                    ← local name resolution
/etc/resolv.conf              ← DNS servers
/etc/nsswitch.conf            ← resolution order (files/dns)
/etc/hostname                 ← system hostname
/etc/sysconfig/network-scripts/  ← RHEL network config
/etc/netplan/                 ← Ubuntu network config
/etc/exports                  ← NFS shares (server)
/etc/fstab                    ← persistent mounts (NFS client)
/etc/ssh/sshd_config          ← SSH server config
~/.ssh/authorized_keys        ← allowed public keys for user
~/.ssh/known_hosts            ← trusted server fingerprints
/var/log/secure               ← RHEL SSH/auth logs
/var/log/auth.log             ← Ubuntu SSH/auth logs
```

---

## ⚡ Full Networking Decision Flow

```
Networking problem?
        │
        ├─ Can't reach IP? → TCP/IP issue
        │       └── ip addr, ip route, ping, mtr
        │
        ├─ Can't resolve hostname? → DNS issue
        │       └── dig, nslookup, /etc/resolv.conf, /etc/hosts
        │
        ├─ Can't access NFS share? → NFS issue
        │       └── showmount, rpcinfo, exportfs, firewall
        │
        └─ Can't SSH in? → SSH issue
                └── sshd status, port 22, authorized_keys, /var/log/secure
```

---

**Zura's Networking Interview Rules:**
- They WILL ask "Server can't reach the internet — how do you troubleshoot?" → ping IP → ping DNS → dig hostname → check routes. That exact flow.
- They WILL ask "How do you secure SSH?" → Disable root login, key-only auth, change port, MaxAuthTries, AllowUsers. Fire these fast.
- They WILL ask "What's the difference between NFS v3 and v4?" → NFSv4 uses single port (2049), has built-in security (Kerberos), stateful protocol. NFSv3 needs portmapper (111), stateless.

---
*Zura's Topic 3 Networking Sheet — Know this or get roasted in interviews* 🔥
