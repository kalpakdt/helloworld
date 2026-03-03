# 🛠️ Topic 4: Patch Management — Zura Style

No compliance lecture. Just how to actually keep systems updated without breaking production.

---

## ⚡ The Patch Flow — One Line

```
Check → Test → Stage → Deploy → Verify → Rollback Plan → Document
```

Skip any step → you're the hero who broke production.

---

## 🔴 Stage 1 — YUM (Red Hat Legacy)

**CentOS 7, RHEL 6/7. Still everywhere. Know it cold.**

```bash
# Update package lists
yum check-update

# Update specific package
yum update httpd -y

# Update ALL packages
yum update -y
yum upgrade -y                        # same thing

# Update kernel only (reboot needed)
yum update kernel -y
yum list kernel                       # see available kernels

# List installed packages
yum list installed
yum list installed | grep httpd

# Search available packages
yum search nginx
yum info nginx

# Remove package
yum remove httpd -y
yum autoremove                        # remove unneeded deps

# History and rollback
yum history                           # see all transactions
yum history list                      # detailed list
yum history undo 5                    # undo transaction #5
yum history redo 5                    # redo transaction #5

# Clean cache (saves space)
yum clean all
yum clean packages                    # remove cached RPMs

# Enable/disable repo temporarily
yum --disablerepo=epel update         # skip EPEL
yum --enablerepo=testing update       # use testing repo

# Check security updates only
yum list-sec
```

**YUM Key Files:**
```
/etc/yum.conf                        ← main config
/etc/yum.repos.d/*.repo              ← all repos
/var/cache/yum/                      ← cached packages
/var/log/yum.log                     ← update history
```

---

## 🟢 Stage 2 — DNF (Modern Red Hat)

**RHEL 8+, CentOS Stream, Rocky/Alma Linux. YUM's faster replacement.**

```bash
# DNF = YUM on steroids. Same syntax, better performance
dnf update -y                         # update everything
dnf upgrade -y                        # same thing
dnf update httpd -y                   # specific package

# Module streams (RHEL 8+ feature)
dnf module list                       # list available modules
dnf module list nginx                 # nginx streams
dnf module enable nginx:1.18          # enable specific stream
dnf module disable nginx              # disable module
dnf module install nginx:1.18         # install entire module

# Group management
dnf group list                       # list groups
dnf group install "Web Server"        # install group
dnf group remove "Web Server"         # remove group

# History and rollback (better than YUM)
dnf history                           # transaction history
dnf history list                      # detailed list
dnf history undo last-3               # undo last 3 transactions
dnf history undo 123456               # undo specific transaction ID

# Clean and cache
dnf clean all
dnf clean packages                    # remove cached RPMs

# Repo management
dnf repolist                          # list enabled repos
dnf repolist all                      # all repos (enabled + disabled)
dnf config-manager --enable rhel-9-appstream
dnf config-manager --set-enabled epel
```

**DNF vs YUM — the table:**

| Command | YUM | DNF |
|---|---|---|
| Update all | `yum update` | `dnf update` |
| Update specific | `yum update pkg` | `dnf update pkg` |
| Search | `yum search pkg` | `dnf search pkg` |
| History | `yum history` | `dnf history` |
| Repo list | `yum repolist` | `dnf repolist` |
| Modules | ❌ No | `dnf module list` |

---

## 🟡 Stage 3 — APT (Debian/Ubuntu)

**The DEB package manager. Different syntax, same concepts.**

```bash
# Update package lists FIRST (always)
apt update                            # refresh repo metadata
apt list --upgradable                 # what's available to update

# Update packages
apt upgrade                           # safe upgrade (no new deps)
apt full-upgrade                      # aggressive upgrade (new deps OK)
apt dist-upgrade                      # same as full-upgrade

# Install/Remove
apt install nginx apache2 -y          # install multiple
apt remove nginx                      # remove (keep config)
apt purge nginx                       # remove + config
apt autoremove                        # remove unused deps
apt autoclean                         # clean old package files

# Search and info
apt search nginx
apt show nginx                        # package details
apt list --installed | grep nginx     # installed packages

# Pin packages (hold back updates)
echo "nginx hold" > /etc/apt/preferences.d/nginx-hold
apt-mark hold nginx                   # hold specific package
apt-mark unhold nginx                 # allow updates again

# History (logs, no native rollback)
cat /var/log/apt/history.log
zgrep "install nginx" /var/log/apt/history.log.*.gz

# Fix broken packages (life saver)
apt --fix-broken install
dpkg --configure -a                   # configure pending packages
apt update --fix-missing
```

**APT Key Files:**
```
/etc/apt/sources.list                 ← main repos
/etc/apt/sources.list.d/*.list        ← additional repos
/var/cache/apt/archives/              ← downloaded DEBs
/var/log/apt/                         ← history logs
```

**APT vs DNF/YUM:**

| Task | APT | DNF/YUM |
|---|---|---|
| Update lists | `apt update` | `yum check-update` |
| Update all | `apt upgrade` | `yum update` |
| Install | `apt install pkg` | `yum install pkg` |
| Remove | `apt purge pkg` | `yum remove pkg` |
| Search | `apt search pkg` | `yum search pkg` |
| Repo list | `cat /etc/apt/sources.list` | `yum repolist` |

