Two weeks back, I migrated from an older version of Gitlab - v7.1.1 to v8.9.6. We already had all our code residing on the older version of gitlab which needed to be migrated to the new gitlab server. This should have been a fairly simple task considering gitlab v8.9.6 has an import feature which allows user to import a project from various sources. This feature was not goingt to help as our old & new gitlab servers were in two different clouds which were seperated on different physical network. Let make a case : 

Problem : Migrating gitlab projects from a gitlab server across diffrenet cloud.

Challenge : Migrate gitlab project along with all the project commits.   
  
Solution : 

* Take backup of all projects on old gitlab server on to any development host.
git clone --mirror git@oldgitlab.server:me/Consul.git

This will create a clone of old gitlab repo 'Consul' - with files, commits i.e. it creates a bare repository.

* Copy the Consul.git directory to the new gitlab server under the following directory path - /var/opt/gitlab/git-data/repositories/root/

root@gitlab:/home/ubuntu# ls -lh /var/opt/gitlab/git-data/repositories/root/
total 20K
drwxrwx--- 7 git  git  4.0K Jul 14 13:17 Consul.git
drwxr-xr-x 7 git  git  4.0K Jul 19 11:35 gittest.git

* Execute gitlab-rake command to import repository
This should create and register a project repository on the new gitlab server. 

root@gitlab:/home/ubuntu# gitlab-rake gitlab:import:repos
Processing root/Consul.git
* Consul (root/Consul.git) exists
Processing root/gittest.git
Instance method "run" is already defined in Object, use generic helper instead or set StateMachines::Machine.ignore_method_conflicts = true.
* Failed trying to create gittest (root/gittest.git)
   Errors: {:base=>["Failed to create repository via gitlab-shell"]}


* Incase an error is encountered in gitlab-rake command 

We need to set the user and file permission on the project repository directory being added. 
 
root@gitlab:/var/opt/gitlab/git-data/repositories/root# chown git.git -R gittest.git

root@gitlab:/var/opt/gitlab/git-data/repositories/root# ls -lh
total 20K
drwxrwx--- 7 git git 4.0K Jul 14 13:17 Consul.git
drwxr-xr-x 7 git git 4.0K Jul 19 11:35 gittest.git

Fix the directory and file permission

root@gitlab:/var/opt/gitlab/git-data/repositories/root# chmod o-xr -R gittest.git
root@gitlab:/var/opt/gitlab/git-data/repositories/root# ls -lh
total 20K
drwxrwx--- 7 git git 4.0K Jul 14 13:17 Consul.git
drwxrwx--- 7 git git 4.0K Jul 19 11:35 gittest.git

There is also a possibilty that the hooks directory in a repository that can have a symlink pointing to another directory.
Remove this symlink.

root@gitlab:/var/opt/gitlab/git-data/repositories/root# gitlab-rake gitlab:import:repos

Processing root/Consul.git
* Consul (root/Consul.git) exists
Processing root/gittest.git
* Created gittest (root/gittest.git)

Done!

Congratulations you have migrated gitlab repository.
