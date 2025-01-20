---
layout: post
title:  "Using Find & Sed"
date:   2024-7-30
tags:
  - find
  - sed
  - linux
  - command
  - ubuntu
  - files
---

### Find files/directories older than +days and check their size 

```
find . -mtime +7 | awk '{print "du -shc " $1}'
```

### Find files/directories older than +days and delete them

```
  find . -mtime +60 -delete

  find . -type f -mtime +15  -print0 | xargs -0 rm
```
### Find files/directories older than +days and list them 

```
find . -type f -mtime +60  -print0 | xargs -0 ls 
```

#### Find file more than specific size

```
find /var/log  -type f -size +100M
```

#### Find file resursively
```
find . -name <filename>
```

### md5sum of all files in a directory

```
find . -type f -exec md5sum {} + | LC_ALL=C sort | md5sum
```

### Find files in a directory and perform  processing on them and write to log file

```
cd in dir :\> find . -name '*.gz' -exec sh -c 'gzip -t -v {}' ';'  &>> /logfile
cd in dir :\> find . -name '*.gz' | xargs -n1 gzip -t -v &>> /logfile
```

### Find character & remove it along with trailing characters

```
root@ns207-ny2:/var/log# less syslog | awk '{print $7}' |  cut -f1 -d"#" | grep 10. |  sort |uniq
10.136.29.178
10.136.3.141
10.2.101.102
10.2.101.105
```

### search for file modified in 3 days & rsync them to specified directories & create directories if missing
  
```  
  cd /mnt/n1/Volumes/jocker/wc2/grid/data/public/partners
  find neustar -type f -mtime -3 | xargs -I% rsync -avph -R % /mnt/qnap/Volumes/jocker/wc2/grid/data/public/partners
```

### Using Parallel command with find

```

find /mnt/backups/n1/mmap/ -maxdepth 1 -mindepth 1 -type d | parallel -n1 -j10 --eta --retry 
--joblog /var/log/mmap_backup_jobs/$RANDOM 'aws s3 --exact-timestamps sync s3://new-app-backup/chat/mmap/{/}/ {}/'

```

### Remove trailing character using sed

```
cat /tmp/chat_pixel_logs-archive-chat_cookies-BAD.log | awk '{print "du -sh "$2}' | sed 's/.$//'

cat chat_pixel_logs-archive-chat_ua-GOOD.log | awk '{print $1}' | grep -v gzip | sed 's/.$//' >> chat_pixel_logs-archive-chat_ua-GOODgzFiles.log
```

### Search for a pattern in a file & comment the line

```
sed -i /nfs/s/^/#/ /etc/fstab
```

### Remove trailing character using sed

```
cat /tmp/pixel_logs-archive-dmp_cookies-BAD.log | awk '{print "du -sh "$2}' | sed 's/.$//'

cat pixel_logs-archive-dmp_ua-GOOD.log | awk '{print $1}' | grep -v gzip | sed 's/.$//' >> pixel_logs-archive-dmp_ua-GOODgzFiles.log
```

### Split each string from a file using "/" pattern and print the specific words and remove last 3 character in the string

```
cat 4TStorage_allBadList_gzip_files.log | awk '{split($0,a,"/"); print a[4]"/"a[5] }' | sed 's/...$//'
```

### List files and rename them by removing last character from each of the file

```
#file: 
root@mysql1-n1:/mnt/storage170TB/backups/mysql/06/mysql1-ny.eqx.io/sprocs# ls
06_mysql1-ny.eqx.io_backup01-ny2.21_routines.sql.gz}
06_mysql1-ny.eqx.io_backup01-ny2.21analytics_joker_dmp.sql.gz}

for i in `ls`; do mv $i $(ls $i | sed 's/.$//') ; done

#result:

root@mysql1-ny2-a1:/mnt/storage170TB/backups/mysql/06/mysql1-ny.eqx.io/sprocs# ls
06_mysql1-ny.eqx.io_backup01-ny2.21_routines.sql.gz
06_mysql1-ny.eqx.io_backup01-ny2.21analytics_joker_dmp.sql.gz
```