---
layout: single
title: "Tracking Unexpected Permission Changes"
subtitle: ""
date: 2026-09-01 18:15:00 +0100
background: '/image/01.jpg'
tags: ['linux']
toc: true
toc_sticky: true
categories:
  - Linux
---

{% raw %}

## Tracking Unexpected Permission Changes

This lab simulates a directory whose permissions are changed regularly to random values. The goal is to identify what causes the change.

### 1. Simulation

Create the directory:

```bash
mkdir -p /lab/permtest
chmod 755 /lab/permtest
```

Create a script that applies a random permission:

```bash
cat > /usr/local/bin/change-tmp-perms.sh <<'EOF'
#!/bin/bash
PERMS=(700 711 733 750 755 770 777)
chmod "${PERMS[$RANDOM % ${#PERMS[@]}]}" /lab/permtest
EOF

chmod +x /usr/local/bin/change-tmp-perms.sh
```

Create a systemd service:

```bash
cat > /etc/systemd/system/change-permission.service <<'EOF'
[Unit]
Description=Change permissions of spool tmp

[Service]
Type=oneshot
ExecStart=/usr/local/bin/change-tmp-perms.sh
EOF
```

Create a timer:

```bash
cat > /etc/systemd/system/change-permission.timer <<'EOF'
[Unit]
Description=Run permission changer regularly

[Timer]
OnBootSec=20
OnUnitActiveSec=30

[Install]
WantedBy=timers.target
EOF
```

Activate it:

```bash
systemctl daemon-reload
systemctl enable --now change-permission.timer
```

Watch the permissions:

```bash
watch -n 2 'stat -c "%a %n" /lab/permtest'
```

You should see values changing, for example:

```text
755 /lab/permtest
700 /lab/permtest
777 /lab/permtest
733 /lab/permtest
```

---

## 2. Solve the Problem

### Fast method: `journalctl`

Check recent events from the current boot:

```bash
journalctl -r -b
```

You may see:

```text
Started Change permissions of spool tmp.
change-permission.service: Succeeded.
```

Now inspect the service:

```bash
systemctl status change-permission.service
systemctl cat change-permission.service
```

You find:

```text
ExecStart=/usr/local/bin/change-tmp-perms.sh
```

Inspect the script:

```bash
cat /usr/local/bin/change-tmp-perms.sh
```

Root cause found.

---

### Deeper method: `auditctl`

Watch attribute changes:

```bash
auditctl -w /lab/permtest -p a -k tmp_perm
```

Wait for the next permission change.

Then:

```bash
ausearch -k tmp_perm -i
```

Look for:

```text
comm="chmod"
exe="/usr/bin/chmod"
PROCTITLE="chmod ..."
pid=
ppid=
```

This shows exactly which process changed the directory.

Remove the temporary rule:

```bash
auditctl -W /lab/permtest -k tmp_perm
```

---

## 3. Stop the Cause

```bash
systemctl disable --now change-permission.timer
chmod 755 /lab/permtest
```

### Troubleshooting Flow

```text
permissions change regularly
        ↓
journalctl -r -b
        ↓
systemctl status / cat
        ↓
inspect script

If unclear:
        ↓
auditctl -w PATH -p a
        ↓
ausearch -k KEY -i
        ↓
find process / command / parent
```

**Use `journalctl` for a quick clue. Use `auditctl` when you need exact evidence of what changed the permissions.**


{% endraw %}