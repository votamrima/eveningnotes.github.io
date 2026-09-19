---
layout: single
title: "SOS Reports – Focused Diagnostic Collection"
subtitle: ""
date: 2026-08-29 18:15:00 +0100
background: '/image/01.jpg'
tags: ['linux']
toc: true
toc_sticky: true
categories:
  - Linux
---

{% raw %}

`sos report` collects system configuration and diagnostic data into an archive, normally under `/var/tmp`. ([Red Hat Documentation][1])

## 1. Generate a normal SOS report

Install if needed:

````bash
dnf install -y sos
````

Create report:

````bash
sos report
````

Non-interactive:

````bash
sos report --batch
````

Find the result:

````bash
ls -lh /var/tmp/sosreport*
````

---

## 2. Generate only specific information

List available plugins:

````bash
sos report --list-plugins
````

Run only the filesystem plugin:

````bash
sos report --batch --only-plugins filesys
````

Useful short form:

````bash
sos report --batch -o filesys
````

Plugin-specific options are enabled with `-k plugin.option`. ([GitExtract][2])

For detailed **ext2/ext3/ext4** information:

````bash
sos report --batch \
  -o filesys \
  -k filesys.dumpe2fs
````

`dumpe2fs` reads ext2/ext3/ext4 filesystem metadata. ([man7.org][3])

Before collecting, identify the filesystems:

````bash
findmnt
````

Only ext4:

````bash
findmnt -t ext4
````

Example:

````text
/       /dev/mapper/rhel-root ext4
/boot   /dev/sda2            ext4
/data   /dev/sdb1            ext4
````

---

## 3. Exclude a partition such as `/boot`

First find the device behind `/boot`:

````bash
findmnt -no SOURCE /boot
````

Example:

````text
/dev/sda2
````

Generate filesystem information but skip `dumpe2fs` for that device:

````bash
sos report --batch \
  -o filesys \
  -k filesys.dumpe2fs \
  --skip-commands='*dumpe2fs*/dev/sda2*'
````

`--skip-commands` supports shell-style wildcard matching. ([Debian Manpages][4])

A reusable version:

````bash
BOOTDEV=$(findmnt -no SOURCE /boot)

sos report --batch \
  -o filesys \
  -k filesys.dumpe2fs \
  --skip-commands="*dumpe2fs*${BOOTDEV}*"
````

### Key idea

````text
Full system:
sos report

Specific area:
sos report -o <plugin>

Extra plugin data:
-k plugin.option

Exclude something:
--skip-commands='pattern'
````

For filesystem troubleshooting:

````text
findmnt
   ↓
identify devices
   ↓
-o filesys
   ↓
-k filesys.dumpe2fs
   ↓
skip unwanted devices
````

[1]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/getting_the_most_from_your_support_experience/generating-an-sos-report-for-technical-support_getting-the-most-from-your-support-experience?utm_source=chatgpt.com "Chapter 1. Generating an sos report for technical support | Getting the most from your Support experience | Red Hat Enterprise Linux | 9 | Red Hat Documentation"
[2]: https://gitextract.com/sosreport/sos?utm_source=chatgpt.com "Full Code of sosreport/sos for AI - Complete Repository Source | GitExtract"
[3]: https://man7.org/linux/man-pages/man8/dumpe2fs.8.html?utm_source=chatgpt.com "dumpe2fs(8) - Linux manual page"
[4]: https://manpages.debian.org/unstable/sos/sos-report.1.en.html?utm_source=chatgpt.com "sos-report(1) — sos — Debian unstable — Debian Manpages"


{% endraw %}