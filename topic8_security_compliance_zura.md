# 🛡️ Topic 8: Security & Compliance — Zura Style

No "be secure" platitudes. Just locks that actually stop hackers.

---

## ⚡ The Security Layers — One Line

```
SELinux → Mandatory Access Control   |   firewalld → Network firewall
Compliance → CIS/STIG benchmarks     |   Hardening → No low-hanging fruit
```

Lock it down or get pwned. Simple.

---

## 🔴 Stage 1 — SELinux (Red Hat's Guard Dog)

**Not AppArmor. Not disable-this-shit. SELinux enforces policy.**

### States & Commands
```bash
# Check status
getenforce                           # Enforcing/Permissive/Disabled
sestatus                             # full details

# Change mode (temporary)
setenforce 0                         # permissive
setenforce 1                         # enforcing

# Permanent change
vim /etc/selinux/config
SELINUX=enforcing                     # reboot

# Context (labels everything)
ls -Z /var/www/html                  # file contexts
ps -eZ                               # process contexts
id -Z                                # user context

# Common fixes
restorecon -Rv /var/www/html         # relabel files
semanage fcontext -a -t httpd_exec_t '/myapp/script.sh'  # add custom context
restorecon -v /myapp/script.sh       # apply

# Audit logs (the goldmine)
ausearch -m AVC -ts recent           # recent denials
audit2allow -a -M mypolicy           # create policy from logs
semodule -i mypolicy.pp              # install policy
```

**SELinux Booleans (toggle features):**
```bash
getsebool -a | grep httpd             # httpd booleans
setsebool -P httpd_can_network_connect 1  # allow httpd outbound
semanage boolean -l                  # all booleans
```

**SELinux Modes Table:**
| Mode | Denials Logged? | Blocks Access? |
|---|---|---|
| Enforcing | ✅ Yes | ✅ Yes |
| Permissive | ✅ Yes | ❌ No |
| Disabled | ❌ No | ❌ No |

---

## 🟢 Stage 2 — firewalld (Dynamic Firewall)

**nftables backend. Zones. Services over ports.**

```bash
# Status & runtime
firewall-cmd --state                  # running?
firewall-cmd --list-all               # current zone details
firewall-cmd --get-default-zone       # active zone
firewall-cmd --get-active-zones       # all active zones

# Services (preferred over ports)
firewall-cmd --add-service=http --permanent
firewall-cmd --add-service=ssh --zone=public --permanent
firewall-cmd --add-port=8080/tcp --permanent  # fallback

# Zones
firewall-cmd --list-zones             # all zones
firewall-cmd --zone=internal --add-interface=eth1 --permanent
firewall-cmd --change-interface=eth0 --zone=public

# Rich rules (advanced)
firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" service name="ssh" accept' --permanent

# Load permanent rules
firewall-cmd --reload

# Masquerade (NAT)
firewall-cmd --add-masquerade --permanent
firewall-cmd --add-forward-port=port=8080:toaddr=192.168.1.10:port=80 --permanent

# Direct rules (nftables raw)
firewall-cmd --direct --add-rule ipv4 filter INPUT 1 -p tcp --dport 22 -j ACCEPT
```

**firewalld Zones:**
```
public     ← default, untrusted
internal   ← trusted LAN
dmz        ← semi-trusted
trusted    ← fully trusted
drop       ← drop everything
```

---

## 🟣 Stage 3 — Compliance Standards (CIS/STIG)

**CIS Benchmarks: Free hardening guides.**
**STIGs: DoD security guides (SCAP compliant).**

### CIS Linux Hardening
```
1. Install only needed packages (yum groupremove "Graphical Administration Tools")
2. Disable unused filesystems (no_vfat, no_usb_storage in kernel cmdline)
3. Enforce password policy (/etc/security/pwquality.conf)
4. SSH hardening (Protocol 2, no root, key auth)
5. AIDE (file integrity monitoring)
6. Auditd rules (/etc/audit/rules.d/)
```

### Quick CIS Audit
```bash
# Password age
chage -l root

# Sudo no password
visudo | grep NOPASSWD

# Core dumps disabled
sysctl fs.suid_dumpable  # 0

# AIDE install/check
dnf install aide -y
aide --init
mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
aide --check
```

---

## 📋 Stage 4 — Server Hardening Checklist

```
☐ SELinux: Enforcing + audit2allow fixes
☐ firewalld: SSH/HTTP only, zones configured
☐ SSH: PermitRootLogin no, keys only, Fail2Ban
☐ Passwords: No weak/default, pwquality, no reuse
☐ Services: Disable unused (systemctl disable telnet)
☐ Kernel: sysctl hardening (rp_filter=1, no icmp_redirect)
☐ Updates: Auto-security patches
☐ Logs: rsyslog/auditd → central SIEM
☐ Users: No shared accounts, sudo logging
☐ Files: 644/755 perms, no world-writable
```

**Kernel Hardening (/etc/sysctl.conf):**
```
net.ipv4.ip_forward = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.tcp_syncookies = 1
kernel.randomize_va_space = 2
vm.swappiness = 10
```

---

## 📊 Stage 5 — SELinux vs AppArmor vs firewalld

| Tool | What | Distro | Granularity |
|---|---|---|---|
| SELinux | File/process access | RHEL | Mandatory (labels) |
| AppArmor | App confinement | Ubuntu | Path-based profiles |
| firewalld | Network | All | Zones/services |

---

## 🚨 Stage 6 — Security Troubleshooting

```bash
# SELinux blocking Apache?
ausearch -m AVC -c httpd | audit2allow

# Firewall blocking?
firewall-cmd --info-service=ssh
ss -tulnp | grep 22

# SSH brute force?
fail2ban-client status sshd

# Compliance scan
# OpenSCAP: oscap xccdf eval --profile stig --results-arf results.arf /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

---

## 🧩 Key Files Cheatsheet

```
/etc/selinux/config                   # SELinux mode
/etc/selinux/config.d/                # SELinux overrides
/var/log/audit/audit.log              # SELinux denials
/etc/fstab (seclabel)                 # SELinux mounts

/etc/firewalld/firewalld.conf         # firewalld config
/etc/firewalld/zones/*.xml            # zone rules
/usr/lib/firewalld/services/*.xml     # service defs

/etc/ssh/sshd_config                  # SSH hardening
/etc/security/limits.conf             # resource limits
/etc/sysctl.conf                      # kernel tunables
/etc/pam.d/                           # PAM auth
```

---

## ⚡ Full Security Decision Flow

```
New server?
        │
        1. Base image: CIS hardened
        │
        2. SELinux: Enforcing
        │
        3. firewalld: public zone, add services only
        │
        4. SSH: Keys, no root, Fail2Ban
        │
        5. Updates: Enable repos, cron security patches
        │
        6. Audit: AIDE + auditd
        │
        └─ Test: lynis audit system
```

---

**Zura's Security Interview Rules:**
- They WILL ask "SELinux blocking httpd — fix it." → ausearch AVC → audit2allow → semodule.
- They WILL ask "firewalld vs iptables?" → firewalld is zone/service abstraction over nftables/iptables.
- They WILL ask "CIS hardening steps?" → Minimal packages, SELinux, SSH no root, auditd.
- Boss flex: Ansible hardening + Wazuh PCI DSS at FCOOS/Wipro [file:1].

---
*Zura's Topic 8 Security Sheet — Know this or get roasted in interviews* 🔥

**ALL 8 TOPICS COMPLETE. You're fucking armed. Crush tomorrow.**
