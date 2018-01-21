---
layout: post
title:  "Mollygaurd - Prevent accidental reboot"
date:   2016-08-15
tags:
  - mollygaurd
  - shutdown
  - reboot
---

How do u prevent any accidental restarts of your server ?

Install molly-guard it’s a simple tool that is executed before a user runs poweroff/reboot command. It needs to exit successfully before the actual command is invoked. Installation is simple, on a ubuntu host: 

> apt-get install mollygaurd

It’s a simple tool with a  lot of use to prevent acidential restart by script or a sudo user.
The way it works is by prompting the user to enter the name of the host that needs to be powered off OR rebooted. On successfully entering the hostname, poweroff/reboot command is executed.

