---
title: MySQL定时备份
date: 2024-04-29 10:03
tags:
  - MySQL
categories:
  - 学习
---

这里采用 crontab 和 mysqldump 结合的方式对数据库进行备份：

首先编写一个bash脚本
```sh
bash
#!/bin/bash
DATE=$(date +"%Y%m%d")
TIME=$(date +"%H%M%S")
DB_USER="root"
DB_PASS="123456"
DB_NAME="aurora"
BACKUP_DIR="/home/cmz/mysql/backup/aurora"

mkdir -p $BACKUP_DIR/$DATE

# 由于我的MySQL部署在docker，所以需要加上docker exec 容器名，如果不是则不需要
docker exec mysql mysqldump -u$DB_USER -p$DB_PASS $DB_NAME > $BACKUP_DIR/$DATE/db_$TIME.sql

find $BACKUP_DIR/* -mtime +7 -exec rm -rf {} \;
```
这个脚本将会在指定目录下创建以日期为名称的件夹，并且在该文件夹内创建以时间为名称的数据库备份文件，最后删除7天前的备份。

接下来，需要使用crontab配置一个定时执行备份脚本的任务。打开终端并输入以下命令以编辑crontab文件：

crontab -e
然后，在文件底部添加以下行：

0 0 * * * /bin/bash /home/cmz/mysql/backup/aurora/aurora_backup.sh >/dev/null 2>&1 &
这将在每天午夜执行backup.sh脚本。

保存并退出
保存并关闭crontab文件，现在您已经设置好了自动按天备份MySQL的任务。其中，备份文件将储存在指定目录下的日期文件夹内，根据您的需要可以更改参数，例如日期格式、数据库用户名和密码、备份路径等等。
