---
layout: post
title: "在 CentOS 6.2 上安装 gitolite + GitLab"
date: 2012-07-12 09:17
comments: true
categories: [CentOS, VCS, Collaboration, Gitolite, GitLab, Apache, Passenger]
published: false
---
## 版本控制系统和软件开发项目管理系统

### 版本控制系统（VCS/SCM）

如今广泛使用的版本控制系统包括：

* 集中式的版本控制系统（CVCS）  
  - [Subversion](http://subversion.tigris.org/)
* 分布式的版本控制系统（DVCS）
  - [Git](http://git-scm.com)
  - [Mercurial](http://hg-scm.com/) 
  
### 源代码托管服务

若你做的是开源项目，可以选择源代码托管服务：

- 老牌的 [sourceforge](http://sf.net)
- Google 旗下的 [googlecode](http://code.google.com/)
- 使用 hg 做为 SCM 的 [bitbucket](http://bitbucket.org/)
- 使用 git 做为 SCM 的 **[github](http://github.com/)** 和 [Gitorious](http://gitorious.org/)

### 软件开发项目管理系统

若公司内部要搭建一个类似源代码托管的服务，可选的开源软件开发项目管理系统包括：

* 老牌的 [trac](http://trac.edgewall.org/) - Subversion 时代的首选
* 使用 Git 做为 SCM 的基于 ROR 开发的开源软件包括： 
  - **[GitLabHQ](https://github.com/gitlabhq/gitlabhq)**
  - [Redmine](http://www.redmine.org/)
  - [Gitorious](http://gitorious.org/gitorious)


## 准备工作

### 选择 Minimal 安装 CentOS 6.2/6.3

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

## 方案说明
  
下面将介绍在 CentOS 6.2 系统下安装 GitLab 的过程。

- 需要创建一个专用的仓库账号
  - 通常名为 git
  - **此账号应该禁止以口令方式登录 Shell**
- 为了更好地管理 git 仓库的权限，应该安装 Gitolite
  - 推荐以独立的用户身份 gitolite 安装 gitolite 软件
  - EPEL 仓库提供了两个版本的 gitolite
    - V2版本的用户是 gitolite；V3版本的用户是 gitolite3。
- GitLab 是 ROR 应用程序
  - 需要安装 Ruby、 Rail、Apache + Passenger 等相关软件。
  - 推荐以独立的用户身份 gitlab 安装 GitLab
- 为了使 Gitolite 和 GitLab 协同工作，通常：
  - 将用户 gitolite/gitolite3、gitlab、apache 均加入 git 组
  - 将 Gitolite 的管理者用户也加入  git 组
  - 并对相应的目录开放对 git 组的写权限


## gitolite

### 准备专用的仓库账号

```bash
useradd -r -m -c 'git version control' git
```

### 安装 gitolite3

```bash
yum install gitolite3
```

### 为 gitolite 的管理者准备 SSH 用户公钥

将在客户端上生成的 SSH 用户公钥复制到服务器的 /tmp 目录

例如要使用本地新建的 gadmin 用户的公钥需要如下操作：

```bash
useradd gadmin
su - gadmin
ssh-keygen -q -N '' -t rsa -f ~/.ssh/id_rsa
cp ~/.ssh/id_rsa.pub /tmp/gadmin.pub
ssh-keyscan localhost >> ~/.ssh/known_hosts
chmod 644 ~/.ssh/known_hosts
usermod -G git gadmin
exit
```

### 初始化配置 gitolite3

```bash
su - git
gitolite setup -pk /tmp/gadmin.pub
ssh-keyscan localhost >> ~/.ssh/known_hosts
chmod 644 ~/.ssh/known_hosts

chmod 770 ~
chmod 600 ~/.ssh/authorized_keys
chmod 660 projects.list
chmod -R g+rwX repositories/
chmod -R o-rwx repositories/
# 将 ~/.gitolite.rc 文件中的 UMASK => 0077 改为  UMASK => 0007
sed -i 's/0077/0007/g' ~/.gitolite.rc
exit
```

### 以 gitolite3 的管理者身份克隆 gitolite-admin 仓库

```bash
su - gadmin
cat <<END> .ssh/config
host gitolite
    user git
    hostname localhost
    port 22
    identityfile ~/.ssh/gadmin
END

chmod 600 .ssh/config

```

参考： http://sitaramc.github.com/gitolite/glssh.html
