---
layout: single
title: "Boot Loader on BIOS-Based RHEL 8 Systems"
subtitle: ""
date: 2026-04-03 18:15:00 +0100
background: '/image/01.jpg'
tags: ['linux','storage']
categories:
  - Linux
---

{% raw %}

## Quick navigation

- [Quick navigation](#quick-navigation)
- [1. The main idea](#1-the-main-idea)
- [2. BIOS, MBR, and GRUB2](#2-bios-mbr-and-grub2)
- [3. The RHEL 8 boot sequence](#3-the-rhel-8-boot-sequence)
- [4. Important GRUB2 files](#4-important-grub2-files)
- [5. Changing the GRUB2 configuration](#5-changing-the-grub2-configuration)
- [6. Managing BLS boot entries](#6-managing-bls-boot-entries)
- [7. Using `grubby`](#7-using-grubby)
- [8. Reinstalling GRUB2 in rescue mode](#8-reinstalling-grub2-in-rescue-mode)
- [9. Early kdump](#9-early-kdump)
- [10. Short summary](#10-short-summary)

## 1. The main idea

On a BIOS-based RHEL 8 system, the boot chain is:

**BIOS → MBR → GRUB2 → kernel + initramfs → systemd → default target**

Each component starts the next one. Therefore, the location of a failure helps identify what must be repaired.

## 2. BIOS, MBR, and GRUB2

When the machine starts, BIOS performs a **Power-On Self-Test (POST)**, initializes critical hardware, and looks for a bootable disk containing a **Master Boot Record (MBR)**. BIOS loads the first part of GRUB2 from the disk and gives it control.

The MBR is only **512 bytes**:

- It contains the disk partition table.
- Only about **446 bytes** remain for boot-loader code.
- Additional GRUB2 modules are therefore stored in the **MBR gap**, between the MBR and the first partition.

The MBR gap should be at least **31 KiB**. RHEL installations created by Anaconda normally start the first partition at sector 2048, leaving a **1 MiB gap**, which is sufficient for GRUB2.

GRUB2 can store its first stage either in the disk's MBR or in the first sector of a partition marked as bootable.

## 3. The RHEL 8 boot sequence

1. BIOS runs POST and initializes critical hardware.
2. BIOS searches the configured disks for a bootable MBR.
3. BIOS finds GRUB2 and passes control to it.
4. GRUB2 reads `/boot/grub2/grub.cfg` and displays the boot menu.
5. GRUB2 loads the selected kernel and its `initramfs` into memory.
6. GRUB2 passes control, kernel arguments, and the `initramfs` location to the kernel.
7. The kernel uses drivers from `initramfs`, initializes critical hardware, and starts `/sbin/init` as PID 1. On RHEL 8, `/sbin/init` links to `systemd`.
8. The initial `systemd` starts `initrd.target` and mounts the real root file system at `/sysroot`.
9. The kernel switches from the temporary `initramfs` root to the real root file system. `systemd` then re-executes itself from the real system.
10. `systemd` activates the default target and resolves unit dependencies.
11. The system reaches its intended state and presents a text or graphical login.

The `initramfs` is a small temporary file system containing the drivers and kernel modules needed to find and mount the real root file system.

## 4. Important GRUB2 files

| Path | Purpose |
|---|---|
| `/boot/` | Stores kernels and initial RAM file systems. |
| `/boot/grub2/` | Stores BIOS GRUB2 configuration and extension modules. |
| `/boot/grub2/grub.cfg` | Generated main configuration file for a BIOS system. |
| `/etc/grub2.cfg` | Symbolic link to the BIOS `grub.cfg`. |
| `/etc/grub.d/` | Contains helper scripts used to generate `grub.cfg`. |
| `/etc/default/grub` | Contains variables used during configuration generation. |
| `/boot/grub2/grubenv` | A 1 KiB environment file containing values such as the saved or default entry. |
| `/boot/efi/EFI/redhat/grub.cfg` | GRUB2 configuration path on a UEFI system. |

Normally, do **not** edit `grub.cfg` directly. It is generated automatically and may be replaced after a kernel or configuration update.

## 5. Changing the GRUB2 configuration

Common variables in `/etc/default/grub` include:

- `GRUB_TIMEOUT` — seconds before GRUB2 starts the default entry automatically.
- `GRUB_DEFAULT` — default menu entry; entry numbering starts at `0`.
- `GRUB_CMDLINE_LINUX` — kernel arguments added to every Linux boot entry.

After changing `/etc/default/grub`, regenerate the BIOS configuration:

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```

This command combines the variables in `/etc/default/grub` with the scripts in `/etc/grub.d/` and writes a new `grub.cfg`.

## 6. Managing BLS boot entries

The **Boot Loader Specification (BLS)** stores each boot entry in a separate drop-in file. This allows kernels and boot-menu entries to be added or removed without directly rewriting the main GRUB2 configuration.

To enable BLS configuration, add this setting to `/etc/default/grub`:

```ini
GRUB_ENABLE_BLSCFG=true
```

Regenerate the BIOS GRUB2 configuration:

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```

Create the BLS entry for the currently running kernel:

```bash
kernel-install add "$(uname -r)" "/lib/modules/$(uname -r)/vmlinuz"
```

The resulting boot-entry files are stored under `/boot/loader/entries/`.

## 7. Using `grubby`

`grubby` is a convenient wrapper for managing BLS and kernel boot entries.

```bash
# Show the default kernel
grubby --default-kernel

# Show the index of the default entry
grubby --default-index

# List all kernel menu entries
grubby --info=ALL

# Display one specific entry
grubby --info /boot/vmlinuz-<kernel-version>

# Make a kernel the persistent default
grubby --set-default /boot/vmlinuz-<kernel-version>
```

Use `grubby` when you need to inspect entries, select the default kernel, or persistently add or remove kernel arguments.

## 8. Reinstalling GRUB2 in rescue mode

Reinstall GRUB2 when the **MBR or MBR gap is damaged**. If the installed system cannot boot, use a RHEL installation source or PXE rescue environment.

1. Boot from the installation source.
2. Select **Troubleshooting → Rescue a Red Hat Enterprise Linux system**, or append `inst.rescue` to the boot command line.
3. Select the option that mounts the installed system under `/mnt/sysroot`.
4. Switch into the installed system:

   ```bash
   chroot /mnt/sysroot
   ```

5. Reinstall GRUB2 on the boot disk:

   ```bash
   /sbin/grub2-install /dev/vda
   ```

6. Reboot and verify that GRUB2 loads correctly.

> `/dev/vda` is only an example. Always identify the actual boot disk before running `grub2-install`; specify the whole disk, not a partition.

## 9. Early kdump

Traditional kdump might start too late to capture a crash that occurs during the early boot process. RHEL 8 provides **early kdump** for this situation.

Early kdump places the crash kernel and its `initramfs` inside the normal booting kernel's `initramfs`. During early boot, they are loaded into the memory reserved with `crashkernel`, making it possible to capture an early kernel crash dump.

The required early-kdump `dracut` modules are provided by the `kexec-tools` package.

## 10. Short summary

| Situation | Correct action or location |
|---|---|
| Change global GRUB2 settings | Edit `/etc/default/grub`. |
| Apply changed settings on BIOS | Run `grub2-mkconfig -o /boot/grub2/grub.cfg`. |
| Inspect kernel boot entries | Use `grubby --info=ALL`. |
| Change the default kernel | Use `grubby --set-default ...`. |
| Manage separate boot entries | Use BLS files under `/boot/loader/entries/`. |
| Repair a damaged MBR or MBR gap | Enter rescue mode, `chroot /mnt/sysroot`, and run `grub2-install` on the boot disk. |
| Investigate a crash during early boot | Use early kdump support. |

The safest mental model is simple: **edit the source configuration, regenerate the generated file, and reinstall GRUB2 only when its disk-level boot code is damaged.**


{% endraw %}