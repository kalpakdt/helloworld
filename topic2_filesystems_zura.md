# 🗄️ Topic 2: File Systems — Zura Style

No storage theory bullshit. Just what you need to not lose data and not lose the interview.

---

## ⚡ The File System Family — One Line

```
EXT4 → Default Linux workhorse
XFS  → Big files, high performance, RHEL default
ZFS  → The god-tier FS. Snapshots, checksums, pools. Solaris born, Linux adopted.
```

Know when to use which. That's the whole game.

---

## ⚡ Full Flow — How Linux Uses File Systems

```
PHYSICAL DISK
    │
    ▼
PARTITION (fdisk / parted / gdisk)
    │
    ▼
FORMAT with FS (mkfs.ext4 / mkfs.xfs / zpool create)
    │
    ▼
MOUNT to a directory (mount / /etc/fstab)
    │
    ▼
USE IT — read, write, store shit
```

That's it. Now let's tear each one apart.

---

## 🟢 Stage 1 — EXT4 (Fourth Extended Filesystem)

**The reliable old dog. Default on Ubuntu/Debian. Doesn't do fancy shit but never fucks you.**

- Max file size: **16TB**
- Max volume size: **1 Exabyte**
- Journaling: ✅ Yes (crash recovery)
- Introduced: 2008

```bash
# Create EXT4 filesystem
mkfs.ext4 /dev/sdb1
mkfs.ext4 -L "mydisk" /dev/sdb1        # with label

# Mount it
mount /dev/sdb1 /mnt/data
mount -t ext4 /dev/sdb1 /mnt/data      # explicit type

# Check filesystem info
tune2fs -l /dev/sdb1                   # detailed FS info
dumpe2fs /dev/sdb1                     # dump all superblock info
blkid /dev/sdb1                        # UUID and type

# Check and repair EXT4 (UNMOUNT FIRST — seriously)
umount /dev/sdb1
fsck.ext4 /dev/sdb1                    # check
fsck.ext4 -y /dev/sdb1                 # auto-fix everything
e2fsck -f /dev/sdb1                    # force check

# Resize EXT4
resize2fs /dev/sdb1                    # expand to full partition size
resize2fs /dev/sdb1 10G               # resize to 10G

# Disk usage
df -hT                                 # show FS type + usage
df -hT /dev/sdb1

# Persistent mount (add to /etc/fstab)
# UUID=xxxx  /mnt/data  ext4  defaults  0  2
blkid /dev/sdb1                        # get UUID first
echo "UUID=$(blkid -s UUID -o value /dev/sdb1) /mnt/data ext4 defaults 0 2" >> /etc/fstab
mount -a                               # test fstab without reboot
```

**EXT4 /etc/fstab fields explained:**
```
UUID=xxx   /mnt/data   ext4   defaults   0   2
  │           │          │        │      │   │
  device    mountpoint  type   options  dump  fsck-order
                                              (0=skip, 1=root, 2=others)
```

**When to use EXT4:**
- General purpose Linux servers
- Boot partitions
- When you want simple and stable — no drama

---

## 🔵 Stage 2 — XFS (eXtended File System)

**The speed demon. RHEL's default since RHEL 7. Handles massive files like a champ.**

- Max file size: **8 Exabytes**
- Max volume size: **8 Exabytes**
- Journaling: ✅ Yes (metadata journaling)
- Default on: **RHEL 7+, CentOS 7+**
- Can grow but **CANNOT shrink** — remember this or you're fucked

```bash
# Create XFS filesystem
mkfs.xfs /dev/sdb1
mkfs.xfs -L "xfsdisk" /dev/sdb1       # with label
mkfs.xfs -f /dev/sdb1                  # force (overwrite existing)

# Mount it
mount /dev/sdb1 /mnt/data
mount -t xfs /dev/sdb1 /mnt/data

# XFS info
xfs_info /mnt/data                     # FS details (must be mounted)
xfs_info /dev/sdb1

# Check and repair XFS
umount /dev/sdb1
xfs_repair /dev/sdb1                   # repair
xfs_repair -n /dev/sdb1               # dry run (check only, no fix)
xfs_check /dev/sdb1                    # older check tool

# Grow XFS (online — NO unmount needed!)
xfs_growfs /mnt/data                   # grow to fill partition
xfs_growfs -D 20480 /mnt/data         # grow to specific blocks

# XFS dump and restore (backup tool)
xfsdump -f /backup/data.dump /mnt/data
xfsrestore -f /backup/data.dump /mnt/restore

# Freeze/unfreeze (for consistent snapshots)
xfs_freeze -f /mnt/data               # freeze writes
xfs_freeze -u /mnt/data               # unfreeze

# Quota management
xfs_quota -x -c 'report -h' /mnt/data
```

