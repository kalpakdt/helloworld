# 💾 Topic 7: Backup & Recovery — Zura Style

No "backups are important" bullshit. Just tools that actually restore when shit hits the fan.

---

## ⚡ The Backup Rule — One Line

```
3-2-1: 3 copies, 2 media types, 1 offsite. Anything less = gambling.
```

Veeam/NetBackup/Bacula all follow it. Pick one, master it.

---

## 🔴 Stage 1 — Veeam (VM King)

**VMware/Hyper-V snapshot magic. Agentless for VMs.**

```bash
# Veeam Agent for Linux (bare metal)
# Install
wget https://repo.veeam.com/veeamsnap/rpm/veeamsnap-latest.noarch.rpm
rpm -i veeamsnap-latest.noarch.rpm

# Create snapshot
veeamsnap create --path /snapshots/snap1.snap

# Backup job (CLI)
veeamconfig job start --id JOB_ID
veeamconfig session list                # list running sessions
veeamconfig job list                    # list jobs

# Restore
veeamconfig filesystem restore --id SESSION_ID --path /data --target /restore

# Common paths
/var/log/veeamsnap/veeamsnap.log       # logs
/etc/veeamsnap.conf                    # config
```

**Veeam Workflow:**
```
1. Proxy server snapshots VM
2. Backup proxy copies changed blocks
3. Dedup/Compress to repo
4. Replicate to offsite
5. Instant VM recovery (from proxy)
```

**Veeam Ports:**
```
9392 TCP  ← Backup service
9401 TCP  ← Proxy
6162 TCP  ← Replication
```

---

## 🟢 Stage 2 — NetBackup (Symantec/Veritas Enterprise)

**Big iron. Tape/library support. Policy-based.**

```bash
# Client commands (bpclntcmd)
bpclntcmd --server masterserver --policy POLICY_NAME  # register client
bplist -s -l -R / -pi POLICY_NAME -pt FULL              # list backups
bpplinfo POLICY_NAME -U                                 # policy info

# Backup
bpbackup -p POLICY_NAME -s FULL /data                    # manual backup

# Restore (interactive)
bprestore -D 7 -pi POLICY_NAME -pt FULL                  # restore last 7 days
bprestore                                                # GUI mode

# Inventory (tape/library)
vmchange -resv r1 -h robot1 -m TAPE001                  # reserve tape
vxlogview -p nb 51216                                    # logs (process ID)

# Common paths (UNIX client)
/usr/openv/netbackup/              ← install dir
/usr/openv/var/log/                ← logs
/etc/init.d/vxlogd                 ← log daemon
```

**NetBackup Policy Types:**
```
Standard      ← Files/directories
VMware        ← VM snapshots
Oracle        ← DB hot backups
MS-Windows    ← NT/2008/2012
NDMP          ← NAS appliances
```

---

## 🟡 Stage 3 — Bacula (Open Source Beast)

**Director + Storage Daemon + File Daemon. Config-file driven.**

```
Components:
- Director (bacula-dir)  ← brain
- Storage Daemon (bacula-sd) ← tape/disk writer
- File Daemon (bacula-fd) ← client agent
```

```bash
# Console commands (bconsole)
bconsole
*status director                     # all jobs
*status client fdname                # client status
*.backup                             # start backup job
*restore where=/tmp/restore select all done  # restore everything

# List backups
*list jobs
*list files jobid=123
*estimate job=FullBackup client=fdname fileset=FullSet

# Reload config
*bscan                               # scan storage for new volumes
*reload

# Config files
/etc/bacula/bacula-dir.conf          ← director
/etc/bacula/bacula-sd.conf           ← storage
/etc/bacula/bacula-fd.conf           ← client
```

**Bacula Job Types:**
```
Backup    ← Full/Inc/Diff
Restore   ← everything
Verify    ← check integrity
Copy      ← duplicate to secondary storage
Migrate   ← tape-to-disk or vice versa
```

---

## 📊 Stage 4 — Tool Comparison (Interview Gold)

| Feature | Veeam | NetBackup | Bacula |
|---|---|---|---|
| VM Snapshots | ✅ Native | ✅ VMware | ❌ Manual |
| Tape Support | ✅ | ✅ Best | ✅ |
| Agentless VMs | ✅ | Partial | ❌ |
| GUI | Excellent | Good | Web (bweb) |
| Cost | Paid | $$$ | Free |
| Scale | Medium | Enterprise | Medium |
| Learning Curve | Easy | Steep | Medium |

---

## 🚨 Stage 5 — Restore Troubleshooting (The Real Test)

```
Backup succeeded but restore failed?
        │
        ├─ Permissions? → Run as root/service account
        │
        ├─ Space? → Check target disk
        │
        ├─ Catalog mismatch? → bscan (Bacula), inventory (NetBackup)
        │
        ├─ Tape mounted? → vmchange
        │
        └─ Logs! → /var/log/veeamsnap/, vxlogview, bacula-dir.log
```

**Common Restore Commands:**
```bash
# Veeam: veeamconfig filesystem restore --target /restore
# NetBackup: bprestore -R /tmp/restore_script
# Bacula: *restore select all done (in bconsole)
```

---

## 🛡️ Stage 6 — 3-2-1 Best Practices

```
3 COPIES: Primary + Local replica + Offsite
2 MEDIA: Disk + Tape/Cloud
1 OFFSITE: S3, Glacier, Iron Mountain

Test restores MONTHLY. Backups without tested restores = toilet paper.
RPO: 1 hour (how much data loss OK)
RTO: 4 hours (how long downtime OK)
```

**Boss Self-Host: Restic to S3 — CLI gold [file:1]**
```bash
restic backup /data --repo s3:s3.amazonaws.com/bucket
restic snapshots
restic restore latest --target /restore
```

---

## 🧩 Key Files Cheatsheet

```
# Veeam
/etc/veeamsnap.conf
/var/log/veeamsnap/

/usr/openv/netbackup/bp.conf         # NetBackup client config
/usr/openv/var/log/                  # NetBackup logs

# Bacula
/etc/bacula/*.conf
/var/spool/bacula/                   # working dir
/var/lib/bacula/                     # catalog DB
```

---

## ⚡ Full Backup Decision Flow

```
Backup need?
        │
        ├─ VMs? → Veeam
        │
        ├─ Enterprise tapes? → NetBackup
        │
        ├─ Open source budget? → Bacula
        │
        └─ Files only? → Restic/rsync to S3
```

---

**Zura's Backup Interview Rules:**
- They WILL ask "Backup succeeded, restore failed — why?" → Permissions, catalog out of sync, no space.
- They WILL ask "3-2-1 rule?" → 3 copies, 2 media, 1 offsite. Test restores.
- They WILL ask "RPO vs RTO?" → Recovery Point (data loss), Recovery Time (downtime).
- Drop your Restic S3 automation — interviewers eat that up [file:1].

---
*Zura's Topic 7 Backup Sheet — Know this or get roasted in interviews* 🔥
