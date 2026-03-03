# 💀 L2 Linux — The New Shit You Didn't Cover — Zura Style

One file. Five topics. Zero fluff. Read it or cry in the interview.

---

# 🔐 SECTION 1: User & Access Management
### LDAP / AD / sudo / PAM — Because "useradd" alone won't cut it

No textbook AAA bullshit. Just how Linux actually authenticates humans.

---

## ⚡ Auth Flow — One Line

```
User types password
        │
        ▼
PAM checks rules (/etc/pam.d/)
        │
        ▼
SSSD talks to LDAP/AD
        │
        ▼
Access granted or go fuck yourself
```

---

## 👤 Local User Management (Basics First)

```bash
# Users
useradd -m -s /bin/bash john          # create with home + shell
useradd -u 1500 -g admins john        # custom UID + group
usermod -aG wheel john                # add to wheel (sudo group)
usermod -L john                       # lock account
usermod -U john                       # unlock account
userdel -r john                       # delete + nuke home dir

# Password
passwd john                           # set password
chage -l john                         # password aging info
chage -M 90 john                      # expire every 90 days
chage -E 2025-12-31 john             # account expires date
chage -d 0 john                       # force password change now

# Groups
groupadd admins
groupmod -n newadmins admins          # rename group
gpasswd -a john admins                # add to group
gpasswd -d john admins                # remove from group
groups john                           # what groups is john in?
id john                               # UID/GID info

# Who's logged in
who
w
last                                  # login history
lastb                                 # failed logins (bad)
lastlog                               # last login per user
```

---

## 🔑 sudo Configuration

```bash
# The only safe way to edit sudoers
visudo                                # validates syntax before saving
visudo -f /etc/sudoers.d/john        # edit custom file (recommended)

# Key syntax patterns
john    ALL=(ALL)    ALL              # john can run everything as anyone
john    ALL=(ALL)    NOPASSWD: ALL    # no password — DON'T DO THIS IN PROD
john    ALL=(root)   /sbin/reboot    # only reboot, nothing else
%wheel  ALL=(ALL)    ALL             # entire wheel group

# Better practice — put in sudoers.d NOT sudoers
echo "john ALL=(ALL) NOPASSWD:/bin/systemctl" > /etc/sudoers.d/john
chmod 440 /etc/sudoers.d/john

# Check sudo access
sudo -l -U john                      # what can john do?
sudo -l                              # what can I do?

# Sudo logs
tail -f /var/log/secure | grep sudo  # RHEL
tail -f /var/log/auth.log | grep sudo # Ubuntu
```

---

## 🏢 LDAP / AD Integration (sssd + realmd)

**realmd = joins AD. sssd = talks to LDAP/AD.**

```bash
# ── ACTIVE DIRECTORY JOIN ──

# Install deps
dnf install sssd realmd oddjob oddjob-mkhomedir adcli samba-common-tools -y
apt install sssd realmd adcli samba-common-bin krb5-user -y

# Discover AD domain
realm discover company.com

# Join the domain
realm join --user=Administrator company.com
realm join -v --user=Administrator company.com  # verbose

# Verify join
realm list                           # show domains
id user@company.com                  # test AD user
getent passwd user@company.com       # check NSS

# Allow specific users/groups
realm permit user@company.com
realm permit -g "Linux Admins@company.com"
realm deny --all                     # deny all except permitted

# ── SSSD CONFIG ──
cat /etc/sssd/sssd.conf
# [sssd]
# domains = company.com
# services = nss, pam
#
# [domain/company.com]
# ad_domain = company.com
# krb5_realm = COMPANY.COM
# realmd_tags = manages-system joined-with-adcli
# cache_credentials = True
# id_provider = ad
# auth_provider = ad
# access_provider = ad
# ad_gpo_access_control = permissive

systemctl restart sssd
sssctl domain-status company.com     # check SSSD domain health
sssctl user-checks user@company.com  # debug user auth

# ── LDAP (non-AD) ──
dnf install openldap-clients sssd sssd-ldap -y

# Test LDAP query
ldapsearch -x -H ldap://192.168.1.20 -b "dc=company,dc=com" "(uid=john)"
ldapsearch -x -D "cn=admin,dc=company,dc=com" -W -H ldap://192.168.1.20 -b "dc=company,dc=com"

# SSSD for LDAP
# /etc/sssd/sssd.conf
# [domain/LDAP]
# id_provider = ldap
# auth_provider = ldap
# ldap_uri = ldap://192.168.1.20
# ldap_search_base = dc=company,dc=com
# cache_credentials = true

# Auto-create home dirs on first login
authselect select sssd with-mkhomedir --force
systemctl enable --now oddjobd
```

