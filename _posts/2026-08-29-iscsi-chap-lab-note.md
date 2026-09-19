---
layout: single
title: "iSCSI + CHAP — Quick Lab Note"
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

## 1. Create new target on `servera`

Install and start:

```bash
dnf install -y targetcli
systemctl enable --now target
```

Create backing file:

```bash
mkdir -p /root/iscsi
truncate -s 1G /root/iscsi/test0.img
```

Open `targetcli`:

```bash
targetcli
```

Create backstore:

```text
/backstores/fileio create test0 /root/iscsi/test0.img
```

Create target:

```text
/iscsi create iqn.2026-09.home.lab:servera
```

Create LUN:

```text
/iscsi/iqn.2026-09.home.lab:servera/tpg1/luns create /backstores/fileio/test0
```

---

## 2. Create new client on `serverb`

Install tools:

```bash
dnf install -y iscsi-initiator-utils
```

Check client IQN:

```bash
cat /etc/iscsi/initiatorname.iscsi
```

Example:

```text
InitiatorName=iqn.2026-09.lab.serverb:client
```

Back on `servera`:

```bash
targetcli
```

Create ACL:

```text
/iscsi/iqn.2026-09.home.lab:servera/tpg1/acls create iqn.2026-09.lab.serverb:client
```

Enable CHAP:

```text
cd /iscsi/iqn.2026-09.home.lab:servera/tpg1
set attribute authentication=1
set auth userid=chapuser
set auth password=chap123456
```

Set same CHAP on ACL:

```text
cd acls/iqn.2026-09.lab.serverb:client
set attribute authentication=1
set auth userid=chapuser
set auth password=chap123456
```

Save:

```text
saveconfig
exit
```

Open firewall:

```bash
firewall-cmd --permanent --add-service=iscsi-target
firewall-cmd --reload
```

---

## 3. Configure / reset client

On `serverb`, set CHAP in:

```bash
vi /etc/iscsi/iscsid.conf
```

Use:

```ini
node.session.auth.authmethod = CHAP
node.session.auth.username = chapuser
node.session.auth.password = chap123456
```

Restart:

```bash
systemctl restart iscsid
```

Reset old saved iSCSI configuration:

```bash
iscsiadm -m node --logout
iscsiadm -m node -o delete
```

Check:

```bash
iscsiadm -m node
```

Should be empty.

---

## 4. Discover and login

Discover target:

```bash
iscsiadm -m discovery -t sendtargets -p <servera_IP>
```

Example:

```bash
iscsiadm -m discovery -t sendtargets -p 192.168.56.30
```

Login:

```bash
iscsiadm -m node \
-T iqn.2026-09.home.lab:servera \
-p 192.168.56.30:3260 \
--login
```

Verify:

```bash
iscsiadm -m session
lsblk
```

## Remember

```text
servera:
backstore → target → LUN → ACL → CHAP

serverb:
CHAP config → reset old node → discovery → login → lsblk
```

If login fails with:

```text
authorization failure
```

check the saved client node settings:

```bash
iscsiadm -m node -T <TARGET_IQN> -p <IP>:3260
```



{% endraw %}