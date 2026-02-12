---
layout: single
title: "Disabling beep in bash"
subtitle: ""
date: 2024-04-01 18:15:00 +0100
background: '/image/01.jpg'
tags: ['linux','bash']
---

{% raw %}


To disable the beep in bash you need to uncomment (or add if not already there) the line ``set bell-style none`` in your ``/etc/inputrc file``. This file is protected and should be edited using ``sudo``  or ``root`` user.

Source: 
- [Stackoverflow](https://stackoverflow.com/questions/36724209/disable-beep-in-wsl-terminal-on-windows-10)

{% endraw %}