---

## 🔒 PAM (Pluggable Authentication Modules)

**PAM controls every auth decision. Fuck with it wrong = locked out.**

```bash
# PAM config dir
ls /etc/pam.d/
# system-auth      ← RHEL main auth stack
# common-auth      ← Ubuntu main auth stack
# sshd             ← SSH specific
# su               ← su command
# sudo             ← sudo auth

# PAM module types:
# auth       → verify identity
# account    → account checks (expired? locked?)
# password   → password changes
# session    → session setup/teardown

# PAM control flags:
# required   → must pass, but continue
# requisite  → must pass, stop immediately
# sufficient → if pass, skip rest
# optional   → doesn't matter

# Password quality (/etc/security/pwquality.conf)
vim /etc/security/pwquality.conf
# minlen = 12             ← minimum length
# dcredit = -1            ← at least 1 digit
# ucredit = -1            ← at least 1 uppercase
# lcredit = -1            ← at least 1 lowercase
# ocredit = -1            ← at least 1 special char
# maxrepeat = 3           ← no more than 3 same chars
# difok = 5               ← 5 chars must differ from old

# Account lockout (faillock)
vim /etc/security/faillock.conf
# deny = 5                ← lock after 5 fails
# unlock_time = 900       ← 15 min lockout

# Check locked accounts
faillock --user john
faillock --reset --user john          # unlock manually

# PAM limits (/etc/security/limits.conf)
* soft nproc 1024                     # max processes
* hard nofile 65536                   # max open files
john hard maxlogins 3                 # max 3 sessions
```

---

## 🔍 Auth Troubleshooting

```bash
# Debug SSSD
sssctl domain-status company.com
sssctl user-checks john@company.com
cat /var/log/sssd/sssd_company.com.log

# Kerberos tickets (AD auth)
klist                                # show tickets
kinit john@COMPANY.COM              # get ticket manually
kdestroy                             # remove tickets

# PAM debug (test auth)
pamtester sshd john authenticate
authselect check                     # validate config

# Who can sudo?
cat /etc/sudoers
ls /etc/sudoers.d/
grep -r "john" /etc/sudoers.d/
```

---
---

# ⚙️ SECTION 2: Core Services
### systemd / cron / rsyslog / sshd — The daily grind

---

## ⚡ Services Flow — One Line

```
Boot → systemd starts all → cron schedules jobs → rsyslog logs everything → sshd lets you in
```

---

## 🔧 systemd (You Should Know This Cold)

```bash
# Service management
systemctl start|stop|restart|reload|status httpd
systemctl enable httpd               # start at boot
systemctl disable httpd              # don't start at boot
systemctl enable --now httpd         # enable + start right now
systemctl mask httpd                 # completely block starting
systemctl unmask httpd

# List services
systemctl list-units --type=service
systemctl list-units --type=service --state=failed
systemctl list-units --type=service --state=running

# Boot analysis
systemd-analyze                      # total boot time
systemd-analyze blame                # slowest services
systemd-analyze critical-chain       # dependency chain

# Journalctl (logs)
journalctl -u httpd                  # service logs
journalctl -u httpd -f               # follow (tail)
journalctl -u httpd -n 50            # last 50 lines
journalctl -u httpd --since "1 hour ago"
journalctl -b                        # current boot
journalctl -b -1                     # previous boot
journalctl -p err                    # errors only
journalctl -p err -b                 # errors since boot

# Custom service unit file
vim /etc/systemd/system/myapp.service
# [Unit]
# Description=My App
# After=network.target
#
# [Service]
# Type=simple
# User=appuser
# ExecStart=/usr/local/bin/myapp
# Restart=always
# RestartSec=10
#
# [Install]
# WantedBy=multi-user.target

systemctl daemon-reload              # reload after editing unit file
systemctl enable --now myapp

# Targets (runlevels)
systemctl get-default                # current target
systemctl set-default multi-user.target
systemctl isolate rescue.target      # switch now (no reboot)
```

