---
layout: single
title: "Repairing LVM Issues by Restoring Metadata"
subtitle: ""
date: 2026-03-03 18:15:00 +0100
background: '/image/01.jpg'
tags: ['linux','storage']
toc: true
toc_sticky: true
categories:
  - Linux
---

{% raw %}



## Purpose

LVM automatically saves Volume Group metadata. These archives can undo an incorrect LVM change, such as accidentally shrinking a logical volume.

The common mistake is:

````bash
lvresize -L 20M /dev/vg_lab/lv_data
````

This sets the LV to exactly 20 MiB. To add 20 MiB, use:

````bash
lvresize -L +20M /dev/vg_lab/lv_data
````

---

## LVM Structure

````text
Physical Device → PV → VG → LV → Filesystem → Mount Point
````

Example:

````text
/dev/sdb → vg_lab → lv_data → XFS → /mnt/lvm-demo
````

Common LV paths:

````bash
/dev/vg_lab/lv_data
/dev/mapper/vg_lab-lv_data
````

---

## Metadata Backups and Archives

Historical metadata is stored in:

````bash
/etc/lvm/archive/
````

The archive usually represents the VG state **before** a change.

Current metadata is stored in:

````bash
/etc/lvm/backup/
````

Use the archive created immediately before the bad command.

List available versions:

````bash
vgcfgrestore -l vg_lab
````

Check the `Description` field to identify the correct archive.

---

## Diagnose the Problem

Check the mount and storage layout:

````bash
grep /mnt/lvm-demo /etc/fstab
findmnt /mnt/lvm-demo
lsblk -f
pvs
vgs
lvs
````

If mounting fails after an accidental LV reduction, inspect the metadata history before making further changes.

---

## Restore Previous Metadata

Unmount the filesystem:

````bash
umount /mnt/lvm-demo
````

Restore the selected archive:

````bash
vgcfgrestore -f /etc/lvm/archive/vg_lab_00002-123456789.vg vg_lab
````

`vgcfgrestore` restores VG and LV metadata, not deleted files or filesystem contents.

Reactivate the LV:

````bash
lvchange -an /dev/vg_lab/lv_data
lvchange -ay /dev/vg_lab/lv_data
````

Then verify and mount:

````bash
lvs
lsblk
mount /mnt/lvm-demo
findmnt /mnt/lvm-demo
df -h /mnt/lvm-demo
````

---

## Correctly Extend an LV and Filesystem

To add space to an XFS filesystem:

````bash
lvextend -r -L +20M /dev/vg_lab/lv_data
````

The `-r` option grows the filesystem with the LV.

Without `-r`:

````bash
lvextend -L +20M /dev/vg_lab/lv_data
xfs_growfs /mnt/lvm-demo
````

---

## Critical Syntax Difference

````bash
-L 20M
````

sets the total size to 20 MiB.

````bash
-L +20M
````

adds 20 MiB to the current size.

Example:

````text
40 MiB + -L 20M  → 20 MiB
40 MiB + -L +20M → 60 MiB
````

---

## XFS Warning

XFS can grow but normally cannot shrink.

Safe:

````bash
lvextend -r -L +1G /dev/vg_lab/lv_data
````

Dangerous:

````bash
lvreduce ...
````

Reducing an LV beneath an existing XFS filesystem can make it unusable.

---

## Essential Commands

````bash
vgcfgrestore -l <VG>
vgcfgrestore -f <archive> <VG>
vgcfgbackup <VG>

pvs
vgs
lvs

lvchange -an /dev/<VG>/<LV>
lvchange -ay /dev/<VG>/<LV>
````

---

## Recovery Workflow

````text
Check mount and LVM status
        ↓
Review metadata history
        ↓
Select the archive before the bad command
        ↓
Unmount and deactivate the LV
        ↓
Restore metadata with vgcfgrestore
        ↓
Reactivate, mount, and verify
````

Key points:

````text
/etc/lvm/archive/ = historical metadata
/etc/lvm/backup/  = current metadata

-L 20M  = set size
-L +20M = add space
````

> `vgcfgrestore` restores LVM metadata, not filesystem files. Recovery works only if the underlying filesystem data has not been overwritten.

# References:
- [configuring_and_managing_logical_volumes](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html-single/configuring_and_managing_logical_volumes/index)


{% endraw %}