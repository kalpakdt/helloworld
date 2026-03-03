# 🖥️ Topic 1: Operating Systems — Zura Style

No textbook garbage. Just what actually matters when admins fuck up servers.

---

## ⚡ The OS Family Tree — One Line

```
RHEL → CentOS → AlmaLinux/Rocky   (RPM Family)
Ubuntu → Debian                    (DEB Family)
AIX → IBM UNIX                     (Proprietary)
Solaris → Oracle UNIX              (Proprietary)
```

Know the family. Everything else flows from it.

---

## 🔴 Stage 1 — RHEL (Red Hat Enterprise Linux)

**The enterprise workhorse. Most interviews will grill you on this.**

- Subscription-based. You pay Red Hat. They support your ass.
- Package manager: **YUM** (older) / **DNF** (RHEL 8+)
- Config lives in `/etc/` like a normal OS should

```bash
# Check RHEL version
cat /etc/redhat-release
cat /etc/os-release

# Update everything
dnf update -y
yum update -y        # older systems

# Install a package
dnf install httpd -y

# List installed packages
rpm -qa
rpm -qi httpd        # info on specific package

# Check repos
dnf repolist
dnf repolist all

# Enable/disable repo
dnf config-manager --enable rhel-9-baseos-rpms
dnf config-manager --disable some-repo

# Register with Red Hat (subscription)
subscription-manager register --username=user --password=pass
subscription-manager attach --auto
subscription-manager status
```

**Key RHEL paths to know:**
```
/etc/yum.repos.d/        ← repo files live here
/var/log/messages        ← main system log
/etc/sysconfig/          ← network, service configs
/etc/redhat-release      ← version file
```

---

## 🟡 Stage 2 — CentOS (Community Enterprise OS)

**Free RHEL clone. Most orgs ran this before Red Hat killed it.**

- CentOS 7 = still alive, EOL June 2024 — dead now
- CentOS Stream = rolling release, upstream of RHEL — not stable for prod
- Rocky Linux / AlmaLinux = the real CentOS replacements now

```bash
# Check CentOS version
cat /etc/centos-release

# CentOS 7 — still uses yum
yum install vim -y
yum remove vim -y
yum list installed

# CentOS Stream / Rocky / Alma — uses dnf
dnf install vim -y

# Check if system is EOL
cat /etc/centos-release
# If it says CentOS Linux 7 → tell them to migrate NOW
```

**CentOS vs RHEL vs Rocky — interviewers love this:**

| Thing | RHEL | CentOS 7 | CentOS Stream | Rocky/Alma |
|---|---|---|---|---|
| Cost | Paid | Free | Free | Free |
| Support | Red Hat | Community | Community | Community |
| Stability | Production | Production | Rolling | Production |
| Future | ✅ Alive | ❌ Dead | ⚠️ Upstream | ✅ Alive |

---

## 🟠 Stage 3 — Ubuntu

**The dev favorite. Debian-based. APT is king here.**

```bash
# Check Ubuntu version
lsb_release -a
cat /etc/os-release

# Package management
apt update                    # refresh package list
apt upgrade -y                # upgrade all packages
apt install nginx -y          # install package
apt remove nginx              # remove (keeps config)
apt purge nginx               # remove + nuke config
apt autoremove                # clean orphaned packages

# Search for a package
apt search nginx
apt show nginx                # package info

# List installed
dpkg -l
dpkg -l | grep nginx

# Repo sources
cat /etc/apt/sources.list
ls /etc/apt/sources.list.d/

# Add a repo (PPA)
add-apt-repository ppa:somedev/somerepo
apt update

# Fix broken packages (life saver)
apt --fix-broken install
dpkg --configure -a
```

**Key Ubuntu paths:**
```
/etc/apt/sources.list        ← main repo file
/var/log/syslog              ← main system log (NOT /var/log/messages)
/etc/netplan/                ← network config (Ubuntu 18+)
/etc/apt/sources.list.d/     ← additional repos
```

---

## 🔵 Stage 4 — AIX (IBM Unix)

**Old enterprise beast. Runs on IBM POWER hardware. Don't bullshit this one.**

- IBM's own Unix — not Linux. Behaves differently.
- Package manager: **installp** and **rpm** both exist
- Uses **LVM** heavily — logical volumes everywhere

```bash
# Check AIX version
oslevel -s
oslevel -r              # release level

# OS info
uname -a

# Package management (AIX way)
lslpp -l                        # list installed filesets
lslpp -l | grep openssh         # check specific package
installp -aXgd /dev/cd0 all     # install from media
installp -u fileset_name        # uninstall

# Disk and LVM (AIX loves LVM)
lspv                    # list physical volumes
lsvg                    # list volume groups
lsvg rootvg             # details of rootvg
lslv                    # list logical volumes

# Process management
ps -ef
topas                   # AIX top equivalent (not top)

# Network
ifconfig -a
netstat -an
no -o                   # network options (AIX specific)

# Services (AIX uses SRC — System Resource Controller)
lssrc -a                # list all services
startsrc -s inetd       # start a service
stopsrc -s inetd        # stop a service
```

**AIX Golden Rule:** Don't assume Linux commands work. `topas` not `top`. `lslpp` not `rpm -qa`. `lsvg` not `vgdisplay`.