---

## ⏰ cron (Scheduled Jobs)

```bash
# User crontab
crontab -l                           # list
crontab -e                           # edit
crontab -r                           # DELETE (don't fat-finger this)
crontab -l -u john                   # another user's crontab

# Cron syntax:
# * * * * * command
# │ │ │ │ │
# │ │ │ │ └─ Day of week (0=Sun, 7=Sun)
# │ │ │ └─── Month (1-12)
# │ │ └───── Day of month (1-31)
# │ └─────── Hour (0-23)
# └───────── Minute (0-59)

# Examples:
0 2 * * * /scripts/backup.sh         # daily 2am
*/5 * * * * /scripts/check.sh        # every 5 min
0 0 * * 0 /scripts/weekly.sh         # every Sunday midnight
0 9 1 * * /scripts/monthly.sh        # 1st of month, 9am

# System cron dirs (no user — root runs these)
ls /etc/cron.d/                      # custom cron files
ls /etc/cron.daily/                  # scripts run daily
ls /etc/cron.weekly/
ls /etc/cron.monthly/
cat /etc/crontab                     # system crontab

# Cron logs
grep cron /var/log/cron              # RHEL
grep cron /var/log/syslog            # Ubuntu
journalctl -u crond

# Restrict cron
cat /etc/cron.allow                  # only these users
cat /etc/cron.deny                   # block these users
```

---

## 📜 rsyslog (Log Management)

```bash
# Config
vim /etc/rsyslog.conf

# Facility.Severity format:
# kern.crit             ← kernel critical
# mail.*                ← all mail logs
# *.emerg               ← everything emergencies
# auth,authpriv.*       ← auth logs

# Severity levels (high to low):
# emerg > alert > crit > err > warning > notice > info > debug

# Local log rules
*.info;mail.none;authpriv.none;cron.none    /var/log/messages
authpriv.*                                  /var/log/secure
mail.*                                      /var/log/maillog
cron.*                                      /var/log/cron

# Forward logs to remote syslog server (SIEM)
*.* @192.168.1.100           # UDP
*.* @@192.168.1.100          # TCP (reliable)
*.* @@192.168.1.100:6514     # specific port

# Log rotation (/etc/logrotate.d/)
vim /etc/logrotate.d/myapp
# /var/log/myapp/*.log {
#     daily
#     rotate 30
#     compress
#     delaycompress
#     missingok
#     notifempty
#     postrotate
#         systemctl reload myapp
#     endscript
# }

logrotate -d /etc/logrotate.d/myapp  # dry run
logrotate -f /etc/logrotate.d/myapp  # force rotate now

# Test rsyslog
logger "test message from Zura"      # send test log
tail -f /var/log/messages            # watch it appear

# Restart rsyslog
systemctl restart rsyslog
rsyslogd -N1                         # validate config
```

---

## 🔐 sshd (Advanced Config)

```bash
vim /etc/ssh/sshd_config

# The good stuff beyond basics:
Port 2222                             # non-standard port
AddressFamily inet                    # IPv4 only
ListenAddress 192.168.1.10           # bind to specific IP
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
MaxAuthTries 3
MaxSessions 5
ClientAliveInterval 300
ClientAliveCountMax 2
Banner /etc/ssh/banner
AllowUsers john admin
AllowGroups sshusers admins
DenyUsers contractor temp

# Match blocks (per-user/IP overrides)
Match User sftp_only
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no

Match Address 192.168.1.0/24
    PermitRootLogin yes               # allow root from internal only

# Apply
sshd -t                               # test config
systemctl reload sshd                 # apply without killing sessions

# SSH logs
journalctl -u sshd -f
grep "Failed password" /var/log/secure | awk '{print $11}' | sort | uniq -c | sort -rn
# ^ top brute force IPs
```

---
---

# 🖥️ SECTION 3: VM Environments
### VMware / KVM — Containers before containers were cool

---

## ⚡ VM Flow — One Line

```
Physical host (hypervisor) → runs multiple VMs → each VM is isolated Linux
```

---

## 🔵 VMware (Guest OS side)

