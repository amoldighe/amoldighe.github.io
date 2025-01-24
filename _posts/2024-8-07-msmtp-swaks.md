---
layout: post
title:  "Msmtp & Swaks"
date:   2024-8-07
tags:
  - msmtp
  - swaks
  - linux
  - command
  - ubuntu
  - SMTP
  - apt-get
---

I have come across regular use case for sending report or alerts emails for developer applications, platform services or just testing sending emails from a linux box.
One of the qiuckest way is to use an SMTP client & SMTP server which take care of email delivery.

### MSMTP 

Download & Install msmtp - https://marlam.de/msmtp/download/
It is also available as a package in Debian repository, please check for your distro.

```
apt-get install msmtp msmtp-mta mailutils
```

Configure msmtp with credentials from your SMTP server / account for email delivery e.g. a gmail SMTP or AWS SES.

```
root@my-prod-5:~# cat /etc/msmtprc
account default
auth on
tls on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile /var/log/mail.log
host     smtp.gmail.com
port     587
from     @amoldighe.com
user     YOUR_SMTP_USERNAME
password YOUR_SMTP_PASSWORD
```
root@my-prod-5:~# echo "Hello, World!" | mail -s "Hello Test" recipient@amoldighe.com

### Swaks (Swiss Army Knife for SMTP)

Recently I came across this wonderful command line tool for forwarding emails via SMTP - SWAKS http://www.jetmore.org/john/code/swaks/
It's caled the Swiss Army Knife for SMTP as it is a single command line package that handles SMTP configuration + sending of email.


```
apt-get install swaks
```

Post install, it is scriptable, can be used for testing and debugging SMTP servers and sending emails. 

```
swaks \
        --from sender@amoldighe.com \
        --to recipient@amoldighe.com \
        --server smtp.gmail.com \
        --port 587 \
        --auth plain \
        --tls \
        --auth-user 'YOUR_SMTP_USERNAME' \
        --auth-password 'YOUR_SMTP_PASSWORD' \
        --header 'Subject: Test email' \
        --body "This is a test email."
```

