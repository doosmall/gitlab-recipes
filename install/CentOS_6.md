**centos6.3上 gitlab4.0安装指南。**
在RHEL 6.3 上也进行了安装测试，到目前发现并记录了一些不同。

阅读 `doc/install/requirements.md` 了解安装gitlab4.0时硬件和平台环境的要求。

## 概述 ##
本文主要介绍在一个全新的操作系统上安装基于MySQL数据库的 gitlab4.0。

**提示：**
本文的安装步骤测试能够正常执行.
也可以不按照本文的步骤进行安装，但是要遵照gitlab4.0约定的运行环境进行安装(如果你了解的话)。
在AWS上安装或者其他web服务器的配置等请参考"高级技巧" 章节.

**提示：**
如果发现本文的错误还请提交给给我们

**提示：**
绝大多数情况下，你需要用系统root用户执行命令。
需要的时候你需要切换到'git' 或者 'gitlab' 用户上执行命令，

从root用户切换到其它用户用下面的命令，如：

    su - gitlab

**提示：**
很多Linux上的软件安装教程简单的规定: "禁止 selinux 和 防火墙".
ubuntu上的原生 gitlab安装需要完全禁止 StrictHostKeyChecking.
而本安装教程不需要禁止任何安全项，我们只需要按照安全策略简单的进行配置.

- - -

# 安装概要

 GitLab的安装主要有下列组件的安装构成：

1. 安装操作系统 (如CentOS 6.3 Minimal) 和依赖库（ Packages / Dependencies）
2. 安装Ruby
3. 创建系统用户
4. 安装Gitolite
5. 安装GitLab


----------

# 1. 安装操作系统 (CentOS 6.3 Minimal)

首先需要下载一个全新的 CentOS 6.3 "minimal" 系统。 如果知识测试安装可以下载centos的ISO文件用虚拟机安装
centos6.3下载地址：http://mirrors.163.com/centos/
VirtualBox下载：https://www.virtualbox.org/wiki/Downloads
## 增加和更新基本的软件和服务
### 增加 EPEL repository

*登陆到root账号*

    rpm -Uvh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm

### 安装 gitlab 和 gitolite 需要的工具

*登陆到root账号*

    yum -y groupinstall 'Development Tools'

    ### 'Additional Development'
    yum -y install vim-enhanced httpd readline readline-devel ncurses-devel gdbm-devel glibc-devel \
                   tcl-devel openssl-devel curl-devel expat-devel db4-devel byacc \
                   sqlite-devel gcc-c++ libyaml libyaml-devel libffi libffi-devel \
                   libxml2 libxml2-devel libxslt libxslt-devel libicu libicu-devel \
                   system-config-firewall-tui python-devel redis sudo mysql-server wget \
                   mysql-devel crontabs logwatch logrotate sendmail-cf qtwebkit qtwebkit-devel \
                   perl-Time-HiRes

**关于 Redhat EL 6** 


### 更新 CentOS 到最新

*登陆到root账号*

    yum -y update

## 配置 redis
确保redis在下次重启系统时可以自动运行

*登陆到root账号*

    chkconfig redis on
    service redis start

## 配置 mysql
确保MySQL在下次重启系统时可以自动运行。

*登陆到root账号*

    chkconfig mysqld on
    service mysqld start

配置 MySQL ， 设置MySQL root账号的密码，根据提示一路"Yes" 

    /usr/bin/mysql_secure_installation

## 配置 httpd

我们用 Apache 作为gitlab的前端
确保Apache在下次重启系统时可以自动运行。

    chkconfig httpd on

创建文件 **/etc/httpd/conf.d/gitlab.conf** ，并且增加下面的内容(替换 git.example.org 为你的域名!!). 

    <VirtualHost *:80>
      ServerName git.example.org
      ProxyRequests Off
        <Proxy *>
           Order deny,allow
           Allow from all
        </Proxy>
        ProxyPreserveHost On
        ProxyPass / http://localhost:3000/
        ProxyPassReverse / http://localhost:3000/
    </VirtualHost>

注: 如果还有其他web站点在同一台服务器上，你需要在 **/etc/httpd/conf/httpd.conf** 如下配置

    NameVirtualHost *:80

还要配置 selinux 

    setsebool -P httpd_can_network_connect on

## 配置防火墙

   修改 **/etc/sysconfig/iptables** 增加如下内容

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
    -A INPUT -m state --state NEW -m tcp -p tcp --dport 443 -j ACCEPT
    -A INPUT -j REJECT --reject-with icmp-host-prohibited
    -A FORWARD -j REJECT --reject-with icmp-host-prohibited
    COMMIT

## 配置 email

    cd /etc/mail
    vim /etc/mail/sendmail.mc