```bash
# Install VMware Tools (open-vm-tools)
dnf install open-vm-tools -y
systemctl enable --now vmtoolsd

# Verify tools running
vmware-toolbox-cmd -v                 # version
vmware-toolbox-cmd stat speed         # CPU speed
vmware-toolbox-cmd disk list          # disk info

# Check if you're on VMware
dmidecode | grep -i vmware
dmesg | grep -i vmware
lspci | grep -i vmware
cat /sys/class/dmi/id/product_name    # VMware Virtual Platform

# Disk rescan after adding VM disk
echo "- - -" > /sys/class/scsi_host/host0/scan
ls -l /dev/sd*                        # check new disk appeared
lsblk                                 # block devices

# Memory balloon driver
lsmod | grep vmw_balloon              # memory balloon active

# Network in VMware
ip a                                  # check ens192 or eth0
nmcli device status
```

---

## 🟢 KVM (Kernel-Based VM — Hypervisor Side)

```bash
# Check KVM support
egrep -c '(vmx|svm)' /proc/cpuinfo   # >0 = hardware virt supported
lsmod | grep kvm                      # kvm module loaded
kvm-ok                                # direct check (ubuntu)

# Install KVM stack
dnf install qemu-kvm libvirt virt-install virt-manager bridge-utils -y
apt install qemu-kvm libvirt-daemon-system virt-manager bridge-utils -y

systemctl enable --now libvirtd

# virsh — the main CLI (memorize this)
virsh list                            # running VMs
virsh list --all                      # all VMs including stopped
virsh start myvm                      # start VM
virsh shutdown myvm                   # graceful shutdown
virsh destroy myvm                    # force off (HARD KILL)
virsh reboot myvm                     # reboot
virsh suspend myvm                    # pause
virsh resume myvm                     # resume

# VM info
virsh dominfo myvm                    # VM details
virsh domstats myvm                   # runtime stats
virsh vcpuinfo myvm                   # CPU info
virsh dommemstat myvm                 # memory stats

# Console access
virsh console myvm                    # serial console (Ctrl+] to exit)

# Snapshots
virsh snapshot-create-as myvm snap1 "Before upgrade"
virsh snapshot-list myvm             # list
virsh snapshot-revert myvm snap1     # rollback
virsh snapshot-delete myvm snap1     # delete

# Storage pools
virsh pool-list                       # list pools
virsh pool-info default               # pool details
virsh vol-list default                # volumes in pool

# Networking
virsh net-list                        # networks
virsh net-info default                # network info
brctl show                            # bridge interfaces

# Clone a VM
virt-clone --original myvm --name myvm-clone --auto-clone

# Create new VM
virt-install   --name newvm   --ram 2048   --vcpus 2   --disk path=/var/lib/libvirt/images/newvm.qcow2,size=20   --os-variant rhel9   --network bridge=virbr0   --cdrom /iso/rhel9.iso   --graphics none   --console pty,target_type=serial
```

**KVM Troubleshooting:**
```bash
# Permission denied
usermod -aG libvirt $USER
usermod -aG kvm $USER

# VM won't start
journalctl -u libvirtd
virsh dominfo myvm                   # state check

# Storage issues
qemu-img info /var/lib/libvirt/images/myvm.qcow2
qemu-img check /var/lib/libvirt/images/myvm.qcow2

# Network not working in VM
virsh net-start default
virsh net-autostart default
```

---
---

# 🗂️ SECTION 4: LVM (Logical Volume Manager)
### Because partitions are static bullshit and you need flexibility

---

## ⚡ LVM Stack — One Line

```
Physical Disk → PV (Physical Volume) → VG (Volume Group) → LV (Logical Volume) → Filesystem
```

---

## 🔧 LVM Setup

```bash
# ── PHYSICAL VOLUMES ──
pvcreate /dev/sdb /dev/sdc           # init disks as PVs
pvdisplay                             # detailed PV info
pvs                                   # quick list
pvscan                                # scan for PVs
pvremove /dev/sdb                     # remove PV

# ── VOLUME GROUPS ──
vgcreate myvg /dev/sdb /dev/sdc      # create VG from PVs
vgdisplay myvg                        # detailed info
vgs                                   # quick list
vgscan
vgextend myvg /dev/sdd               # add disk to VG
vgreduce myvg /dev/sdb               # remove disk from VG
vgremove myvg                         # delete VG

# ── LOGICAL VOLUMES ──
lvcreate -n data -L 50G myvg         # create 50G LV
lvcreate -n logs -l 100%FREE myvg   # use all remaining
lvcreate -n swap -L 4G myvg         # swap LV
lvdisplay                             # detailed info
lvs                                   # quick list
lvscan                                # scan all LVs
lvremove /dev/myvg/data              # delete LV (DATA GONE)

# ── FORMAT AND MOUNT ──
mkfs.ext4 /dev/myvg/data
mkfs.xfs /dev/myvg/logs
mount /dev/myvg/data /data

# Persistent mount
echo "/dev/myvg/data /data ext4 defaults 0 0" >> /etc/fstab
```

