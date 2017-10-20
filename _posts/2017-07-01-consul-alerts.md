---
layout: post
title:  "Consul for Monitoring"
date:   2017-06-25
tags:
  - consul
  - monitoring
  - ubuntu 14.04
  - key value store
  - serf 
  - raft
  - consul alert
  - mail
  - twilio
  - slack
---

Consul is an amazing tool for Service Discovery, which we have further leveraged to monitor these services and send us alearts incase these services are affected using consul-alerts. For our production private cloud setup, we wanted consul to send alreats via multiple channels like - emails, twilio & slack. Lets look at their setup one by one.

* **Emails** 

* Download the latest consul-alerts binary from - https://github.com/AcalephStorage/consul-alerts and copy to /usr/local/bin/

* Next we will be utilizing the consul key-value store for storing the parameters used by consul-alerts binary  

```
(poc)amol@consul-02:~$ consul kv put consul-alerts/config/notifiers/emai
Success! Data written to: consul-alerts/config/notifiers/email

(poc)amol@consul-02:~$ consul kv put consul-alerts/config/notifiers/emailed true
Success! Data written to: consul-alerts/config/notifiers/email/enabled

(poc)amol@consul-02:~$ consul kv put consul-alerts/config/notifiers/email/cluster-name poc
Success! Data written to: consul-alerts/config/notifiers/email/cluster-name

(poc)amol@consul-02:~$ consul kv put consul-alerts/config/notifiers/email/one-per-alert true
Success! Data written to: consul-alerts/config/notifiers/email/one-per-alert

(poc)amol@consul-02:~$ consul kv put consul-alerts/config/notifiers/email/one-per-node true
Success! Data written to: consul-alerts/config/notifiers/email/one-per-node

(poc)amol@consul-02:~$ consul kv put consul-alerts/config/notifiers/email/port 25
Success! Data written to: consul-alerts/config/notifiers/email/port

(poc)amol@consul-02:~$ consul kv put consul-alerts/config/notifiers/email/receivers amoldighe@mailserver.com
Success! Data written to: consul-alerts/config/notifiers/email/receivers

(poc)amol@consul-02:~$ consul kv put consul-alerts/config/notifiers/email/sender-alias "Consul Alert"
Success! Data written to: consul-alerts/config/notifiers/email/sender-alias

(poc)amol@consul-02:~$ consul kv put consul-alerts/config/notifiers/email/sender-email consul.poc@ril.com
Success! Data written to: consul-alerts/config/notifiers/email/sender-email

(poc)amol@consul-02:~$ consul kv put consul-alerts/config/notifiers/email/url <mention the mail server IP>
Success! Data written to: consul-alerts/config/notifiers/email/url

(poc)amol@consul-02:~$ consul kv put consul-alerts/config/checks/enabled true
Success! Data written to: consul-alerts/config/checks/enabled

```

* Starting consul-alerts 

Start the consul-alerts binary on one of the consul server nodes.


```
(poc)root@consul-03:~# consul-alerts start --consul-dc=poc-dc --watch-checks --watch-events --log-level=debug
INFO[0000] Checking consul agent connection...
INFO[0004] Consul Alerts daemon started
INFO[0004] Consul Alerts Host: consul-03
INFO[0004] Consul Agent: localhost:8500
INFO[0004] Consul Datacenter: jse-2
INFO[0004] Started Consul-Alerts API
INFO[0004] Running for leader election...
INFO[0004] Now acting as leader.
INFO[0004] Now watching for events.
INFO[0004] Now watching for health changes.
INFO[0005] Running health check.
```

Now the consul-alerts process should watch the script execution for all the service discovery scripts and should send mail if any script reach warning or critical state. consul-alerts logs to /var/log/syslog

Due to past experiance with consul-alerts process dying on us, we started running consul-alerts using supervisord.

* Configure supervisord for consul-alert node

We are using the default configuration for Supervisord. If we look at the configuration file /etc/supervisord/supervisord.conf, it indicates that any files found in /etc/supervisor/conf.d and ending in .conf will be included. This is where we can add configurations for consul-alerts, which tells Supervisord how to run and monitor consul-alerts. 

Create a file /etc/supervisor/conf.d/consul-alerts.conf for configuring the consul-alerts with below parameters :

```
[program:consul_alerts]
command=/usr/local/bin/consul-alerts-process.sh  //Create a consul-alerts
process_name=%(program_name)s
numprocs=1
numprocs_start=0
autostart=true
autorestart=true
startsecs=10
startretries=1000000
exitcodes=0,2
stopsignal=TERM
stopwaitsecs=60
redirect_stderr=true
stdout_logfile=/var/log/consul-alerts
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=10
stdout_capture_maxbytes=0
stdout_events_enabled=false
serverurl=AUTO
```

* consul-alerts-process.sh

Create a script  /usr/local/bin/consul-alerts-process.sh and provide execution permission. This will help us start consul-alerts with the required options.

```
#!/bin/bash
export https_proxy='http://<proxy ip>:<proxy port>'
nohup /usr/local/bin/consul-alerts start --consul-dc=poc-dc --watch-checks --watch-events --log-level=debug
```

Please note, I am using proxy server IP for reaching slack.com, this will be explained when we configure slack.

* Start supervisord to read configuration setup for consul-alerts

```
supervisorctl reread
supervisorctl update
```

This should start consul-alert under supervisord

```
15970 ?        Ss     0:01 /usr/bin/python /usr/bin/supervisord -c /etc/supervisor/supervisord.conf
21670 ?        S      0:00  \_ /bin/bash /usr/local/bin/consul-alerts-process.sh
21671 ?        Sl     2:01      \_ consul-alerts start --consul-dc=poc-dc --watch-checks --watch-events --log-level=debug
21680 ?        Sl     0:12          \_ consul watch -http-addr localhost:8500 -datacenter poc-dc -type checks consul-alerts watch checks --alert-addr localhost:9000 --log-level debug
21681 ?        Sl     0:00          \_ consul watch -http-addr localhost:8500 -datacenter poc-dc -type event consul-alerts watch event --alert-addr localhost:9000 --log-level debug
```


* **Twilio**


* **Slack**