**XFS Golden Rules:**
- ✅ Can grow online (no downtime)
- ❌ Cannot shrink — ever. Plan your sizing upfront.
- ✅ Better than EXT4 for large files and parallel I/O
- ✅ Great for databases, media storage, high-throughput workloads

---

## 🟡 Stage 3 — ZFS (Zettabyte File System)

**The god tier. Born in Solaris. Ported to Linux (OpenZFS). Does everything.**

- Not just a filesystem — it's a **volume manager + filesystem combined**
- Built-in: snapshots, checksums, compression, deduplication, RAID
- Max file size: **16 Exabytes**
- No `fsck` needed — self-healing with checksums

```bash
# Install ZFS on Linux (Ubuntu)
apt install zfsutils-linux -y

# Install ZFS on RHEL/CentOS (needs ZFS repo)
dnf install epel-release -y
dnf install zfs -y
modprobe zfs                           # load ZFS kernel module

# ── ZPOOL (storage pool) ──

# Create a simple pool
zpool create mypool /dev/sdb

# Create mirrored pool (RAID-1)
zpool create mypool mirror /dev/sdb /dev/sdc

# Create RAIDZ pool (RAID-5 equivalent)
zpool create mypool raidz /dev/sdb /dev/sdc /dev/sdd

# Create RAIDZ2 (RAID-6 equivalent — survives 2 disk failures)
zpool create mypool raidz2 /dev/sdb /dev/sdc /dev/sdd /dev/sde

# Pool status
zpool status                           # health of all pools
zpool status mypool                    # specific pool
zpool list                             # pool sizes and usage

# Pool scrub (integrity check — run monthly)
zpool scrub mypool
zpool status mypool                    # check scrub results

# ── ZFS DATASETS ──

# Create dataset (like a subdirectory with its own properties)
zfs create mypool/data
zfs create mypool/logs
zfs create mypool/backups

# List datasets
zfs list
zfs list -r mypool                     # recursive

# Set mountpoint
zfs set mountpoint=/data mypool/data

# ── COMPRESSION ──
zfs set compression=lz4 mypool/data   # enable LZ4 compression (fast + good)
zfs set compression=gzip mypool/logs  # gzip (slower, better ratio)
zfs get compression mypool/data       # check compression setting
zfs get compressratio mypool/data     # see actual compression ratio

# ── SNAPSHOTS (the killer feature) ──
# Snapshots are instant, use almost no space initially

# Take a snapshot
zfs snapshot mypool/data@snap1
zfs snapshot mypool/data@before-upgrade

# List snapshots
zfs list -t snapshot
zfs list -t snapshot mypool/data

# Rollback to snapshot (DESTROYS changes after snapshot)
zfs rollback mypool/data@snap1

# Clone a snapshot (create writable copy)
zfs clone mypool/data@snap1 mypool/data-clone

# Destroy snapshot
zfs destroy mypool/data@snap1

# Send snapshot to another server (backup/migration)
zfs send mypool/data@snap1 | ssh backup-server zfs receive backuppool/data

# Incremental send (only changes since last snapshot)
zfs send -i mypool/data@snap1 mypool/data@snap2 | ssh backup-server zfs receive backuppool/data

# ── QUOTAS AND RESERVATIONS ──
zfs set quota=100G mypool/data        # max space this dataset can use
zfs set reservation=10G mypool/data   # guaranteed space for this dataset
zfs get quota mypool/data
zfs get reservation mypool/data

# ── DEDUPLICATION (careful — RAM hungry) ──
zfs set dedup=on mypool/data          # enable dedup (needs lots of RAM)
zfs get dedupratio mypool/data        # check dedup ratio

# ── DESTROY ──
zfs destroy mypool/data               # delete dataset
zpool destroy mypool                  # nuke entire pool
```

**ZFS RAID Types — know this table:**

| ZFS Type | Equivalent | Min Disks | Drives Lost OK |
|---|---|---|---|
| `mirror` | RAID-1 | 2 | 1 |
| `raidz` | RAID-5 | 3 | 1 |
| `raidz2` | RAID-6 | 4 | 2 |
| `raidz3` | RAID-7 | 5 | 3 |
| stripe | RAID-0 | 2 | 0 (no redundancy) |

---

## 📊 Stage 4 — EXT4 vs XFS vs ZFS (Interview Gold)

| Feature | EXT4 | XFS | ZFS |
|---|---|---|---|
| Default on | Ubuntu/Debian | RHEL 7+ | Solaris / optional Linux |
| Max file size | 16 TB | 8 EB | 16 EB |
| Can shrink | ✅ Yes | ❌ No | ❌ No (pool-based) |
| Online grow | ✅ Yes | ✅ Yes | ✅ Yes |
| Snapshots | ❌ No | ❌ No | ✅ Yes (built-in) |
| Compression | ❌ No | ❌ No | ✅ Yes (built-in) |
| RAID built-in | ❌ No | ❌ No | ✅ Yes |
| Self-healing | ❌ No | ❌ No | ✅ Yes (checksums) |
| fsck needed | ✅ Yes | ✅ Yes | ❌ No |
| RAM hungry | 🟢 Light | 🟢 Light | 🔴 Heavy |
| Best for | General use | Large files/DB | Enterprise storage |

