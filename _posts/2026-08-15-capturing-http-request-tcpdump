---
layout: single
title: "Capture an Incoming HTTP Request with tcpdump"
subtitle: ""
date: 2026-08-18 18:15:00 +0100
background: '/image/01.jpg'
tags: ['linux']
toc: true
toc_sticky: true
categories:
  - Linux
---

{% raw %}

## Capture an Incoming HTTP Request with `tcpdump`

### Scenario

````text
servera  ─── HTTP request ───►  serverb
          password=...
````

`servera` sends an HTTP request periodically to `serverb`.
There may be **no web server listening on serverb**.

### 1. Discover the destination port

On `serverb`:

````bash
tcpdump -i any -nn host <servera_IP>
````

Look for something like:

````text
servera.45000 > serverb.8080: Flags [S]
````

This means `servera` tries to connect to **TCP/8080**.

### 2. Start a temporary listener

Without a listener, TCP cannot establish a connection and the HTTP request may never be sent.

````bash
ncat -lk 8080
````

If needed, allow the port:

````bash
firewall-cmd --add-port=8080/tcp
````

### 3. Capture and read the request

In another terminal:

````bash
tcpdump -i any -nn -A tcp port 8080
````

`-A` displays packet payload as readable ASCII.

Example:

````text
POST / HTTP/1.1
Host: serverb

password=Secret
````

### Home-lab simulation

On `servera`:

````bash
while true; do
    curl -d 'password=Secret' http://serverb:8080/
    sleep 60
done
````

On `serverb`:

````bash
tcpdump -i any -nn host <servera_IP>
````

Discover the port → start `ncat` → capture with `tcpdump -A`.

### Remember

````text
tcpdump
   ↓
find destination port
   ↓
ncat -lk PORT
   ↓
tcpdump -A
   ↓
read HTTP payload
````

**Key idea:** `tcpdump` shows the traffic, but TCP needs a **listening port** before the sender can normally transmit the actual HTTP request.




{% endraw %}