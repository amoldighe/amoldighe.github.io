---
layout: post
title:  "Gitlab - postfix setup issues"
date:   2016-09-14
tags:
  - gitlab
  - postfix
---

This is my effort to continue to record the issues I came across while setting up Gitlab for my Devops team members. Please note the Gitlab setup I have is on a VM on my Private Cloud environment and the mail server I am using is a physical box in one of our datacenters. 


* **Issue: Error while starting postfix**

```
gitlab postfix/sendmail[899]: fatal: file /etc/postfix/main.cf: parameter default_privs: unknown user name value: nobody
```
 
**Solution :**

Verify the hostname settings in /etc/postfix/main.cf

> cat /etc/postfix/main.cf | grep hostname
> myhostname = gitlab
 
Check if user nobody is present in /etc/passwd
> ubuntu@gitlab:~$ sudo less /etc/passwd | grep nobody

If not present add the below line to /etc/passwd
```
nobody:x:99:99:Nobody:/:/sbin/nologin
```

Apply the configuration for postfix, execute below commands: 
> newaliases
>
> apt-get install –f
>
> postfix upgrade-configuration
>
> dpkg-reconfigure postfix

 
* **Issue: Error while starting postfix**

```
Jul 14 17:12:03 gitlab postfix/trivial-rewrite[16639]: fatal: bad boolean configuration: append_dot_mydomain = gitlab.localhost
```

**Solution :**
 
Verify the hostname settings in /etc/postfix/main.cf
> vi /etc/postfix/main.cf

change the value for append_dot_mydomain to “no”
> cat /etc/postfix/main.cf | grep append_dot_mydomain
append_dot_mydomain = no


* **Issue: Error while sending mail using postfix**

```
Jul 14 17:17:14 gitlab postfix/smtp[17151]: 7262B206F1: to=<amol.dighe@xxxxx.com>, relay=10.130.0.55[10.130.0.55]:25, delay=39, delays=39/0.01/0.01/0.03, dsn=4.1.8, status=deferred (host 10.130.0.55[10.130.0.55] said: 450 4.1.8 <ubuntu@gitlab.localhost>: Sender address rejected: Domain not found (in reply to RCPT TO command))
```

**Solution :**

Check if the below parameters are proper in /etc/postfix/main.cf as per system hostname, network details, host alias

```
smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated
smtp_generic_maps = hash:/etc/postfix/generic
myhostname = gitlab
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
mydestination = localdomain, localhost, localhost.localdomain, gitlab.localhost
relayhost = 10.130.0.55
mynetworks = 127.0.0.0/8 10.160.170.70/24
mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = all
inet_protocols = all
```
 
As we are using a relay host or another mailhost to send our mails, its IP address need to be set in ‘relayhost’ parameter.

Error indicates that relay is reachable but sender address is rejected by relay, the sender email address used is ubuntu@gitlab.localhost

This is a case where the host is not in a qualified domain and the user email id configured on the host is not a valid email id. A work around is to send mail from a valid email id which the mail server can verify and proceed further with sending the mail.

Open your main.cf file
> vi /etc/postfix/main.cf

Append following parameter
>smtp_generic_maps = hash:/etc/postfix/generic

Add a entry for ubuntu@gitlab.localhost mapping it against a vaild email id e.g. no-reply@xxxxx.com
> vi /etc/postfix/generic
> ubuntu@gitlab.localhost  no-reply@xxxxx.com

Save and close the file. Create or update generic postfix table:
> postmap /etc/postfix/generic

Restart postfix
> /etc/init.d/postfix restart

