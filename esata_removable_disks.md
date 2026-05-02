# eSATA / USB Removable Disk Procedures on GS/DGS Machines

Stability: C3 - Structural / stable

**Source:** https://wiki.anl.gov/gsdaq/Handeling_removable_disks_under_ESATA  
**Last Updated:** 2026-04-25

---

## Table of Contents

- [Overview](#overview)
- [Mounting a Disk](#mounting-a-disk)
- [Unmounting a Disk](#unmounting-a-disk)
- [Formatting a New Disk](#formatting-a-new-disk)
- [Sudo Configuration](#sudo-configuration-for-system-managers)
- [Cross-References](#cross-references)

---

## Overview

Standard procedures for mounting, unmounting, formatting, and labeling removable disks connected via eSATA or USB on Gammasphere/DGS Linux machines (Scientific Linux, Fedora).

---

## Mounting a Disk

1. Power up the disk, then check `dmesg` to see the assigned device name (e.g., `/dev/sdd`). There may be a brief delay before recognition.
2. Alternatively, use `lsblk` if available on the system.
3. Check if the partition has a label:
   ```
   sudo e2label /dev/sdd1
   ```
4. Mount using the label:
   ```
   mkdir ~/esata/`sudo e2label /dev/sdd1`
   sudo /bin/mount /dev/sdd1 ~/esata/`sudo e2label /dev/sdd1`
   lsblk
   df | grep `sudo e2label /dev/sdd1`
   ```

---

## Unmounting a Disk

```
sync;sync;sync
sudo /bin/umount /dev/sdd1
```

**If unmount fails:** Someone may have `cd`'d into the disk in a shell. Find and kill the offending process:

```
lsof | grep <label>
# Example output: bash 8367 dgs cwd DIR ... /media/120514a/user/cs
kill -9 8367
umount /media/<label>
```

---

## Formatting a New Disk

> ⚠️ **Warning:** Verify the device name before formatting. Formatting the wrong disk is destructive and irreversible.

1. Identify the device with `dmesg` or `lsblk` (assume `/dev/sdd`).
2. Partition with `fdisk`:
   ```
   sudo /sbin/fdisk /dev/sdd
   # At fdisk prompt: p, n, p, 1, <Enter>, <Enter>, w
   ```
   This creates `/dev/sdd1`. ✅ verified 2026-04-30 - wiki.anl.gov/gsdaq/Handeling_removable_disks_under_ESATA
3. Format with EXT4:
   ```
   sudo /sbin/mkfs.ext4 -m 0 /dev/sdd1
   ```
   The `-m 0` flag removes the 5% root-reserved space — acceptable for data disks, **not** for system disks. ✅ verified 2026-04-30 - wiki.anl.gov/gsdaq/Handeling_removable_disks_under_ESATA
   - Note: `mkfs.ext4` is slow on Scientific Linux, fast on Fedora 15+. ✅ verified 2026-04-30 - wiki.anl.gov/gsdaq/Handeling_removable_disks_under_ESATA

4. Label the partition (use a unique, date-based label):
   ```
   sudo /sbin/e2label /dev/sdd1 20120611a
   ```
   Auto-mounted USB disks will appear as `/media/20120611a`. ✅ verified 2026-04-30 - wiki.anl.gov/gsdaq/Handeling_removable_disks_under_ESATA

5. Mount and set permissions for non-root writes:
   ```
   sudo /bin/mount /dev/sdd1 ~/data1
   sudo mkdir ~/data1/user
   sudo chmod a+rwx ~/data1/user
   ```

6. Unmount when done:
   ```
   sudo /bin/umount /dev/sdd1
   ```

---

## Sudo Configuration (for System Managers)

Add to `/etc/sudoers` via `visudo`:

```
Cmnd_Alias ESATAFORMAT = /sbin/e2label, /sbin/fdisk, /sbin/mkfs.ext4, /bin/mkdir, /bin/chmod
gamuser ALL=ESATAFORMAT
dgs     ALL=ESATAFORMAT

Cmnd_Alias ESATAMOUNT = /bin/mount, /bin/umount
gamuser ALL=ESATAMOUNT
dgs     ALL=ESATAMOUNT
```

This allows `gamuser` and `dgs` accounts to mount/format removable disks without full root access.

---

## Cross-References

| File | Relationship |
|------|--------------|
| [nfs_layout.md](nfs_layout.md) | NFS storage volumes used alongside removable disks |
| [run_procedures.md](run_procedures.md) | Experiment run procedures including data storage |
| [overview_DGS.md](overview_DGS.md) | DGS machine overview (GS/DGS Linux hosts that use this) |
| [expMemory_2008_Chiara.md](expMemory_2008_Chiara.md) | Active experiment log — data storage locations and disk usage |

---

*Created: 2026-04-25 | Last reviewed: 2026-04-25*
