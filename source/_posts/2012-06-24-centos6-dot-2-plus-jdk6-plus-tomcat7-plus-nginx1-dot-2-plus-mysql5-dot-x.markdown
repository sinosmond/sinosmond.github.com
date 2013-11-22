---
layout: post
title: "CentOS6.2 + JDK6 + Tomcat7 + Nginx1.2 + MySQL5.X"
date: 2012-06-24 22:36
comments: true
categories: [CentOS, JDK, Tomcat, Nginx, MySQL]
toc: true
---

## 说明

- **本指南不涉及多机集群。**
  * 是一个单机 Web 应用解决方案
  * 也可用于 Java Web 应用软件的测试环境
  * 笔者将其用在基于 DRBD 存储的 KVM 虚拟机里
- **本指南的尽量使用 YUM 仓库安装所需软件。**
  * 方便日后更新
  * 方便日后使用 Puppet 进行管理 

## 安装 CentOS 6.2

### 分区设置

```bash
# df -hT
文件系统    类型      容量  已用  可用 已用%% 挂载点
/dev/mapper/vg0-lv_root
              ext4     50G  3.2G   44G   7% /
tmpfs        tmpfs    3.9G     0  3.9G   0% /dev/shm
/dev/mapper/vg0-lv_backup
              ext4     42G  176M   40G   1% /backup
/dev/sda1     ext4    485M   51M  409M  12% /boot
/dev/mapper/vg0-lv_mysql
              ext4     98G  188M   93G   1% /var/lib/mysql
```

**提示：在 vg0 卷组中应保留一定的剩余空间，以便使用 LVM 快照对 mysql 进行备份。**

### 选择安装组件

选择 Minimal 安装 CentOS 6.2。

## 配置使用官方仓库的国内镜像站点

### 方法1：

```bash
# cd /etc/yum.repos.d/
# cp CentOS-Base.repo{,.orig}
# sed -i "s/mirror.centos.org/mirrors.163.com/g" CentOS-Base.repo
# sed -i "s/^mirrorlist/#mirrorlist/g" CentOS-Base.repo
# sed -i "s/^#baseurl/baseurl/g" CentOS-Base.repo
```

### 方法2：