---

## 📈 Extending LVM (The Real Value)

```bash
# Scenario: /data is full, add space

# Step 1: Add new disk
pvcreate /dev/sdd

# Step 2: Add to existing VG
vgextend myvg /dev/sdd

# Step 3: Extend LV
lvextend -L +20G /dev/myvg/data     # add 20GB
lvextend -l +100%FREE /dev/myvg/data # add all available

# Step 4: Resize filesystem (ONLINE - NO UNMOUNT)
resize2fs /dev/myvg/data            # EXT4 — no downtime
xfs_growfs /data                    # XFS — mount point not device

# One-liner extend + resize (EXT4)
lvextend -r -L +20G /dev/myvg/data  # -r auto resizes FS

# Verify
df -hT /data
lvs
```

---

## 📸 LVM Snapshots

```bash
# Create snapshot (consistent backup point)
lvcreate -s -n data_snap -L 5G /dev/myvg/data
lvs                                  # snap shows in list

# Mount snapshot read-only
mount -o ro /dev/myvg/data_snap /mnt/snap

# Restore from snapshot (DESTROYS CURRENT DATA)
umount /data
lvconvert --merge /dev/myvg/data_snap
lvchange -ay /dev/myvg/data

# Remove snapshot
umount /mnt/snap
lvremove /dev/myvg/data_snap
```

---

## 🚨 LVM Troubleshooting

```bash
# LV not found after reboot
vgscan
lvscan
vgchange -ay                         # activate all VGs

# Missing PV (disk removed/failed)
vgdisplay | grep "Act PV"           # partial VG
pvs --partial                        # show partial info

# Disk full — check PV space
vgs                                  # VFree column
pvs                                  # PFree column

# Verify LVM metadata
vgcfgbackup myvg                    # backup metadata
vgcfgrestore myvg                   # restore metadata (recovery)
```

---
---

# ⏰ SECTION 5: NTP (Network Time Protocol)
### Time sync: Wrong time = broken TLS, failed Kerberos, fucked logs

---

## ⚡ Time Sync — One Line

```
Hardware clock (BIOS) → System clock (kernel) → NTP syncs to internet → All servers same time
```

---

## 🔴 chrony (Modern — RHEL 7+, Ubuntu 20+)

```bash
# Install
dnf install chrony -y
apt install chrony -y

systemctl enable --now chronyd

# Config
vim /etc/chrony.conf
# pool 2.rhel.pool.ntp.org iburst    ← use pool
# server 192.168.1.1 iburst prefer   ← internal NTP server
# makestep 1.0 3                     ← big step if >1sec in first 3 adjustments
# rtcsync                            ← sync hardware clock
# allow 192.168.1.0/24               ← allow this subnet to use us as server

systemctl restart chronyd

# Check sync status
chronyc tracking                     # detailed sync info
chronyc sources                      # NTP sources list
chronyc sources -v                   # verbose
chronyc sourcestats                  # stats per source
chronyc activity                     # online/offline servers

# Manual force sync
chronyc makestep                     # force immediate step
timedatectl set-ntp true             # enable NTP
timedatectl status                   # check NTP sync status
```

---

## 🟡 ntpd (Legacy — older RHEL/SUSE)

```bash
# Install
dnf install ntp -y

vim /etc/ntp.conf
# server 0.pool.ntp.org iburst
# server 192.168.1.1 prefer iburst

systemctl enable --now ntpd

# Check sync
ntpq -p                              # peer list (reach=377 = good)
ntpstat                              # sync status
ntpdate -q pool.ntp.org              # test without setting

# Force sync (ntpd must be stopped)
systemctl stop ntpd
ntpdate pool.ntp.org
systemctl start ntpd
```