---

## 🔵 Stage 4 — Satellite Server (Red Hat Enterprise)

**The enterprise patch manager. Centralizes RHEL patching.**

```bash
# Register client to Satellite
subscription-manager register --org="MyOrg" --activationkey="MyKey"
subscription-manager identity
subscription-manager repos --list
subscription-manager attach --auto

# Content views and lifecycle environments
hammer content-view list              # list content views
hammer lifecycle-environment list     # dev/test/prod environments

# Promote content view to environment
hammer content-view promote --id=1 --to=Testing

# Client pulls updates from Satellite
dnf update -y                         # uses Satellite repos automatically

# Check Satellite registration
subscription-manager status
rct cat-cert /etc/pki/consumer/primary.pem  # cert info
```

---

## 🟣 Stage 5 — Patch Manager Plus (ManageEngine)

**Commercial patch manager. GUI + automation.**

```
Core concept: Agent-based, works across Windows/Linux/Mac.
Dashboard → Patch approval → Deploy → Compliance reports.
```

**CLI equivalents (when GUI is down):**
```bash
# Agent status
/opt/ManageEngine/PatchManagerPlus/MSPatchManagerAgent/bin/AgentStatus.sh

# Manual scan (triggers GUI scan)
PatchManagerAgent.sh scan

# Check patch status
PatchManagerAgent.sh status
```

---

## 📊 Stage 6 — Patch Strategy (The Real Interview Question)

| Phase | What | Why |
|---|---|---|
| **Inventory** | `yum list installed`, `apt list --installed` | Know what you have |
| **Security Check** | `yum list-sec`, `apt list --upgradable` | Critical fixes first |
| **Test Environment** | Clone prod, test patches | Don't break prod |
| **Staging Rollout** | 10% of servers first | Catch issues early |
| **Full Deploy** | All remaining servers | Go time |
| **Validation** | Services up, apps working | Verify success |
| **Monitoring** | Logs, alerts for issues | Catch post-patch problems |

---

## 🚨 Stage 7 — Patch Troubleshooting

```bash
# YUM/DNF repo errors
yum clean all
yum makecache
dnf clean all
dnf makecache

# GPG key missing (RHEL repo)
rpm --import https://repo.key
dnf update --nogpgcheck               # emergency only

# Dependency hell (APT)
apt --fix-broken install
apt autoremove
apt autoclean

# Kernel panic after update
# Boot to previous kernel (GRUB menu)
# yum downgrade kernel-5.14.0-70.el9   # rollback kernel

# Service broken after patch
systemctl status httpd
journalctl -u httpd -b                # logs since boot
rpm -V httpd                          # verify package files intact

# Rollback specific package
yum history undo last-1
dnf history undo last-1

# Check what changed (RPM)
rpm -qa --last 20                     # last 20 packages installed
rpm -V httpd                          # verify files changed
```

**Patch Rollback Priority:**
```
1. yum/dnf history undo (fastest)
2. Boot previous kernel (GRUB menu)
3. yum downgrade specific-package
4. Time machine (if you have snapshots)
5. Restore from backup (nuclear option)
```

---

## 🛡️ Stage 8 — Pre-Production Patch Checklist

```bash
#!/bin/bash
# Run this BEFORE patching production

echo "=== PATCH PRE-CHECK ==="
df -hT | grep -v tmpfs
echo "=== Services ==="
systemctl list-units --type=service --state=running | wc -l
echo "=== Available updates ==="
yum check-update | wc -l              # RHEL
apt list --upgradable | wc -l         # Ubuntu
echo "=== Disk space OK ==="
df -h / /boot /var | tail -1
echo "=== Backups recent? ==="
echo "=== Reboot OK window? ==="
```

---

## 🧩 Key Files Cheatsheet

```
# Red Hat
/etc/yum.conf
/etc/yum.repos.d/*.repo
/var/log/yum.log
/var/cache/yum/

/etc/dnf/dnf.conf
/var/log/dnf.log
/var/cache/dnf/

# Ubuntu
/etc/apt/sources.list
/etc/apt/sources.list.d/
/var/log/apt/history.log
/var/cache/apt/archives/

# Satellite
/etc/rhsm/rhsm.conf
/etc/yum.repos.d/redhat.repo
```

---

## ⚡ Full Patch Decision Flow

```
New patch available?
        │
        ├─ Security critical? → Test + deploy ASAP
        │
        ├─ Kernel update? → Test + schedule reboot window
        │
        ├─ App package? → Test in staging first
        │
        └─ Minor update? → Batch with others weekly
```

---

**Zura's Patch Interview Rules:**
- They WILL ask "Kernel update broke production — what do you do?" → GRUB previous kernel → yum downgrade → investigate logs → document.
- They WILL ask "APT dependency hell — fix it" → apt --fix-broken install → autoremove → autoclean.
- They WILL ask "Difference between yum update and yum upgrade?" → Same command. Just different names.
- They WILL ask "How do you test patches?" → Clone environment → snapshot → deploy → validate services → rollback if broken.

---
*Zura's Topic 4 Patch Management Sheet — Know this or get roasted in interviews* 🔥
