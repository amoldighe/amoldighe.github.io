---
layout: post
title:  "Consul Alerts"
date:   2017-07-01
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

# Emails  

* Download the latest consul-alerts binary from - https://github.com/AcalephStorage/consul-alerts and copy to /usr/local/bin/

* Next we will be utilizing the consul key-value store for storing the parameters used by [consul-alerts](https://github.com/AcalephStorage/consul-alerts) 

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


# Twilio

To integrate Twilio alerts with Consul, a Twilio account is required. Twilio provide an api endpoint url for your account which can be used to make calls to any international phone number. In our case we have the consul monitor script to place a Twilio call by using this shell script - [twilio.sh](https://github.com/amoldighe/consul-twilio/blob/master/consul-twilio.sh) 

For the Twilio script to work, the following setup is required on the Consul. 

* Add the below keys end with '/' indicating folder 

```
consul kv put consul-alerts/config/notifiers/twilio/
consul kv put consul-alerts/config/notifiers/twilio/call-history/
```

* Add the below key - value on any consul node 

We will be using our web proxy to communicate with Twilio from our Private Cloud setup.

```
consul kv put consul-alerts/config/notifiers/twilio/from-number  "+11111111111"

consul kv put consul-alerts/config/notifiers/twilio/proxy-url "http://proxy-server-ip:port"

consul kv put consul-alerts/config/notifiers/twilio/user-key "user-key-provided-by-twilio"

consul kv put consul-alerts/config/notifiers/twilio/voice-xml "http://demo.twilio.com/docs/voice.xml"

consul kv put consul-alerts/config/notifiers/twilio/xpost-url "https://api.twilio.com/2010-04-01/Accounts/twilio-account-key/Calls.json"

consul kv put consul-alerts/config/notifiers/twilio/devops-numbers
```

The value for devops-numbers can be added from Consul dashboard dashboard using the key-value store in below format of one value per line and update the key.

```
e.g.

9999999999
9999999998
9999999997
9999999996
9999999995
9999999994
9999999993
9999999992
9999999991
```
* Place the new twilio script in /etc/consul.d/script/twilio-call/twilio.sh

Use /bin/bash to invoke the twilio script from any of the alert script.

The twilio.sh script also has logic to populate the call history details to Consul dashboard key-value section.


# Slack

Slack integration with Consul helped us send Consul alerts to a Slack channel which is being monitored by L1 team. 
The integration setup is simple as consul-alerts supports slack integration using the key values mentioned on github for [consul-alerts](https://github.com/AcalephStorage/consul-alerts)

* Login to slack account and follow the steps to create a incomming webhook for Consul on Slack channel under Custom Integrations.

* Slack will provide a incomming webhook URL which can be used to send the alert.

* Using web proxy for consul-alerts

As consul alert is running inside a Private CLoud server, we will be using our web proxy to communicate with slack server to send an alert to the slack channel. This is accomplished using a shell script /usr/local/bin/consul-alerts-process.sh to start consul-alerts. 

```
#!/bin/bash
export https_proxy='http://<proxy ip>:<proxy port>'
nohup /usr/local/bin/consul-alerts start --consul-dc=poc-dc --watch-checks --watch-events --log-level=debug
```

This script uses export hhtps_proxy to set the proxy before starting consul-alerts. This will help consul-alerts hit the slack webhook url which will be set as a notifier key-vlaue in the below commands on any one of the Consul server.

1. To create slack notifiers
```
consul kv put consul-alerts/config/notifiers/slack/
```
2. Enable slack notifier
```
consul kv put consul-alerts/config/notifiers/slack/enabled "true"
```
3. Set Consul cluster name
```
consul kv put consul-alerts/config/notifiers/slack/cluster-name "cloud-poc" 
```
4. Set Slack incoming webhook url
```
consul kv put consul-alerts/config/notifiers/slack/url "https://hooks.slack.com/services/slack/incomming/web-hook-hash-url"
```
5. Set slack channel name
```
consul kv put consul-alerts/config/notifiers/slack/channel "#cloud-poc-alerts"
```
6. Set username (empty here, as no username needed)
```
consul kv put consul-alerts/config/notifiers/slack/username " "
```
7. Set icon url (default icon will be used)
```
consul kv put consul-alerts/config/notifiers/slack/icon-url
```
8. Set default icon as icon-emoji
```
consul kv put consul-alerts/config/notifiers/slack/icon-emoji
```

Watch the slack channel for alert !!