---

## 🟣 Stage 5 — Solaris (Oracle Unix)

**Sun Microsystems baby. Oracle owns it now. ZFS and Zones are its superpowers.**

- Uses **SMF** (Service Management Facility) — not systemd
- Package manager: **pkg** (IPS) on Solaris 11+, **pkgadd** on older
- Famous for **ZFS** and **Solaris Zones** (containers before Docker existed)

```bash
# Check Solaris version
uname -a
cat /etc/release

# Package management (Solaris 11+ IPS)
pkg install vim                 # install
pkg uninstall vim               # remove
pkg update                      # update all
pkg list                        # list installed
pkg search vim                  # search

# Older Solaris (pkgadd)
pkgadd -d /cdrom/pkgname        # install
pkgrm PKGname                   # remove
pkginfo                         # list installed

# Service management (SMF — not systemd!)
svcs -a                         # list all services
svcs svc:/network/ssh:default   # status of SSH
svcadm enable ssh               # enable service
svcadm disable ssh              # disable
svcadm restart ssh              # restart
svccfg                          # configure services

# Zones (Solaris containers)
zoneadm list -v                 # list all zones
zoneadm -z myzone boot          # boot a zone
zlogin myzone                   # login to zone
zonecfg -z myzone info          # zone config

# ZFS (Solaris invented this)
zfs list                        # list datasets
zpool status                    # pool health
zpool list                      # list pools
```

**SMF vs systemd — interviewers ask this:**

| Thing | Solaris SMF | Linux systemd |
|---|---|---|
| List services | `svcs -a` | `systemctl list-units` |
| Start service | `svcadm enable svc` | `systemctl start svc` |
| Stop service | `svcadm disable svc` | `systemctl stop svc` |
| Restart | `svcadm restart svc` | `systemctl restart svc` |
| Logs | `svcs -x` | `journalctl -u svc` |

---

## 🔍 Stage 6 — Cross-OS Comparison (Interview Gold)

**The one table that makes you look smart:**

| Feature | RHEL/CentOS | Ubuntu | AIX | Solaris |
|---|---|---|---|---|
| Package Mgr | DNF/YUM | APT | installp | pkg/pkgadd |
| Init System | systemd | systemd | SRC | SMF |
| Log location | `/var/log/messages` | `/var/log/syslog` | `/var/adm/messages` | `/var/adm/messages` |
| Network config | `/etc/sysconfig/network-scripts/` | `/etc/netplan/` | `smitty` | `ipadm` |
| Version check | `cat /etc/redhat-release` | `lsb_release -a` | `oslevel -s` | `cat /etc/release` |
| Hardware arch | x86_64 | x86_64/ARM | IBM POWER | SPARC/x86 |

---

## 🔥 Common Health Check Commands — All OS

```bash
# CPU usage
top
htop
mpstat -P ALL 1        # per-core stats

# Memory
free -h
vmstat -s

# Disk
df -hT                 # disk usage with filesystem type
du -sh /var/log/*      # directory sizes
iostat -x 1            # disk I/O stats

# Running processes
ps aux --sort=-%cpu | head -10     # top CPU hogs
ps aux --sort=-%mem | head -10     # top memory hogs

# System uptime and load
uptime
w

# OS and kernel version
uname -r               # kernel version
uname -a               # everything
cat /etc/os-release    # OS details

# Check failed services immediately
systemctl --failed
journalctl -b -p err   # boot errors
```

---

## ⚡ Full OS Decision Flow

```
What OS are you on?
        │
        ├─ RPM-based? → RHEL / CentOS / Rocky / Alma
        │       └── Use: dnf, yum, rpm, /var/log/messages
        │
        ├─ DEB-based? → Ubuntu / Debian
        │       └── Use: apt, dpkg, /var/log/syslog
        │
        ├─ IBM hardware? → AIX
        │       └── Use: lslpp, installp, topas, lsvg, SRC
        │
        └─ Oracle/Sun hardware? → Solaris
                └── Use: pkg, SMF, svcadm, ZFS Zones
```

---

## 🧩 Key Files Cheatsheet

```
# RHEL/CentOS
/etc/redhat-release          ← version
/etc/yum.repos.d/            ← repos
/var/log/messages            ← system log
/etc/sysconfig/network-scripts/  ← network

# Ubuntu
/etc/os-release              ← version
/etc/apt/sources.list        ← repos
/var/log/syslog              ← system log
/etc/netplan/                ← network

# AIX
/etc/motd                    ← banner
/var/adm/messages            ← system log
/etc/filesystems             ← filesystem table (not /etc/fstab)

# Solaris
/etc/release                 ← version
/var/adm/messages            ← system log
/etc/vfstab                  ← filesystem table (not /etc/fstab)
```

---

**Zura's OS Interview Rule:** They WILL ask "What's the difference between RHEL and Ubuntu?" — that table above is your answer. They WILL ask about AIX or Solaris to see if you panic. Don't panic. Know `oslevel -s` and `svcs -a` and you're already ahead of 80% of candidates. 🔥

---
*Zura's Topic 1 OS Sheet — Know this or get roasted in interviews*
