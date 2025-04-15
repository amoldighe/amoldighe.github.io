---
layout: post
title:  "Xserver for Python"
date:   2025-02-01
tags:
  - VNC
  - xvfb
  - X11
  - XVNC
  - xclock
  - xmessage
  - x11vnc
  - virtual display
---


One recent usecase of a python developer was to run a GUI application on a cloud server. The application was displaying stock market metrics from multiple sources on this python dashboard. 
Our application servers were running Ubuntu 22.04 server edition, we employed the below approch to run a quick X server test for this requirement. 

* Run a headless X server & connect using VNC

The X server virtual framebuffer (Xvfb) allows creating non-visible display windows without the computation burden of a full X11 graphics environment. 
Xvfb is effective for most Linux systems without displays.

```
apt install xvfb
```

* Start the Xvfb server

```
Xvfb :99 -ac  -screen 0 1280x720x24&
export DISPLAY=:99
```

The above command starts Xvfb on display :99 with a screen resolution of 1024x768 and 24-bit color depth. 
Next command sets the DISPLAY environment variable by exporting the DISPLAY environment variable to point to the virtual display. 
In this case, set it to :99 since we started Xvfb on display :99

* Install XVNC server 

```
apt install x11vnc
x11vnc -shared -forever -display :99&

```

The purpose of XNVC is to have a virtual display and a VNC server which is accessible remotely.
XNVC server connects to xvfb display on :99 & a VNC server is started on port 5900 

* Test using X11 apps

```
apt install x11-apps
xclock &
xmessage -center "Hello World" &
```

Here I have installed xclock & also sending a message "Hello World" on virtual display for test.

* You can connect to the display on the application server on port 5900 using a VNC application from your local machine.

* Both XVFB & XVNC servers can be added to system start up by writing a systemd service.
