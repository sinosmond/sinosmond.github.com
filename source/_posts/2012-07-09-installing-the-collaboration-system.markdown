---
layout: post
title: "在 CentOS 6.2 上安装协作系统"
date: 2012-07-09 11:24
comments: true
categories: [CentOS, Collaboration, DokuWiki, eGroupware]
toc: true
---
## 说明

协作系统的开源软件很丰富，常用的有：

- [Dokuwiki](http://dokuwiki.org/) | [MoinMoin](http://moinmo.in/) -- 通常用于文档协作 
- [eGroupware](http://www.egroupware.org/) | [TikiWiki](http://tiki.org/) -- 通常用于通用的项目协作
- [GitLabHQ](https://github.com/gitlabhq/gitlabhq) | [Redmine](http://www.redmine.org/) | [Gitorious](http://gitorious.org/gitorious) -- 通常用于软件开发项目的协作


## CentOS 6.2 的安装和基本配置

### 选择 Minimal 安装 CentOS 6.2

配置使用官方仓库的国内镜像站点

```bash
cd /etc/yum.repos.d/
mv CentOS-Base.repo{,.orig}
wget -O CentOS-Base.repo http://mirrors.163.com/.help/CentOS6-Base-163.repo
```

### 安装必要的工具和软件

```bash
yum -y install openssh-clients policycoreutils-python
yum -y install yum-plugin-{security,priorities} yum-utils
yum -y install system-config-{firewall-tui,network-tui} ntsysv
yum -y install wget lftp rsync elinks mutt tree vim-enhanced git
yum -y install jpackage-utils
```

### 配置主机名、网络参数、本地域名解析

```bash
system-config-network-tui
service network restart
vim /etc/hosts
```

### 配置第三方仓库

1、安装 *release*.rpm 并导入仓库公钥

```bash
cd ;  mkdir RPM ;  cd RPM
wget http://mirrors.sohu.com/fedora-epel/6/x86_64/epel-release-6-7.noarch.rpm
wget http://mirrors.sohu.com/dag/redhat/el6/en/x86_64/rpmforge/RPMS/rpmforge-release-0.5.2-2.el6.rf.x86_64.rpm
rpm -ivh epel-release-6-7.noarch.rpm
rpm -ivh rpmforge-release-0.5.2-2.el6.rf.x86_64.rpm

rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-*
```

2、配置仓库优先级

- [base]、[updates]、[extras]
  * `priority=1`
- [centosplus]、[contrib]
  * `priority=2`
- [epel]
  * `priority=10`
- [rpmforge]
  * `priority=15`

3、配置使用非官方仓库的国内镜像站点

(1) EPEL

```bash
cd /etc/yum.repos.d/
cp epel.repo{,.orig}
sed -i "s/^mirrorlist/#mirrorlist/g" epel.repo
sed -i "s/^#baseurl/baseurl/g" epel.repo
sed -i "s#download.fedoraproject.org/pub/epel#mirrors.sohu.com/fedora-epel#g" epel.repo
```

(2) RPMForge

```bash
cd /etc/yum.repos.d/
cp rpmforge.repo{,.orig}
sed -i "s#baseurl = http://apt.sw.be#baseurl = http://mirrors.sohu.com/dag#g" rpmforge.repo
yum makecache
```

### 安装 etckeeper 并更新系统

```bash
yum -y install etckeeper tig
etckeeper init
etckeeper commit 'init'
cd /etc
tig
```

```bash
yum -y update
reboot
```

### 安装 LAMP 环境

```bash
yum install mysql-server mysqltuner
yum install httpd php php-common php-gd php-mcrypt php-mbstring php-xml php-pear php-pecl-apc php-imap
yum install php-mysql php-pdo phpMyAdmin

chkconfig --level 35 mysqld on
chkconfig --level 35 httpd on

service mysqld start
service httpd start

```

### 配置防火墙

```bash
system-config-firewall-tui

## 开启 80 端口。
```

## Apache + PHP + Dokuwiki

### 下载最新的 DokuWIki 稳定版本

EPEL 仓库提供了 Dokuwiki 的 RPM，可以直接使用 yum 安装，但其版本较旧。 下面下载最新的稳定版本。

```bash
cd /var/www/
wget http://www.splitbrain.org/_media/projects/dokuwiki/dokuwiki-2012-01-25a.tgz
tar xzf dokuwiki-2012-01-25a.tgz
mv dokuwiki-2012-01-25a dokuwiki
chown root.root dokuwiki
chown apache dokuwiki/conf
chown apache dokuwiki/data -R
```

### 编写 Dokuwiki 的 Apache 配置文件

```bash
vim /etc/httpd/conf.d/dokuwiki.conf
```

```apache
Alias /wiki /var/www/dokuwiki

<Directory /var/www/dokuwiki>
        order deny,allow
        allow from all

### 当在 Dokuwiki 配置界面的 “高级设置” 部分，将 "使用更整洁的 URL" 设置为 “.htaccess” 
### 或 在 /var/www/dokuwiki/conf/local.php 文件中配置了 $conf['userewrite'] = '1';
### 之后，清除如下 Rewrite* 配置指令的注释符 
#  RewriteEngine on
#  RewriteBase /wiki

#  RewriteCond %{HTTPS} !=on
#  RewriteRule ^lib/exe/xmlrpc.php$      https://%{SERVER_NAME}%{REQUEST_URI} [L,R=301]

#  RewriteRule ^_media/(.*)              lib/exe/fetch.php?media=$1  [QSA,L]
#  RewriteRule ^_detail/(.*)             lib/exe/detail.php?media=$1  [QSA,L]
#  RewriteRule ^_export/([^/]+)/(.*)     doku.php?do=export_$1&id=$2  [QSA,L]
#  RewriteRule ^$                        doku.php  [L]
#  RewriteCond %{REQUEST_FILENAME}       !-f
#  RewriteCond %{REQUEST_FILENAME}       !-d
#  RewriteRule (.*)                      doku.php?id=$1  [QSA,L]
#  RewriteRule ^index.php$               doku.php
</Directory>

<LocationMatch "/wiki/(data|conf|bin|inc)/">
        order allow,deny
        deny from all
        satisfy all
</LocationMatch>

<Files ~ "^([\._]ht|README$|VERSION$|COPYING$)">
    Order allow,deny
    Deny from all
    Satisfy All
</Files>
```

```bash
service httpd restart
```

### 安装常用的 Dokuwiki 插件

```bash
cd /var/www/dokuwiki/lib/plugins
## 安装 sidebar 插件（这是支持 sidebar 的最简单的方法）
## （您也可以下载支持 sidebar 的模板，但要停用此插件）
git clone git://github.com/chimeric/dokuwiki-plugin-sidebarng.git sidebarng
## 安装 snippets 插件（可使用此插件实现页面模板的功能）
git clone git://github.com/cosmocode/dokuwiki-plugin-snippets.git snippets
## 安装 语法 插件
git clone git://github.com/selfthinker/dokuwiki_plugin_wrap.git wrap
git clone git://github.com/dokufreaks/plugin-comment.git comment
## 安装 功能 插件
git clone git://github.com/dokufreaks/plugin-pagelist.git pagelist
git clone git://github.com/dokufreaks/plugin-include.git include
git clone git://github.com/dokufreaks/plugin-tag.git tag
git clone git://github.com/dokufreaks/plugin-cloud.git cloud
git clone git://github.com/splitbrain/dokuwiki-plugin-captcha.git captcha
git clone git://github.com/dokufreaks/plugin-discussion.git discussion
git clone git://github.com/cosmocode/pagenav.git pagenav
git clone git://github.com/glensc/dokuwiki-plugin-lightboxv2.git lightbox
git clone git://github.com/splitbrain/dokuwiki-plugin-gallery.git gallery
#git clone git://github.com/chimeric/dokuwiki-plugin-wikicalendar.git wikicalendar
#git clone git://github.com/splitbrain/dokuwiki-plugin-jukebox.git jukebox
#git clone git://github.com/splitbrain/dokuwiki-plugin-vshare.git vshare
#git clone git://github.com/gturri/nspages.git nspages
#git clone git://github.com/splitbrain/dokuwiki-plugin-s5.git s5
#git clone git://github.com/splitbrain/dokuwiki-plugin-translation.git translation
#git clone git://github.com/dokufreaks/plugin-blogtng.git blogtng
#git clone git://github.com/dokufreaks/plugin-blog.git blog
#（blog 和 blogng 不能同时使用，当前 blogng 需要 PHP 对 SQLite2 的支持）
## 安装 维护 插件
git clone git://github.com/Imhodad/dokuwiki-plugin-searchstats.git searchstats
git clone git://github.com/cosmocode/clearhistory.git clearhistory
git clone git://github.com/Taggic/orphanmedia.git orphanmedia
git clone git://github.com/Taggic/langdelete.git langdelete
git clone git://github.com/MrBertie/pagequery.git pagequery
git clone git://github.com/danny0838/dw-editx.git editx
#git clone git://github.com/desolat/DokuWiki-Pagemove-Plugin.git pagemove
git clone git://github.com/splitbrain/dokuwiki-plugin-sync.git sync
git clone git://github.com/splitbrain/dokuwiki-plugin-badbehaviour.git badbehaviour
```

### 安装和配置 Dokuwiki

- 您可以使用 Web 界面安装 Dokuwiki，使用浏览器打开 http://IPorFQDN/wiki/install.php 安装 Dokuwiki。
- 您也可以直接编辑配置文件 /var/www/dokuwiki/conf/local.php 实现 DokuWiki 的配置。如下是一个配置文件例子：

```php
<?php
/*
 * Dokuwiki's Main Configuration File - Local Settings
 * Auto-generated by config plugin
 * Run for user: osmond
 * Date: Thu, 05 Jul 2012 17:47:40 +0800
 */

$conf['title'] = 'MyWiki';
$conf['lang'] = 'zh';
$conf['license'] = '1';
$conf['youarehere'] = 1;
$conf['typography'] = '0';
$conf['useheading'] = '1';
$conf['useacl'] = 1;
$conf['superuser'] = '@admin';
$conf['disableactions'] = 'register';
$conf['userewrite'] = '1';
$conf['useslash'] = 1;
$conf['hidepages'] = 'snippets|sidebar';
$conf['plugin']['wikicalendar']['timezone'] = 'Asia/Shanghai';
$conf['plugin']['pagelist']['showdate'] = '2';
$conf['plugin']['pagelist']['showtags'] = 1;
$conf['plugin']['tag']['toolbar_icon'] = 1;
$conf['plugin']['s5']['template'] = 'flower';
$conf['plugin']['discussion']['automatic'] = 1;
$conf['plugin']['discussion']['allowguests'] = 0;
$conf['plugin']['discussion']['linkemail'] = 1;
$conf['plugin']['discussion']['useavatar'] = 0;
$conf['plugin']['discussion']['newestfirst'] = 1;
$conf['plugin']['translation']['translations'] = 'en';
$conf['plugin']['translation']['about'] = 'en';

// end auto-generated content
```


## Apache + PHP + MySQL + eGroupware

### 安装 eGroupware

```bash
cd /etc/yum.repos.d/
wget http://download.opensuse.org/repositories/server:eGroupWare/CentOS_6/server:eGroupWare.repo

echo "priority=5" >> server\:eGroupWare.repo

vim rpmforge.repo
# 在 [rpmforge] 段末尾添加如下行
exclude = php-jpgraph
# 保存退出 vi

yum makecache
yum install eGroupware
```

### 安装 eGroupware 的注意事项 

1、RPM 安装后执行的脚本：/usr/share/egroupware/doc/rpm-build/post_install.php，若您之前已经设置过 MySQL 的 root 口令，
请适当编辑此文件，之后重新执行此脚本。

```bash
# 运行如下命令可获得更多信息：
/usr/share/egroupware/doc/rpm-build/post_install --help 

vim /usr/share/egroupware/doc/rpm-build/post_install.php
```

```php
        'db_root'     => 'root',        // mysql root user/pw to create database
        'db_root_pw'  => 'YourPassword',
```

```bash
php /usr/share/egroupware/doc/rpm-build/post_install.php
```

2、安装日志： ~/eGroupware-install.log

3、eGroupware 的 RPM 自动安装了 Apache 的配置文件 /etc/httpd/conf.d/egroupware.conf。
以便可以使用 http://IPorFQDN/egroupware 的 URL 访问。

参考：[eGroupware Release notes  1.8](http://community.egroupware.org/index.php?page_name=wiki&lang=&wikipage=releasenotes1.8)


### 解决项目模块的甘特图中文乱码问题

1、从 Windows 系统上传字体文件

```bash
mkdir /usr/share/fonts/truetype/
### 上传字体文件
ls 
simfang.ttf  simhei.ttf  simsun.ttc
### 创建链接文件
cd /usr/share/egroupware/projectmanager/inc/ttf-bitstream-vera-1.10
ln /usr/share/fonts/truetype/simsun.ttc
ln /usr/share/fonts/truetype/simhei.ttf
ln /usr/share/fonts/truetype/simfang.ttf
```

2、修改 jpgraph 的 jpgraph_ttf.inc.php

```bash
vim /usr/share/jpgraph/src/jpgraph_ttf.inc.php
```

```php
            /* Chinese fonts */
            FF_SIMSUN  =>  array(
                FS_NORMAL =>'simsun.ttc',
                FS_BOLD =>'simhei.ttf',
                FS_ITALIC =>'simfang.ttf',
                FS_BOLDITALIC =>'simhei.ttf' ),
```

3、 修改 egroupware 的项目模块的相应代码

```bash
vim /usr/share/egroupware/projectmanager/inc/class.projectmanager_ganttchart.inc.php
```

```php
找到并注释掉如下两行
// if ($this->gantt_char_encode) $text = preg_replace('/[^\x00-\x7F]/e', '"&#".ord("$0").";"',$text);

// $graph->scale->SetDateLocale(common::setlocale());
```