---

## 🔧 Stage 5 — Partition & Mount Workflow

```bash
# Step 1: See your disks
lsblk                                  # block device tree
fdisk -l                               # all disks and partitions
lsblk -f                               # with filesystem info

# Step 2: Partition the disk
fdisk /dev/sdb                         # interactive partitioning
# Commands inside fdisk:
# n → new partition
# p → primary
# w → write and exit
# d → delete partition
# l → list partition types

# Or use parted (for GPT / large disks)
parted /dev/sdb mklabel gpt
parted /dev/sdb mkpart primary ext4 0% 100%

# Step 3: Format
mkfs.ext4 /dev/sdb1
mkfs.xfs /dev/sdb1
# or ZFS pool creation (no mkfs needed)

# Step 4: Create mount point
mkdir -p /mnt/data

# Step 5: Mount
mount /dev/sdb1 /mnt/data

# Step 6: Verify
df -hT
mount | grep sdb1

# Step 7: Make persistent (/etc/fstab)
blkid /dev/sdb1                        # get UUID
vim /etc/fstab
# Add: UUID=xxxx /mnt/data ext4 defaults 0 2

# Step 8: Test fstab (before reboot!)
mount -a                               # if no errors, you're good
```

---

## 🚨 Stage 6 — Troubleshooting File Systems

```bash
# Disk full — what's eating space?
df -hT                                 # overall usage
du -sh /* 2>/dev/null | sort -h       # top level dirs
du -sh /var/log/* | sort -h           # find log hogs
find / -size +1G -type f 2>/dev/null  # files over 1GB

# Inode exhaustion (disk shows free space but can't create files)
df -i                                  # check inode usage
find /tmp -type f | wc -l             # count files in tmp

# Can't unmount — device busy
fuser -m /mnt/data                    # what's using it
lsof /mnt/data                        # open files on mount
umount -l /mnt/data                   # lazy unmount (last resort)
umount -f /mnt/data                   # force unmount

# Read-only filesystem (usually corruption or dirty shutdown)
dmesg | grep -i "read-only"
dmesg | grep -i "error"
mount -o remount,rw /                 # remount root as rw (emergency)

# EXT4 journal recovery
e2fsck -f /dev/sdb1                   # force check after dirty shutdown

# XFS corruption
xfs_repair /dev/sdb1                  # repair XFS
xfs_repair -L /dev/sdb1              # nuke log and repair (last resort)

# ZFS pool degraded
zpool status                          # check health
zpool scrub mypool                    # find errors
zpool replace mypool /dev/sdb /dev/sde  # replace failed disk
zpool clear mypool                    # clear error counters

# Check mount options
cat /proc/mounts                      # currently mounted FSes
mount | column -t                     # cleaner view
```

---

## 🧩 Key Files Cheatsheet

```
/etc/fstab                    ← persistent mounts (edit carefully!)
/proc/mounts                  ← currently mounted filesystems
/proc/filesystems             ← supported FS types by kernel
/sys/block/                   ← block device info
/dev/disk/by-uuid/            ← UUID to device symlinks
/dev/disk/by-label/           ← label to device symlinks

# Logs to check for FS errors
/var/log/messages             ← RHEL/CentOS
/var/log/syslog               ← Ubuntu
dmesg                         ← kernel ring buffer (real-time)
```

---

## ⚡ Full File System Decision Flow

```
Need a filesystem?
        │
        ├─ Simple Linux server, general use?
        │       └── → EXT4. Done. Stop overthinking.
        │
        ├─ RHEL server, big files, databases, high I/O?
        │       └── → XFS. It's the default for a reason.
        │
        ├─ Need snapshots, compression, built-in RAID, or self-healing?
        │       └── → ZFS. Prepare for RAM usage.
        │
        ├─ Running Solaris?
        │       └── → ZFS. No choice, it's baked in.
        │
        └─ Need to shrink the filesystem later?
                └── → Use EXT4. XFS and ZFS won't shrink.
```

---

**Zura's FS Interview Rule:**
- They WILL ask "EXT4 vs XFS — when do you use which?" → See the table above.
- They WILL ask "What makes ZFS special?" → Snapshots, checksums, built-in RAID, compression, self-healing. Fire those off fast.
- They WILL ask "XFS can't shrink — what do you do if you over-provisioned?" → Backup, destroy, recreate smaller, restore. That's the only way. Don't lie and say you can shrink it.

---
*Zura's Topic 2 File Systems Sheet — Know this or get roasted in interviews* 🔥
