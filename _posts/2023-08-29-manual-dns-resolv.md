---
layout: single
title: "Preventing NetworkManager from overwriting /etc/resolv.conf"
subtitle: ""
date: 2023-08-29 08:15:00 +0100
background: '/image/01.jpg'
tags: ['linux','network']
categories:
  - Linux
---

{% raw %}

Because ``/etc/resolv.conf`` is managed by NetworkManager service, any manual changes to the file are overwritten when the network service is restarted. To prevent this, change the value of ``dns`` parameter to ``none`` in ``/etc/NetworkManager/NetworkManager.conf``:

1. Edit /etc/resolv.conf file
2. Add ``dns=none`` under the ``[main]`` section in ``/etc/NetworkManager/NetworkManager.conf``.
3. Restart the ``NetworkManager`` service


## Reference
- [Chapter 21. Manually configuring the /etc/resolv.conf file](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/manually-configuring-the-etc-resolv-conf-file_configuring-and-managing-networking#disabling-dns-processing-in-the-networkmanager-configuration_manually-configuring-the-etc-resolv-conf-file)

{% endraw %}