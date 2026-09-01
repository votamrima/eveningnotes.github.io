---
layout: single
title: "UEFI-Based Boot Loader RHEL 8 family Systems"
subtitle: ""
date: 2026-04-03 18:15:00 +0100
background: '/image/01.jpg'
tags: ['linux','storage']
toc: true
toc_sticky: true
categories:
  - Linux
---

{% raw %}

## Quick navigation

- [Quick navigation](#quick-navigation)
- [1. UEFI basics](#1-uefi-basics)
- [2. UEFI and Secure Boot chain](#2-uefi-and-secure-boot-chain)
- [3. Important GRUB2 differences: BIOS vs UEFI](#3-important-grub2-differences-bios-vs-uefi)
- [4. Managing UEFI boot targets](#4-managing-uefi-boot-targets)
  - [View the current boot order](#view-the-current-boot-order)
  - [Delete an entry](#delete-an-entry)
  - [Select a target for the next boot only](#select-a-target-for-the-next-boot-only)
  - [Create a new entry](#create-a-new-entry)
- [5. Generating Boot Loader Specification entries](#5-generating-boot-loader-specification-entries)
- [6. Managing kernel entries with `grubby`](#6-managing-kernel-entries-with-grubby)
- [7. Repairing GRUB2 on a UEFI system](#7-repairing-grub2-on-a-uefi-system)
  - [Step 1: Confirm the ESP](#step-1-confirm-the-esp)
  - [Step 2: Reinstall the signed EFI components](#step-2-reinstall-the-signed-efi-components)
  - [Step 3: Regenerate the UEFI GRUB2 configuration](#step-3-regenerate-the-uefi-grub2-configuration)
  - [Step 4: Verify the firmware entry](#step-4-verify-the-firmware-entry)
- [Quick summary](#quick-summary)

## 1. UEFI basics

UEFI replaces the legacy BIOS boot method. It supports modern hardware, disks larger than 2 TiB, and GPT partitioning.

Unlike BIOS, which searches disks for boot code, UEFI stores registered boot targets in firmware memory (NVRAM). Each target points to an EFI application on the disk.

For GRUB2 to boot in UEFI mode, the system needs an **EFI System Partition (ESP)**:

- Formatted with a FAT file system
- Normally mounted at `/boot/efi`
- Recommended size: 512 MiB
- Usually created automatically by the Anaconda installer

## 2. UEFI and Secure Boot chain

With Secure Boot enabled, each boot component must be trusted and correctly signed.

The normal boot chain is:

1. UEFI firmware starts `shim.efi`.
2. `shim.efi`, signed with a key trusted by the firmware, loads `grubx64.efi`.
3. GRUB2 loads the selected kernel and initramfs.
4. The kernel starts `systemd`, which brings the system to the configured target.

Additional keys can be built into `shim` or stored as **Machine Owner Keys (MOK)** in NVRAM. If a required key is missing, `MokManager.efi` can be used to enroll it.

If no normal boot entry exists, UEFI can start the fallback application:

```text
/boot/efi/EFI/BOOT/BOOTX64.efi
```

It loads `fallback.efi`, which uses `/boot/efi/EFI/redhat/BOOT.CSV` to register `shim.efi` and boot the system.

## 3. Important GRUB2 differences: BIOS vs UEFI

| Component | BIOS system | UEFI system |
|---|---|---|
| Partitioning | MBR commonly used | GPT recommended |
| Boot source | Boot code in MBR | EFI application on ESP |
| GRUB2 directory | `/boot/grub2/` | `/boot/efi/EFI/redhat/` |
| GRUB configuration | `/boot/grub2/grub.cfg` | `/boot/efi/EFI/redhat/grub.cfg` |
| Configuration link | `/etc/grub2.cfg` | `/etc/grub2-efi.cfg` |
| Kernel commands | `linux16`, `initrd16` | `linuxefi`, `initrdefi` |

`grub2-mkconfig` detects the firmware type and generates the appropriate commands automatically.

> **Critical warning:** Do not use `grub2-install` to repair GRUB2 on a UEFI system. It creates a generic, unsigned `grubx64.efi`, registers the wrong boot target instead of `shim.efi`, and can prevent a Secure Boot system from starting.

## 4. Managing UEFI boot targets

The `efibootmgr` utility reads and modifies boot entries stored in UEFI NVRAM. Root privileges are required for changes.

### View the current boot order

```bash
efibootmgr
```

Important fields:

- `BootCurrent` — entry used for the current boot
- `BootOrder` — order in which entries are tried
- `Boot0003*` — an individual active entry; the number is hexadecimal

Display each entry's disk, partition, and EFI file:

```bash
efibootmgr -v
```

### Delete an entry

Delete entry `Boot0004`:

```bash
efibootmgr -b 4 -B
```

### Select a target for the next boot only

Boot from `Boot0002` once without changing the permanent order:

```bash
efibootmgr -n 2
```

### Create a new entry

Create an entry named `LAB` for an EFI application on partition 2 of `/dev/sda`:

```bash
efibootmgr -c -d /dev/sda -p 2 -L "LAB" -l '\EFI\LAB.efi'
```

The EFI application path must use backslashes.

## 5. Generating Boot Loader Specification entries

RHEL like systems can store kernel-specific Boot Loader Specification (BLS) entries in `/boot/loader/entries/`.

1. Enable BLS in `/etc/default/grub`:

   ```ini
   GRUB_ENABLE_BLSCFG=true
   ```

2. Regenerate the UEFI GRUB2 configuration:

   ```bash
   grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
   ```

3. Generate an entry for the running kernel:

   ```bash
   kernel-install add "$(uname -r)" "/lib/modules/$(uname -r)/vmlinuz"
   ```

## 6. Managing kernel entries with `grubby`

On UEFI systems, run `grubby` as root.

Show the default kernel:

```bash
grubby --default-kernel
```

Show its menu index:

```bash
grubby --default-index
```

Persistently select another default kernel:

```bash
grubby --set-default /boot/vmlinuz-<kernel-version>
```

List all kernel menu entries:

```bash
grubby --info=ALL
```

Inspect one kernel entry:

```bash
grubby --info /boot/vmlinuz-<kernel-version>
```

## 7. Repairing GRUB2 on a UEFI system

Use this sequence when EFI boot files or `grub.cfg` are damaged or missing.

### Step 1: Confirm the ESP

Verify that the EFI System Partition exists, is formatted with FAT, and is mounted at `/boot/efi`.

### Step 2: Reinstall the signed EFI components

```bash
yum reinstall grub2-efi shim
```

This restores the trusted GRUB2 and `shim` files under `/boot/efi`.

### Step 3: Regenerate the UEFI GRUB2 configuration

```bash
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
```

### Step 4: Verify the firmware entry

```bash
efibootmgr -v
```

## Quick summary

1. Verify that the machine was booted in UEFI mode.
2. Confirm that `/boot/efi` is mounted and contains the RHEL EFI files.
3. Inspect NVRAM entries with `efibootmgr -v`.
4. Reinstall `grub2-efi` and `shim` if EFI files are missing or damaged.
5. Regenerate `/boot/efi/EFI/redhat/grub.cfg`.
6. Inspect kernel entries with `grubby --info=ALL`.
7. Do **not** run `grub2-install` on a UEFI system.
8. On a UEFI system, boot repair is not about rewriting an MBR. The important pieces are the FAT-formatted ESP, signed `shim` and GRUB2 applications, the UEFI NVRAM boot entry, the UEFI-specific `grub.cfg`, and valid kernel/BLS entries.


{% endraw %}