参考 [163镜像的说明文档](http://mirrors.163.com/.help/centos.html) 下载 REPO 文件。

```bash
# cd /etc/yum.repos.d/
# mv CentOS-Base.repo{,.orig}
# wget -O CentOS-Base.repo http://mirrors.163.com/.help/CentOS6-Base-163.repo
```

## 安装必要的工具和软件

```bash
# yum -y install openssh-clients policycoreutils-python
# yum -y install yum-plugin-{security,priorities} yum-utils
# yum -y install system-config-{firewall-tui,network-tui} ntsysv
# yum -y install wget lftp rsync elinks mutt tree vim-enhanced git
# yum -y install jpackage-utils
```

## 配置主机名、网络参数、本地域名解析

```bash
# system-config-network-tui
# service network restart
# vim /etc/hosts
```

## 配置第三方仓库

### 下载仓库的 *release*.rpm

```bash
# cd
# mkdir RPM
# cd RPM
# wget http://mirrors.sohu.com/fedora-epel/6/x86_64/epel-release-6-7.noarch.rpm
# wget http://mirrors.sohu.com/dag/redhat/el6/en/x86_64/rpmforge/RPMS/rpmforge-release-0.5.2-2.el6.rf.x86_64.rpm
# wget http://rpms.famillecollet.com/enterprise/remi-release-6.rpm
# wget http://mirrors.dotsrc.org/jpackage/6.0/generic/free/RPMS/jpackage-release-6-3.jpp6.noarch.rpm
# wget http://nginx.org/packages/centos/6/noarch/RPMS/nginx-release-centos-6-0.el6.ngx.noarch.rpm
```

### 安装 *release*.rpm 并导入仓库公钥

```bash
# rpm -ivh epel-release-6-7.noarch.rpm
# rpm -ivh rpmforge-release-0.5.2-2.el6.rf.x86_64.rpm
# rpm -ivh remi-release-6.rpm
# rpm -ivh jpackage-release-6-3.jpp6.noarch.rpm
# rpm -ivh nginx-release-centos-6-0.el6.ngx.noarch.rpm

# rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-*
```

### 配置仓库优先级

- [base]、[updates]、[extras]、[remi]
  * `priority=1`
- [centosplus]、[contrib]
  * `priority=2`
- [epel]
  * `priority=10`
- [rpmforge]
  * `priority=15`

### 配置使用非官方仓库的国内镜像站点

1、EPEL

```bash
# cd /etc/yum.repos.d/
# cp epel.repo{,.orig}
# sed -i "s/^mirrorlist/#mirrorlist/g" epel.repo
# sed -i "s/^#baseurl/baseurl/g" epel.repo
# sed -i "s#download.fedoraproject.org/pub/epel#mirrors.sohu.com/fedora-epel#g" epel.repo
```

2、RPMForge

```bash
# cd /etc/yum.repos.d/
# cp rpmforge.repo{,.orig}
# sed -i "s#baseurl = http://apt.sw.be#baseurl = http://mirrors.sohu.com/dag#g" rpmforge.repo
# yum makecache
```

## 安装 etckeeper 并更新系统

```bash
# yum -y install etckeeper tig
# etckeeper init
# etckeeper commit 'init'
# cd /etc
# tig
```

```bash
# yum -y update
# reboot
```

## 安装配置 JDK6

### 安装 JDK6

```bash
# cd; mkdir jdk6; cd jdk6
## 从 http://www.oracle.com/technetwork/java/javase/downloads/index.html 下载最新版
# ll
-rw-r--r--. 1 root root 68826379  5月 17 05:22 jdk-6u33-linux-x64-rpm.bin
# chmod +x jdk-6u33-linux-x64-rpm.bin
# ./jdk-6u33-linux-x64-rpm.bin
```

### 配置 JDK6

1、使用 alternatives 配置当前使用的 Java

```bash
# alternatives --install /usr/bin/java java /usr/java/jdk1.6.0_33/jre/bin/java 20000
# alternatives --install /usr/bin/javac javac /usr/java/jdk1.6.0_33/bin/javac 20000
# alternatives --install /usr/bin/jar jar /usr/java/jdk1.6.0_33/bin/jar 20000

# java -version
java version "1.6.0_33"
Java(TM) SE Runtime Environment (build 1.6.0_33-b03)
Java HotSpot(TM) 64-Bit Server VM (build 20.8-b03, mixed mode)
# javac -version
javac 1.6.0_33

### 日后安装其他 JDK 后可以重新配置当前的 Java
# alternatives --config java
```

[参考](http://www.if-not-true-then-false.com/2010/install-sun-oracle-java-jdk-jre-6-on-fedora-centos-red-hat-rhel/)

2、配置环境变量

- 系统级别 —— /etc/profile
- 用户级别 —— ~/.bashrc

```bash
#  echo '
> export JAVA_HOME=/usr/java/default
> export PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH
> export CLASSPATH=.:$JAVA_HOME/lib:$JAVA_HOME/jre/lib:$CLASSPATH
> ' >> /etc/profile
```

## 安装配置 Tomcat7 和 Ngnix1.2

### 安装 Tomcat7 和 Ngnix1.2

```bash
# yum -y install tomcat7 nginx
# chkconfig --level 345 tomcat7 on 
# chkconfig --level 345 nginx on
```

参考: [不使用 yum 仓库，直接解包安装 Tomcat7](http://www.distrotips.com/centos/installing-tomcat-7-on-centos-6.html)
TODO: [Installing Jenkins](https://wiki.jenkins-ci.org/display/JENKINS/Installing+Jenkins+on+RedHat+distributions)

### 配置 Tomcat7

1、配置 Tomcat7 所需的环境变量

```bash
# vim /etc/tomcat7/tomcat7.conf

## 在如下行
#JAVA_HOME="/usr/lib/jvm/java"
## 之后添加如下行
JAVA_HOME="/usr/java/default"

## 对于生产环境，根据机器性能适当调整 Java 使用的内存
## 在如下行
#JAVA_OPTS="-Xminf0.1 -Xmaxf0.3"
## 之后添加如下行
JAVA_OPTS="-Xms2624m -Xmx2624m -Xss2024K -XX:PermSize=528m -XX:MaxPermSize=856m"
```

2、安装默认 Web 应用案例及文档并测试

```bash
# yum install tomcat7-webapps tomcat7-docs-webapp
# service tomcat7 restart
# elinks http://localhost:8080
# elinks http://YourIP:8080
```

3、编辑 server.xml 文件

```bash
# vim /etc/tomcat7/server.xml

## 对生产环境优化连接
## 找到如下的行
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
## 将其改为：
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443"
               maxHttpHeaderSize="8192"
               useBodyEncodingForURI="true"
               URIEncoding="UTF-8"
               maxThreads="2048" minSpareThreads="100"
               acceptCount="500"
               enableLookups="false" />
```

4、部署自己的 Webapp 并测试

将自己的应用程序放置在 /var/lib/tomcat7/webapps 目录下替换原来的 ROOT 目录的内容；
或将自己的应用程序的 WAR 文件改名为 `ROOT.war` 放置在/var/lib/tomcat7/webapps 目录下，
然后编辑 /etc/tomcat7/Catalina/localhost/ROOT.xml，添加如下内容进行部署：

```xml
<Context path="" docBase="ROOT.war"></Context>
```

因为需要 tomcat 部署 ROOT 目录，且 jpackage 的 Tomcat7 以 tomcat 用户运行，
所以需要适当调整 webapps 目录的权限：

```bash
# chown tomcat /var/lib/tomcat7/webapps
```

重新启动 tomcat 并进行浏览测试。

```bash
# service tomcat7 restart
# elinks http://localhost:8080
# elinks http://YourIP:8080
```

### 配置 Ngnix

1、配置 proxy 通用参数

```bash
# vim /etc/nginx/conf.d/proxy.conf

proxy_redirect     off;
proxy_set_header   Host $host;
proxy_set_header   X-Real-IP $remote_addr;
proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;

proxy_connect_timeout     60;
proxy_read_timeout        60;
proxy_send_timeout        60;
proxy_buffer_size        32k;
proxy_buffers          4 64k;
proxy_busy_buffers_size 128k;
proxy_temp_file_write_size 128k;
proxy_next_upstream error timeout invalid_header http_500 http_503 http_404;
```

2、修改默认主机配置

```bash
# cd /etc/nginx/conf.d/
# cp default.conf{,.orig}
# vim default.conf
```

```nginx
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        ## 添加如下行
        proxy_pass  http://127.0.0.1:8080;
     }
```

3、参数调整和优化

```bash
# vim /etc/nginx/conf.d/optimization.conf
```

```nginx
tcp_nodelay on;
tcp_nopush on;

server_tokens off;
server_name_in_redirect off;
server_names_hash_bucket_size 128;

client_header_buffer_size 32k;
client_body_buffer_size 128k;
client_max_body_size 30m;
large_client_header_buffers 4 32k;

gzip on;
gzip_min_length 1k;
gzip_buffers 4 16k;
gzip_http_version 1.1;
gzip_comp_level 2;
gzip_vary on;
gzip_types text/plain application/x-javascript text/css application/xml;
```

4、重启 Nginx，访问由其代理的 Tomcat 站点

```bash
# service nginx restart
### 使用浏览器测试 http://YourIP 或 http://YourFQDN
```

## 安装配置 MySQL 服务

### 安装 MySQL 服务

1、使用 yum 安装 mysql-server

```bash
## 选择1. 安装 remi 仓库提供的 MySQL 5.5 （建议）
# yum --enablerepo=remi install mysql-server mysqltuner
## 选择2. 安装官方仓库提供的 MySQL 5.1 
# yum install mysql-server mysqltuner
```

2、安装后的基本配置

```bash
# chkconfig --level 35 mysqld on
# service mysqld start
## 为 MySQL 的 root 用户设置口令  
# mysqladmin -u root password 'my0wnSecrectpAs5'
## 或运行 MySQL 的安全安装脚本 
# mysql_secure_installation
## 查看服务的参数配置及性能
# mysqltuner
```

### 配置 MySQL 服务

参考 /usr/share/mysql/ 目录下提供的配置文件模板：

```bash
ll /usr/share/mysql/my-*.cnf
-rw-r--r--. 1 root root  4697  6月  1 23:39 /usr/share/mysql/my-huge.cnf
-rw-r--r--. 1 root root 19779  6月  1 23:39 /usr/share/mysql/my-innodb-heavy-4G.cnf
-rw-r--r--. 1 root root  4671  6月  1 23:39 /usr/share/mysql/my-large.cnf
-rw-r--r--. 1 root root  4682  6月  1 23:39 /usr/share/mysql/my-medium.cnf
-rw-r--r--. 1 root root  2846  6月  1 23:39 /usr/share/mysql/my-small.cnf
```

选择合适的模板将其复制到 /etc/my.cnf ，并作适当修改。例如：

```bash
# mv /etc/my.cnf{,.orig}
# cp /usr/share/mysql/my-large.cnf /etc/my.cnf
# vi /etc/my.cnf
# service mysqld restart
# mysqltuner
```

### MySQL 的 UTF-8 配置

```bash
# vi /etc/my.cnf
```

```
[client]
…………
#设置mysql客户端的字符集  
default-character-set=utf8
…………

[mysqld]
…………
#设置服务器段的字符集
character-set-server=utf8 
collation-server=utf8_general_ci
…………
```


## 安装配置 phpMyAdmin

参考: [install-phpmyadmin-on-fedora-centos-red-hat-rhel](http://www.if-not-true-then-false.com/2012/install-phpmyadmin-on-fedora-centos-red-hat-rhel/)

### 安装 phpMyAdmin

```bash
# yum --enablerepo=remi install php php-fpm php-common php-gd php-mcrypt php-mbstring php-xml php-pecl-apc phpMyAdmin
# chkconfig --level 35  php-fpm on
# service php-fpm start 
# chkconfig --level 35  httpd off
# service httpd stop
# service nginx start
```

### 配置 Nginx 和 PHP-FPM 

为 phpMyAdmin 配置虚拟主机。

```bash
# vim /etc/nginx/conf.d/phpMyAdmin.conf
```

```nginx
server {
       listen   80;
       server_name pma.domain.com;
       access_log /var/log/nginx/phpmyadmin/access.log;
       error_log /var/log/nginx/phpmyadmin/error.log;
       root /usr/share/phpMyAdmin;

       location / {
           index  index.php;
       }

       ## Images and static content is treated different
       location ~* ^.+.(jpg|jpeg|gif|css|png|js|ico|xml)$ {
           access_log        off;
           expires           360d;
       }

       location ~ /\.ht {
           deny  all;
       }

       location ~ /(libraries|setup/frames|setup/libs) {
           deny all;
           return 404;
       }

       location ~ \.php$ {
           include /etc/nginx/fastcgi_params;
           fastcgi_pass 127.0.0.1:9000;
           fastcgi_index index.php;
           fastcgi_param SCRIPT_FILENAME /usr/share/phpMyAdmin$fastcgi_script_name;
       }
}
```

```bash
### 检查配置文件语法的正确性
# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: [emerg] open() "/var/log/nginx/phpmyadmin/access.log" failed (2: No such file or directory)
nginx: configuration file /etc/nginx/nginx.conf test failed
### 为分离的日志创建存放目录
# mkdir /var/log/nginx/phpmyadmin/ 
### 重新检查配置文件语法的正确性
# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
### 重启 Nginx
# service nginx restart
### 配置 pma.domain.com 的域名解析后
### 使用浏览器访问 http://pma.domain.com 测试
```

## 配置防火墙

```bash
#  system-config-firewall-tui

## 开启 80 端口。
```

或编辑 /etc/sysconfig/iptables 开启 80 端口。

```bash
# vim /etc/sysconfig/iptables
```

```
# Firewall configuration written by system-config-firewall
# Manual customization of this file is not recommended.
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
COMMIT
```

```bash
# vim /etc/sysconfig/iptables

-A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT
```

重新启动防火墙

```bash
# chkconfig iptables on
# service iptables restart
```

## 备份

### 使用 mylvmbackup 备份 MySQL 数据库

1、安装 mylvmbackup

```bash
# yum install perl-Config-IniFiles perl-TimeDate
# wget http://www.lenzg.net/mylvmbackup/mylvmbackup-0.13-0.noarch.rpm
# rpm -ivh mylvmbackup-0.13-0.noarch.rpm
```

2、配置 mylvmbackup

```bash
# vi /etc/mylvmbackup.conf
```

修改如下配置：

```
[mysql]
user=root
password=my0wnSecrectpAs5
host=localhost
port=3306
socket=/var/lib/mysql/mysql.sock
mycnf=/etc/my.cnf

[lvm]
vgname=vg0
lvname=lv_mysql
backuplv=
lvsize=50G

[fs]
xfs=0
mountdir=/mnt/mylvmbackup
#backupdir=backupuser@backuphost:/data/backup/mysql
backupdir=/backup/mysql/

[logging]
log_method=both
syslog_remotehost=
```

创建必要的目录

```bash
# mkdir -p /mnt/mylvmbackup/
# mkdir -p /backup/mysql/
# chmod 700 /backup/mysql/
```

3、手工执行 mylvmbackup

```bash
# mylvmbackup

# ll /backup/mysql/
-rw-r--r--. 1 root root 571852  6月 24 21:36 backup-20120624_213601_mysql.tar.gz
```

### 安排 cron 任务备份重要数据和 Mysql

```bash
# mkdir /backup/fs
# chmod 700 /backup/fs

# vi /etc/cron.d/backup
```

```
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
HOME=/

0 1 * * * root mylvmbackup
0 3 * * * root rsync -av /etc /root /var/lib/tomcat7/webapps /root /home /backup/fs/
```

```bash
# service crond restart
```