增加一行 smtp gateway hostname

    define(`SMART_HOST', `smtp.example.com')dnl

注释掉下面这行

    EXPOSED_USER(`root')dnl

至于奥在这行前面增加 'dnl ' 如：

    dnl EXPOSED_USER(`root')dnl
 
启用这个配置

    make
    chkconfig sendmail on


## 重启系统

    reboot

----------

# 2. 安装Ruby
下载和编译:

*logged in as root*

    mkdir /tmp/ruby && cd /tmp/ruby
    wget http://ftp.ruby-lang.org/pub/ruby/1.9/ruby-1.9.3-p327.tar.gz
    tar xfvz ruby-1.9.3-p327.tar.gz
    cd ruby-1.9.3-p327
    ./configure
    make
    make install

Install the Bundler Gem:

*logged in as root*

    gem install bundler

----------

# 3. System Users

## Create users for Git and Gitolite
*logged in as root*

    adduser \
      --system \
      --shell /bin/bash \
      --comment 'Git Version Control' \
      --create-home \
      --home-dir /home/git \
      git

    adduser \
      --shell /bin/bash \
      --comment 'GitLab user' \
      --create-home \
      --home-dir /home/gitlab \
      gitlab

    usermod -a -G git gitlab 

Because the gitlab user will need a password later on, we configure it right now, so we are finished with all the user stuff.

    passwd gitlab # please choose a good password :)  

*logged in as root*

    # Generate the SSH key
    sudo -u gitlab -H ssh-keygen -q -N '' -t rsa -f /home/gitlab/.ssh/id_rsa

## Forwarding all emails

Now we want all logging of the system to be forwarded to a central email address

*logged in as root*

    echo adminlogs@example.com > /root/.forward
    chown root /root/.forward
    chmod 600 /root/.forward
    restorecon /root/.forward

    echo adminlogs@example.com > /home/gitlab/.forward
    chown gitlab /home/gitlab/.forward
    chmod 600 /home/gitlab/.forward
    restorecon /home/gitlab/.forward
    
----------

# 4. Gitolite

## Clone GitLab's fork of the Gitolite source code:

*logged in as root*

    cd /home/git
    sudo -u git -H git clone -b gl-v320 https://github.com/gitlabhq/gitolite.git /home/git/gitolite

## Setup Gitolite with GitLab as its admin:

**Important Note:**
GitLab assumes *full and unshared* control over this Gitolite installation.

*logged in as root*

    # Add Gitolite scripts to $PATH
    sudo -u git -H mkdir /home/git/bin
    sudo -u git -H sh -c 'printf "%b\n%b\n" "PATH=\$PATH:/home/git/bin" "export PATH" >> /home/git/.profile'
    sudo -u git -H sh -c 'gitolite/install -ln /home/git/bin'

    # Copy the gitlab user's (public) SSH key ...
    cp /home/gitlab/.ssh/id_rsa.pub /home/git/gitlab.pub
    chmod 0444 /home/git/gitlab.pub

    # ... and use it as the admin key for the Gitolite setup
    sudo -u git -H sh -c "PATH=/home/git/bin:$PATH; gitolite setup -pk /home/git/gitlab.pub"

### Fix the directory permissions for the configuration directory:

    # Make sure the Gitolite config dir is owned by git
    chmod 750 /home/git/.gitolite/
    chown -R git:git /home/git/.gitolite/

### Fix the directory permissions for the repositories:

    # Make sure the repositories dir is owned by git and it stays that way
    chmod -R ug+rwXs,o-rwx /home/git/repositories/
    chown -R git:git /home/git/repositories/

    # Make sure the gitlab user can access the required directories
    chmod g+x /home/git

### Make the git account known and allowed to the gitlab user

*logged in as root*

    su - gitlab

*logged in as **gitlab***    
    
    ssh git@localhost  # type 'yes' and press <Enter>.

The expected behaviour is that you get a message similar to this and then immediately the connection is closed again:

    PTY allocation request failed on channel 0
    hello gitlab, this is git@gitlab running gitolite3 v3.2-gitlab-patched-0-g2d29cf7 on git 1.7.1

## Test if everything works so far
*logged in as **gitlab***

    # Clone the admin repo so SSH adds localhost to known_hosts ...
    # ... and to be sure your users have access to Gitolite
    git clone git@localhost:gitolite-admin.git /tmp/gitolite-admin

    # If it succeeded without errors you can remove the cloned repo
    rm -rf /tmp/gitolite-admin

**Important Note:**
If you can't clone the `gitolite-admin` repository: **DO NOT PROCEED WITH INSTALLATION**!
Check the [Trouble Shooting Guide](https://github.com/gitlabhq/gitlab-public-wiki/wiki/Trouble-Shooting-Guide)
and make sure you have followed all of the above steps carefully.

----------
# 5. GitLab

*logged in as gitlab*

    # We'll install GitLab into home directory of the user "gitlab"
    cd /home/gitlab

## Clone the Source

    # Clone GitLab repository
    git clone https://github.com/gitlabhq/gitlabhq.git gitlab

    # Go to gitlab dir 
    cd /home/gitlab/gitlab
   
    # Checkout to stable release
    git checkout 4-0-stable

**Note:**
You can change `4-0-stable` to `master` if you want the *bleeding edge* version, but
do so with caution!

## Configure it

Copy the example GitLab config

    cp /home/gitlab/gitlab/config/gitlab.yml{.example,}

Edit the gitlab config to make sure to change "localhost" to the fully-qualified domain name of your host serving GitLab where necessary. Also review the other settings to match your setup.

    vim /home/gitlab/gitlab/config/gitlab.yml

Copy the example Unicorn config
    cp /home/gitlab/gitlab/config/unicorn.rb{.example,}

Edit the unicorn config

    vim /home/gitlab/gitlab/config/unicorn.rb

Change the listen parameter so that it reads:

    listen "127.0.0.1:3000"  # listen to port 3000 on the loopback interface

Also review the other settings to match your setup.

## Configure GitLab DB settings

    # MySQL
    cp /home/gitlab/gitlab/config/database.yml{.mysql,}

Edit the database config and set the correct username/password

    vim /home/gitlab/gitlab/config/database.yml

The config should look something like this (where supersecret is replaced with your real password):

    production:
      adapter: mysql2
      encoding: utf8
      reconnect: false
      database: gitlabhq_production
      pool: 5
      username: gitlab
      password: supersecret
      # host: localhost
      # socket: /tmp/mysql.sock
    
## Install Gems
*logged in as **gitlab***

    logout

*logged in as **root***

    cd /home/gitlab/gitlab

    gem install charlock_holmes --version '0.6.9'

    su - gitlab

*logged in as **gitlab***

    cd /home/gitlab/gitlab

    # For mysql db
    bundle install --deployment --without development test postgres

## Configure Git

GitLab needs to be able to commit and push changes to Gitolite. In order to do
that Git requires a username and email. (We recommend using the same address
used for the `email.from` setting in `config/gitlab.yml`)

*logged in as gitlab*

    git config --global user.name "GitLab"
    git config --global user.email "gitlab@localhost"

## Setup GitLab Hooks
*logged in as **gitlab***

    logout

*logged in as **root***

    cd /home/gitlab/gitlab
    cp ./lib/hooks/post-receive /home/git/.gitolite/hooks/common/post-receive
    chown git:git /home/git/.gitolite/hooks/common/post-receive

## Initialise Database and Activate Advanced Features

*logged in as **root***

    su - gitlab

*logged in as **gitlab***

    cd /home/gitlab/gitlab
    bundle exec rake gitlab:app:setup RAILS_ENV=production

The previous command will ask you for the root password of the mysql database and create the defined database and user.

## Install Init Script

Download the init script (will be /etc/init.d/gitlab)

*logged in as **gitlab***

    logout

*logged in as root*

    curl https://raw.github.com/gitlabhq/gitlab-recipes/4-0-stable/init.d/gitlab-centos > /etc/init.d/gitlab
    chmod +x /etc/init.d/gitlab
    chkconfig --add gitlab

Make GitLab start on boot:

    chkconfig gitlab on

Start your GitLab instance:

    service gitlab start
    # or
    /etc/init.d/gitlab start

## Check Application Status

Check if GitLab and its environment is configured correctly:

    su - gitlab

*logged in as **gitlab***

    cd /home/gitlab/gitlab
    bundle exec rake gitlab:env:info RAILS_ENV=production

To make sure you didn't miss anything run a more thorough check with:

    cd /home/gitlab/gitlab
    bundle exec rake gitlab:check RAILS_ENV=production

If you are all green: congratulations, you successfully installed GitLab!
Although this is the case, there are still a few steps to go.

# Done!

Visit YOUR_SERVER for your first GitLab login.
The setup has created an admin account for you. You can use it to log in:

    admin@local.host
    5iveL!fe

**Important Note:**
Please go over to your profile page and immediately change the password, so
nobody can access your GitLab by using this login information later on.

**Enjoy!**


- - -


# Advanced Setup Tips

## Custom Redis Connection

If you'd like Resque to connect to a Redis server on a non-standard port or on
a different host, you can configure its connection string via the
`config/resque.yml` file.

    # example
    production: redis.example.tld:6379


## User-contributed Configurations

You can find things like  AWS installation scripts, init scripts or config files
for alternative web server in our [recipes collection](https://github.com/gitlabhq/gitlab-recipes/).

