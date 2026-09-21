---
layout: single
title: "LUKS: Set a New Password Without Knowing the Old One"
subtitle: ""
date: 2026-08-19 18:15:00 +0100
background: '/image/01.jpg'
tags: ['linux']
toc: true
toc_sticky: true
categories:
  - Linux
---

{% raw %}

### Scenario

You have:

```text
encrypted.img
luks.key
```

The old password is unknown.
`luks.key` contains the LUKS master key.

### Lab preparation

Create a small encrypted image:

```bash
truncate -s 100M encrypted.img
cryptsetup luksFormat encrypted.img
```

Use a temporary password:

```text
OldPass123
```

Save the master key:

```bash
cryptsetup luksDump \
  --dump-master-key \
  --master-key-file luks.key \
  encrypted.img
```

Enter:

```text
OldPass123
```

Now simulate the  situation:

```text
encrypted.img   # encrypted file
luks.key        # master key
old password    # unknown
```

### Set a new password

```bash
cryptsetup luksAddKey \
  --master-key-file luks.key \
  encrypted.img
```

Set a new password:

```text
NewPass123
```

### Test

```bash
cryptsetup open encrypted.img test
```

Enter:

```text
NewPass123
```

Check:

```bash
lsblk
```

Close:

```bash
cryptsetup close test
```

### Useful check

```bash
cryptsetup luksDump encrypted.img
```

You should see an additional active keyslot.

### Remember

```text
create LUKS image
      ↓
save master key
      ↓
forget old password
      ↓
luksAddKey --master-key-file
      ↓
set new password
      ↓
test with cryptsetup open
```

**Key idea:** the old password is not recovered. The master key is used to create a new valid LUKS password.




{% endraw %}