---

## 🔧 Hardware Clock vs System Clock

```bash
# Hardware clock (BIOS)
hwclock --show                       # show BIOS time
hwclock --hctosys                    # sync BIOS → system
hwclock --systohc                    # sync system → BIOS

# System clock
timedatectl                          # full time status
timedatectl list-timezones | grep Asia
timedatectl set-timezone Asia/Kolkata
date                                 # current time
date +"%Y-%m-%d %H:%M:%S"           # formatted

# Check if NTP synced
timedatectl | grep "NTP synchronized"
chronyc tracking | grep "System time"
```

---

## 🚨 NTP Troubleshooting

```bash
# Time not syncing?
systemctl status chronyd
chronyc sources                      # check server reachability
ping pool.ntp.org                    # DNS + connectivity

# Firewall blocking NTP?
firewall-cmd --add-service=ntp --permanent
firewall-cmd --reload
# NTP uses UDP 123

# Big time jump (>1000 seconds, chrony refuses)
systemctl stop chronyd
ntpdate pool.ntp.org                 # force reset
systemctl start chronyd

# Log
journalctl -u chronyd
cat /var/log/chrony/*.log
```

---
---

# 🧩 MASTER KEY FILES CHEATSHEET

```
# User & Access
/etc/passwd                ← user accounts
/etc/shadow                ← hashed passwords
/etc/group                 ← group definitions
/etc/sudoers               ← sudo rules (visudo)
/etc/sudoers.d/            ← custom sudo files
/etc/pam.d/                ← PAM configs
/etc/security/pwquality.conf ← password policy
/etc/security/faillock.conf  ← lockout policy
/etc/sssd/sssd.conf        ← SSSD (AD/LDAP)
/var/log/sssd/             ← SSSD logs

# Core Services
/etc/systemd/system/       ← custom service units
/etc/crontab               ← system cron
/var/spool/cron/           ← user crontabs
/etc/cron.d/               ← package cron jobs
/etc/rsyslog.conf          ← syslog config
/etc/logrotate.d/          ← log rotation
/etc/ssh/sshd_config       ← SSH server

# VMs
/etc/libvirt/              ← libvirt configs
/var/lib/libvirt/images/   ← VM disk images
/var/log/libvirt/          ← KVM logs

# LVM
/etc/lvm/lvm.conf          ← LVM config
/etc/lvm/backup/           ← VG metadata backups

# NTP
/etc/chrony.conf           ← chrony config
/etc/ntp.conf              ← ntpd config
/var/log/chrony/           ← chrony logs
```

---

# ⚡ FULL DECISION FLOWS

```
Auth problem?
        ├─ Local user? → /etc/passwd, /etc/shadow, faillock
        ├─ AD/LDAP? → sssd status, realm list, klist
        └─ SSH denied? → authorized_keys, /var/log/secure, PAM

Service broken?
        ├─ systemctl status + journalctl -u
        ├─ cron not running? → crond status, /var/log/cron
        ├─ logs missing? → rsyslog status, rsyslogd -N1
        └─ SSH broken? → sshd -t, reload, Match blocks

Disk full?
        ├─ df -hT → where is it full?
        ├─ LVM? → lvextend -r + resize2fs/xfs_growfs
        └─ No LVM? → pray or migrate to LVM

Time issues?
        ├─ chronyc tracking → synced?
        ├─ Firewall? → UDP 123
        └─ Big jump? → ntpdate force then restart chronyd

VM problem?
        ├─ VMware → vmtoolsd, dmidecode, dmesg
        └─ KVM → virsh dominfo, journalctl -u libvirtd
```

---

**Zura's L2 Interview Truth Bomb:**
- They WILL ask "AD user can't SSH in." → sssd status → realm list → klist → sshd_config AllowGroups → PAM
- They WILL ask "Extend /data without downtime." → pvcreate → vgextend → lvextend -r. Done.
- They WILL ask "Cron job not running." → crontab -l syntax → crond running? → /var/log/cron → check user restrictions
- They WILL ask "NTP out of sync." → chronyc tracking → sources → firewall UDP 123 → makestep

---
*Zura's L2 New Topics Sheet — Know this or get roasted in BOTH interviews* 🔥💀
