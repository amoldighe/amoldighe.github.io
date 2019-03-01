---
layout: post
title:  "Jenkins - Deploying App"
date:   2018-8-17
tags:
  - Jenkins
  - nodejs
  - SCM
  - github
  - deploy-keys
  - build
  - deploy
---

Jenkins is the most widely used build-n-deploy tool used in Devops arena. In past, I have used Jenkins for infrastructure code deployments, where we were deploying compute nodes in openstack infrastructure. If I have to sum up the high level build & deploy stages, we were getting the openstack packages from one of our local repo, 
saltstack is called to build the configuration file for the compute nodes, the packages are deployed with the necessary configuration and the compute node is rebooted. 
Sounds simple, but it's actualy quite complex considering the jenkins configuration and various build scripts playing their part for the deployment. 
Recently while working with the application team, I came across a deployment requirement which was quite intresting.

## Requirement

* Setup a repository sync for a server running node.js code.

* The node js code should run inside forever.

* The forever session should run under a screen session as the terminal output of the screen session is required by the developers.

## Solution

We will be using our Jenkins server for syncing the code from github repository to the node js server which ia a VPS instance on AWS.

* Prepeare the SCM  

First we need to prepare the Jenkins server to get the code from github using the SCM (Source Code Management) option.

Add the repository under SCM along with an existing or new credential key. 

Add the public key of the keypair as deploy key to repository. [Click here to know more about deploy key](https://developer.github.com/v3/guides/managing-deploy-keys/#deploy-keys)


* Run job on Jenkins master

While creating the Jenkins job, we need to specify that, it runs on Master, by selecting "Restrict where this project can be run" under General option and select the "Label Expression" as "master" 

* Configure the SCM 

Under Source Code Management, add the git repository URL and select the key as credential. Jenkins will immediatly try to talk to the git repository and check if it can connect.

We are going to use 'Poll SCM' and select the Schedule to sync and build the repo every hour 

```
H * * * *
```

Under "Build Environment" we will be using Binding option to assign a variable (SSHKEY) to our credential key file. This is the credential key file which will be used to talk to the node js server instance. The credential is added using Jenkins control panel using "Jenkins >> Credential >> Add credentials"

* Next is the Build step, where we will be executing few shell commands :

```
rsync -avhpzi -e "ssh -i ${SSHKEY}" --delete --exclude ".git" --exclude "logs" --exclude ".session" ./ "user@node-vps-instance:/home/ubuntu/ReportScript/"

ssh -i "${SSHKEY}" user@node-vps-instance "screen -X -S reportscript quit"

ssh -i "${SSHKEY}" user@node-vps-instance "screen -S reportscript -d -m bash -c 'cd /home/user/nodedata; sudo NODE_ENV=prod forever bin/www'"
```

## Execution

Lets see what happens when we execute the Jenkins job

* Jenkins master is going to connect to the git repository and fetch the changes on the master branch

* The changes will be synced to the workspace on jenkins master

* Next the shell commands under build will be executed by Jenkins

rsync will use the SSHKEY and copy changes in code from the workspace directory on Jenkins master to the mentioned path on node js server

second shell command is to kill if there is any existing screen session on the node js server

thrid shell command will set the node js environment start a forever session to execute the node js code under a screen session. 

Please note the screen session will be named and started in detached mode.













