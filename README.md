### EmailBackup

自动打包压缩VPS文件发送到指定邮箱

1. 安装mutt msmtp
  ```
  sudo apt install mutt msmtp
  ```
2.配置msmtp
```
 vim ~/.msmtprc
 ```
 ```
defaults
  
account default
host smtp.XX.com
port 465
from XX@XX.com
auth login
tls on
tls_starttls off
user XX@XX.com
password XXXXXXXXXXXXX
logfile ~/.msmtp.log
```
3.配置mutt
```
vim ~/.muttrc
```
```
set envelope_from=yes
set from="XX@XX.com"
set realname="XXXX"
set use_from=yes
set sendmail="/usr/bin/msmtp"
```
4.创建备份脚本

新建vpsbackup文件夹,并创建vpsbackup.sh
```
mkdir vpsbackup
cd vpsbackup
vim vpsbackup.sh
```
```
#!/bin/sh
# 常规定义
MYSQL_USER="root"
MYSQL_PASS="password"
BACK_DIR="/root/vpsbackup/backup"
TO_MAIL="xx@XX.com"

# 备份网站数据目录
NGINX_DATA="/usr/local/nginx/conf"
BACKUP_DEFAULT="/home/wwwroot"

# 定义备份文件名
mysql_DATA=mysql_$(date +%Y%m%d).tar.gz
www_DEFAULT=www_$(date +%Y%m%d).tar.gz
nginx_CONFIG=nginx_$(date +%Y%m%d).tar.gz

# 判断本地备份目录，不存在则创建
if [ ! -d $BACK_DIR ] ;
  then
   /bin/mkdir -p "$BACK_DIR"
fi
 
# 进入备份目录
cd $BACK_DIR
 
# 备份所有数据库
# 导出需要备份的数据库，清除不需要备份的库
mysql -u$MYSQL_USER -p$MYSQL_PASS -B -N -e 'SHOW DATABASES' > $BACK_DIR/databases.db
sed -i '/performance_schema/d' $BACK_DIR/databases.db
sed -i '/information_schema/d' $BACK_DIR/databases.db
sed -i '/mysql/d' $BACK_DIR/databases.db
 
for db in $(cat $BACK_DIR/databases.db)
 do
   mysqldump -u$MYSQL_USER -p$MYSQL_PASS ${db} | gzip -9 - > $BACK_DIR/${db}.sql.gz
done
rm -rf databases.db

# 打包数据库
 tar -zcvf $BACK_DIR/$mysql_DATA *.sql.gz --remove-files >/dev/null 2>&1 
 
# 打包本地网站数据
 tar -zcvf $BACK_DIR/$www_DEFAULT $BACKUP_DEFAULT >/dev/null 2>&1 
 
# 打包Nginx配置文件
 tar -zcvf $BACK_DIR/$nginx_CONFIG $NGINX_DATA >/dev/null 2>&1 
 
# 发送邮件
echo "$(date +%Y-%m-%d)-backup-files!!!" | /usr/bin/mutt -s "$(date +%Y-%m-%d)-backup" -a $BACK_DIR/$nginx_CONFIG  $BACK_DIR/$mysql_DATA  $BACK_DIR/$www_DEFAULT -- $TO_MAIL

# 删除所有文件
#rm -rf $BACK_DIR

# 删除3天以前的文件
find *.gz  -mtime +3 -print|xargs rm -f;
 
exit 0
```
5.创建计划任务

创建一个每周五中午运行一次的计划任务
```
crontab -e
```
```
* 12 * * 5 /root/vpsbackup/vpsbackup.sh
```
**enjoy it!**

