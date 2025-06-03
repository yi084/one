

准备前置
Centos7虚拟机三台，分别为
控制节点(kz,仅主机模式ip：172.168.201.11,4g内存4个cpu,两张网卡,120g硬盘内存)
计算节点(js,仅主机模式ip：172.168.201.12,4g内存4个cpu,两张网卡,120g硬盘内存)
存储节点(cc,仅主机模式ip：172.168.201.13,4g内存4个cpu,两张网卡,120g硬盘内存)
代理可以不用设置

## 1.修改网卡

#### 三台都修改

``` sh
vim /etc/sysconfig/network-scripts/ifcfg-ens36
TYPE="Ethernet"
BOTTPROTO="static"
NAME="ens36"
DEVICE="ens36"
ONBOOT="yes"
IPADDR="172.168.201.11"
NETMASK="255.255.255.0"
重启网卡 	systemctl restart network
```

#### 换源

```sh
cd /etc/yum.repos.d/
mkdir bak
mv * bak
wget -O /etc/yum.repos.d/CentOS-Base.repo https://repo.huaweicloud.com/repository/conf/CentOS-7-reg.repo 
yum clean all
yum makecache
# 检查是否换源成功
yum repolist
```

#### 修改host文件

``` sh
vim /etc/hosts
172.168.201.11 kz
172.168.201.12 js
172.168.201.13 cc
```

#### 改节点名称

``` sh
# 控制节点
hostnamectl set-hostname kz

# 计算节点
hostnamectl set-hostname js

# 存储节点
hostnamectl set-hostname cc
```

-----

## 2.设置全局永久代理

#### 修改/etc/environment

```sh
vim /etc/environment
http_proxy="http://192.168.200.1:10808"
https_proxy="http://192.168.200.1:10808"
# 立即生效
source /etc/environment
```

#### 修改 Shell 配置文件

```sh
echo 'export http_proxy="http://192.168.200.1:10808"' >> ~/.bashrc
echo 'export https_proxy="http://192.168.200.1:10808"' >> ~/.bashrc
source ~/.bashrc
```

#### 配置yum代理

```sh
vim /etc/yum.conf
proxy=http://192.168.200.1:10808
```

#### 配置Wget代理

```sh
echo 'https_proxy = http://192.168.200.1:10808' >> ~/.wgetrc
```

#### 测试

```sh
# 测试代理连通性
curl -v -x http://192.168.200.1:10808 https://www.google.com

# 查看 wget 是否配置了代理
cat ~/.wgetrc | grep -i proxy
wget -e use_proxy=yes -e http_proxy=http://192.168.200.1:10808 -O /dev/null http://www.example.com
sudo yum --setopt=proxy=http://192.168.200.1:10808 install -y nano
```

-----

## 3.控制节点

#### 安装组件chrony

```sh
yum -y install chrony
```

#### 设置统一时间

```sh
timedatectl set-timezone Asia/Shanghai
```

#### 设置内部NTP Server

```sh
vim /etc/chrony.conf
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
server 0.centos.pool.ntp.org iburst
server 1.centos.pool.ntp.org iburst
server 2.centos.pool.ntp.org iburst
server 3.centos.pool.ntp.org iburst

# Record the rate at which the system clock gains/losses time.
driftfile /var/lib/chrony/drift

# Allow the system clock to be stepped in the first three updates
# if its offset is larger than 1 second.
makestep 1.0 3

# Enable kernel synchronization of the real-time clock (RTC).
rtcsync

# Enable hardware timestamping on all interfaces that support it.
#hwtimestamp *

# Increase the minimum number of selectable sources required to adjust
# the system clock.
#minsources 2

# Allow NTP client access from local network.
allow 172.168.201.0/24

# Serve time even if not synchronized to a time source.
local stratum 10

# Specify file containing keys for NTP authentication.
#keyfile /etc/chrony.keys

# Specify directory for log files.
logdir /var/log/chrony

# Select which information is logged.
#log measurements statistics tracking
```

#### 启动chrony服务并配置在系统引导时启动

```sh
systemctl enable chronyd.service
systemctl start chronyd.service
```

#### 启动NTP同步

```sh
timedatectl set-ntp yes
```

## 4.计算节点和存储节点执行操作

#### 安装chrony

```sh
yum install chrony -y
```

#### 编辑“/etc/chrony.conf”文件

```sh
vim /etc/chrony.conf
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
# server 0.centos.pool.ntp.org iburst
# server 1.centos.pool.ntp.org iburst
# server 2.centos.pool.ntp.org iburst
# server 3.centos.pool.ntp.org iburst
server kz iburst
# Record the rate at which the system clock gains/losses time.
driftfile /var/lib/chrony/drift

# Allow the system clock to be stepped in the first three updates
# if its offset is larger than 1 second.
makestep 1.0 3

# Enable kernel synchronization of the real-time clock (RTC).
rtcsync

# Enable hardware timestamping on all interfaces that support it.
#hwtimestamp *

# Increase the minimum number of selectable sources required to adjust
# the system clock.
#minsources 2

# Allow NTP client access from local network.
#allow 192.168.0.0/16

# Serve time even if not synchronized to a time source.
#local stratum 10

# Specify file containing keys for NTP authentication.
#keyfile /etc/chrony.keys

# Specify directory for log files.
logdir /var/log/chrony

# Select which information is logged.
#log measurements statistics tracking

```

####  启动chrony 服务并将其配置为在系统引导时启动

```sh
systemctl enable chronyd.service
systemctl restart chronyd.service
```

#### 验证时间同步

```sh
chronyc sources
```

-----

## 4.把OpenStack依赖包下载到本地 

#### 利用python脚本

```python
import requests
from bs4 import BeautifulSoup
import os

# 目标网页地址,得下载这个目录下的所有文件https://vault.centos.org/7.9.2009/cloud/x86_64/openstack-stein/Packages/
base_url = "https://vault.centos.org/7.9.2009/cloud/x86_64/openstack-stein/Packages/p/"
download_folder = "centos_rpms"

# 创建本地保存目录
if not os.path.exists(download_folder):
    os.makedirs(download_folder)

# 获取网页内容
response = requests.get(base_url)
response.encoding = response.apparent_encoding
soup = BeautifulSoup(response.text, 'html.parser')

# 获取所有链接
links = soup.find_all('a')

# 下载以 .rpm 结尾的文件
for link in links:
    href = link.get('href')
    if href and href.endswith('.rpm'):
        rpm_url = base_url + href
        local_path = os.path.join(download_folder, href)
        print(f"正在下载: {href}")
        try:
            file_data = requests.get(rpm_url, stream=True)
            with open(local_path, 'wb') as f:
                for chunk in file_data.iter_content(chunk_size=1024):
                    if chunk:
                        f.write(chunk)
            print(f"下载完成: {href}")
        except Exception as e:
            print(f"下载失败: {href}, 错误: {e}")

print("全部下载完成！")
```

#### 在Windows命令行执行

```sh
cd D:\OpenStack本地仓库
# 开启8000端口
python -m http.server 8000
```

### **创建本地仓库元数据**(root目录下)

```sh
mkdir rpm
# 进入仓库目录
cd /root/rpm
# 下载
wget -r -np -nH --cut-dirs=1 http://192.168.200.1:8000/
# 生成仓库元数据（如果尚未生成）
sudo createrepo .
```

#### 配置YUM本地仓库地址

```sh
# 创建仓库配置文件
sudo tee /etc/yum.repos.d/local-rpm.repo <<'EOF'
[local-rpm]
name=Local RPM Repository
baseurl=file:///root/rpm
enabled=1
gpgcheck=0
EOF
```

#### 清理并更新YUM缓存

```sh
sudo yum clean all
sudo yum makecache
```

## 5.安装OpenStack

```sh
sudo yum -y install python-openstackclient
# 验证安装
openstack --version
# 正常输出示例：openstack 3.18.1
```

### 控制节点执行

## 6.安装数据库

```sh
yum -y install mariadb mariadb-server python2-PyMySQL
```

#### 配置数据库

```sh
vim  /etc/my.cnf.d/openstack.cnf
# 添加如下内容
# bind-address 的IP为控制节点的管理IP
[mysqld]
 bind-address = 172.168.201.11
 default-storage-engine = innodb
 innodb_file_per_table = on
 max_connections = 4096
 collation-server = utf8_general_ci
 character-set-server = utf8
```

####   启动数据库服务，并设为开机启动

```sh
systemctl enable mariadb.service
systemctl start mariadb.service
```

#### 数据库配置

```sh
# 初次安装mariadb，默认的root密码是空的，直接按”Enter”即可，然后再为root用户设置密码，例如“123456”
[root@localhost ~]# mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none): 
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)
Enter current password for root (enter for none): 
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)
Enter current password for root (enter for none): 
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)
Enter current password for root (enter for none): 
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] Y
New password: 
Re-enter new password: 
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] n
 ... skipping.

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] n
 ... skipping.

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```

#### 修改mariadb.service文件

```sh
vim /usr/lib/systemd/system/mariadb.service
# 添加如下内容
[Service]
Type=simple
User=mysql
Group=mysql
LimitNOFILE=65535
LimitNPROC=65535
```

-----

## 7.在所有的节点都执行

#### 修改limits.conf文件

```sh
vim /etc/security/limits.conf
#*               soft    core            0
#*               hard    rss             10000
# 添加内容
* soft nofile 65536
* hard nofile 65536
#@student        hard    nproc           20
#@faculty        soft    nproc           20
#@faculty        hard    nproc           50
#ftp             hard    nproc           0
#@student        -       maxlogins       4
```

#### 修改login文件

```sh
vim /etc/pam.d/login
#%PAM-1.0
auth [user_unknown=ignore success=ok ignore=ignore default=bad] pam_securetty.so
auth       substack     system-auth
auth       include      postlogin
account    required     pam_nologin.so
account    include      system-auth
password   include      system-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
session    optional     pam_console.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      system-auth
session    include      postlogin
-session   optional     pam_ck_connector.so
# 添加内容
session required /lib64/security/pam_limits.so
```

#### 修改sysctl.conf文件

```sh
vim /etc/sysctl.conf
# sysctl settings are defined through files in
# /usr/lib/sysctl.d/, /run/sysctl.d/, and /etc/sysctl.d/.
#
# Vendors settings live in /usr/lib/sysctl.d/.
# To override a whole file, create a new file with the same in
# /etc/sysctl.d/ and put new settings there. To override
# only specific settings, add a file with a lexically later
# name in /etc/sysctl.d/ and put new settings there.
#
# For more information, see sysctl.conf(5) and sysctl.d(5).
# 添加内容
fs.file-max = 65536
```

#### 执行

```sh
sysctl -p
```

------

## 8.控制节点执行

#### 重启数据库服务

```sh
systemctl daemon-reload
systemctl restart mariadb.service
```

#### 查看是否更改生效

##### 进入MySQL数据库

```sh
[root@localhost ~]# mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 2
Server version: 5.5.68-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> 
```

##### 查看MySQL最大连接数

```mysql
MariaDB [(none)]> show variables like 'max_connections';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 4096  |
+-----------------+-------+
1 row in set (0.03 sec)
```

##### 查看当前服务器正在使用的连接数

```mysql
MariaDB [(none)]> show global status like 'Max_used_connections';
+----------------------+-------+
| Variable_name        | Value |
+----------------------+-------+
| Max_used_connections | 1     |
+----------------------+-------+
1 row in set (0.00 sec)
```

##### 执行exit退出数据库

-----

## 9.安装消息列表 （控制节点）

#### 导入RabbitMQ签名密钥

```sh
sudo rpm --import https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc
```

#### 添加RabbitMQ仓库

```sh
sudo tee /etc/yum.repos.d/rabbitmq.repo <<'EOF'
[rabbitmq_erlang]
name=rabbitmq_erlang
baseurl=https://packagecloud.io/rabbitmq/erlang/el/7/$basearch
repo_gpgcheck=1
gpgcheck=1
enabled=1
gpgkey=https://packagecloud.io/rabbitmq/erlang/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300

[rabbitmq_server]
name=rabbitmq_server
baseurl=https://packagecloud.io/rabbitmq/rabbitmq-server/el/7/$basearch
repo_gpgcheck=1
gpgcheck=1
enabled=1
gpgkey=https://packagecloud.io/rabbitmq/rabbitmq-server/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
EOF
```

#### 安装rabbitmq-server

```sh
清理仓库缓存
sudo yum clean all
yum -y install rabbitmq-server
```

#### 配置消息队列服务启动和开机

```sh
systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service
```

#### 添加并配置Openstack用户

```sh
# <PASSWORD>是rabbitmq服务为openstack用户设置的密码（密码不要包含字符"#"）
# rabbitmqctl add_user openstack <PASSWORD>
rabbitmqctl add_user openstack 123456
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

#### 启动rabbitmq-manager 插件

```sh
rabbitmq-plugins enable rabbitmq_management
```

启动了插件后，可以在浏览器访问rabbitmq服务。访问地址 http://192.168.200.148:15672，用 户名guest，密码guest

#### 修改rabbitmq默认参数

```sh
vim /usr/lib/systemd/system/rabbitmq-server.service
[Service]
# 添加
LimitNOFILE=16384
```

#### 重启消息队列服务

```sh
systemctl daemon-reload
systemctl restart rabbitmq-server
```

## 10.安装memcached（控制节点）

#### 安装memcached

```sh
yum -y install memcached python-memcached
```

#### 编辑“/etc/sysconfig/memcached”文件

```sh
vim /etc/sysconfig/memcached
PORT="11211"
USER="memcached"
MAXCONN="1024"
CACHESIZE="64"
OPTIONS="-l 127.0.0.1,::1,kz"
```

####  启动memcached服务，并设为开机启动

```sh
systemctl enable memcached.service
systemctl start memcached.service
```

-----

## 11.安装etc(控制节点)

### 安装etcd

```sh
yum -y install etcd
```

####  编辑“/etc/etcd/etcd.conf”文件

```sh
vim /etc/etcd/etcd.conf
#[Member]
#ETCD_CORS=""
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
#ETCD_WAL_DIR=""
ETCD_LISTEN_PEER_URLS="http://172.168.201.11:2380"
ETCD_LISTEN_CLIENT_URLS="http://172.168.201.11:2379"
#ETCD_MAX_SNAPSHOTS="5"
#ETCD_MAX_WALS="5"
ETCD_NAME="kz"
#ETCD_SNAPSHOT_COUNT="100000"
#ETCD_HEARTBEAT_INTERVAL="100"
#ETCD_ELECTION_TIMEOUT="1000"
#ETCD_QUOTA_BACKEND_BYTES="0"
#ETCD_MAX_REQUEST_BYTES="1572864"
#ETCD_GRPC_KEEPALIVE_MIN_TIME="5s"
#ETCD_GRPC_KEEPALIVE_INTERVAL="2h0m0s"
#ETCD_GRPC_KEEPALIVE_TIMEOUT="20s"
#
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://172.168.201.11:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://172.168.201.11:2379"
#ETCD_DISCOVERY=""
#ETCD_DISCOVERY_FALLBACK="proxy"
#ETCD_DISCOVERY_PROXY=""
#ETCD_DISCOVERY_SRV=""
ETCD_INITIAL_CLUSTER="kz=http://172.168.201.11:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER_STATE="new"
#ETCD_STRICT_RECONFIG_CHECK="true"
#ETCD_ENABLE_V2="true"
#
#[Proxy]
#ETCD_PROXY="off"
#ETCD_PROXY_FAILURE_WAIT="5000"
#ETCD_PROXY_REFRESH_INTERVAL="30000"
#ETCD_PROXY_DIAL_TIMEOUT="1000"
#ETCD_PROXY_WRITE_TIMEOUT="5000"
#ETCD_PROXY_READ_TIMEOUT="0"
#
#[Security]
#ETCD_CERT_FILE=""
#ETCD_KEY_FILE=""
#ETCD_CLIENT_CERT_AUTH="false"
#ETCD_TRUSTED_CA_FILE=""
#ETCD_AUTO_TLS="false"
#ETCD_PEER_CERT_FILE=""
#ETCD_PEER_KEY_FILE=""
#ETCD_PEER_CLIENT_CERT_AUTH="false"
#ETCD_PEER_TRUSTED_CA_FILE=""
#ETCD_PEER_AUTO_TLS="false"
#
#[Logging]
#ETCD_DEBUG="false"
#ETCD_LOG_PACKAGE_LEVELS=""
#ETCD_LOG_OUTPUT="default"
#
#[Unsafe]
#ETCD_FORCE_NEW_CLUSTER="false"
#
#[Version]
#ETCD_VERSION="false"
#ETCD_AUTO_COMPACTION_RETENTION="0"
#
#[Profiling]
#ETCD_ENABLE_PPROF="false"
#ETCD_METRICS="basic"
#
#[Auth]
#ETCD_AUTH_TOKEN="simple"
```

注意修改IP地址，所有的地址都是控制节点的管理IP

#### 启动etcd服务

```sh
systemctl enable etcd
systemctl start etcd
```

-----

## 12.关闭SELinux(所有节点)

#### 临时关闭SELinux

```sh
setenforce 0
```

#### 永久关闭SELinux（永久关闭的方法需要重启机器才能生效）

```sh
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

-----

## 13.开放防火墙端(所有节点)

#### 开放TCP端口

```sh
firewall-cmd --zone=public --add-port=8082/tcp --permanent
firewall-cmd --zone=public --add-port=8773-8778/tcp --permanent
firewall-cmd --zone=public --add-port=6080-6082/tcp --permanent
firewall-cmd --zone=public --add-port=8386/tcp --permanent
firewall-cmd --zone=public --add-port=5000/tcp --permanent
firewall-cmd --zone=public --add-port=9292/tcp --permanent
firewall-cmd --zone=public --add-port=9191/tcp --permanent
firewall-cmd --zone=public --add-port=9696/tcp --permanent
firewall-cmd --zone=public --add-port=6000-6002/tcp --permanent
firewall-cmd --zone=public --add-port=6200-6202/tcp --permanent
firewall-cmd --zone=public --add-port=8000-8004/tcp --permanent
firewall-cmd --zone=public --add-port=8999/tcp --permanent
firewall-cmd --zone=public --add-port=8777/tcp --permanent
firewall-cmd --zone=public --add-port=8989/tcp --permanent
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --zone=public --add-port=8080/tcp --permanent
firewall-cmd --zone=public --add-port=443/tcp --permanent
firewall-cmd --zone=public --add-port=873/tcp --permanent
firewall-cmd --zone=public --add-port=3260/tcp --permanent
firewall-cmd --zone=public --add-port=3306/tcp --permanent
firewall-cmd --zone=public --add-port=5672/tcp --permanent
firewall-cmd --zone=public --add-port=6088/tcp --permanent
firewall-cmd --zone=public --add-port=6080/tcp --permanent
firewall-cmd --zone=public --add-port=15672/tcp --permanent
firewall-cmd --zone=public --add-port=323/tcp --permanent
firewall-cmd --zone=public --add-port=11211/tcp --permanent
firewall-cmd --zone=public --add-port=123/tcp --permanent
firewall-cmd --zone=public --add-port=69/tcp --permanent
firewall-cmd --zone=public --add-port=5900-5999/tcp --permanent
firewall-cmd --zone=public --add-port=2379-2380/tcp --permanent
firewall-cmd --zone=public --add-port=6640/tcp --permanent
firewall-cmd --zone=public --add-port=22/tcp --permanent
```

#### 开放UDP端口

```sh
firewall-cmd --zone=public --add-port=6640/udp --permanent
firewall-cmd --zone=public --add-port=8082/udp --permanent
firewall-cmd --zone=public --add-port=8773-8778/udp --permanent
firewall-cmd --zone=public --add-port=6080-6082/udp --permanent
firewall-cmd --zone=public --add-port=8386/udp --permanent
firewall-cmd --zone=public --add-port=5000/udp --permanent
firewall-cmd --zone=public --add-port=9292/udp --permanent
firewall-cmd --zone=public --add-port=9191/udp --permanent
firewall-cmd --zone=public --add-port=9696/udp --permanent
firewall-cmd --zone=public --add-port=6000-6002/udp --permanent
firewall-cmd --zone=public --add-port=6200-6202/udp --permanent
firewall-cmd --zone=public --add-port=8000-8004/udp --permanent
firewall-cmd --zone=public --add-port=8999/udp --permanent
firewall-cmd --zone=public --add-port=8777/udp --permanent
firewall-cmd --zone=public --add-port=8989/udp --permanent
firewall-cmd --zone=public --add-port=80/udp --permanent
firewall-cmd --zone=public --add-port=8080/udp --permanent
firewall-cmd --zone=public --add-port=443/udp --permanent
firewall-cmd --zone=public --add-port=873/udp --permanent
firewall-cmd --zone=public --add-port=3260/udp --permanent
firewall-cmd --zone=public --add-port=3306/udp --permanent
firewall-cmd --zone=public --add-port=5672/udp --permanent
firewall-cmd --zone=public --add-port=6088/udp --permanent
firewall-cmd --zone=public --add-port=6080/udp --permanent
firewall-cmd --zone=public --add-port=15672/udp --permanent
firewall-cmd --zone=public --add-port=323/udp --permanent
firewall-cmd --zone=public --add-port=11211/udp --permanent
firewall-cmd --zone=public --add-port=123/udp --permanent
firewall-cmd --zone=public --add-port=69/udp --permanent
firewall-cmd --zone=public --add-port=5900-5999/udp --permanent
firewall-cmd --zone=public --add-port=2379-2380/udp --permanent
firewall-cmd --zone=public --add-port=22/udp --permanent
```

#### 重新加载生效

```sh
firewall-cmd --reload
```

-----

## 14.安装配置并验证Keystone(控制节点)

#### 进入数据库

```sh
mysql -u root -p
```

#### 创建Keystone数据库

```mysql
CREATE DATABASE keystone;
```

#### 授权，允许本地及远程服务器访问mysql，为数据库用户root的密码

```mysql
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY '123456';
Query OK, 0 rows affected (0.14 sec)

MariaDB [(none)]>  GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY '123456';
Query OK, 0 rows affected (0.00 sec)

```

注意将修改为需要的密码。（openstack的账户密码设置中，不支持特殊符号#； openstack对密码的复杂度没有要求，可以设置为不带特殊字符的密码；若在设置密码时，一定 要包含特殊符号，openstack仅支持如下如下特殊字符：& = $ - _ . + ! * ( ) ）

#### 退出数据库

```mysq
exit
```

-----

## 15.安装Keystone(控制节点)

#### 安装Keystone包

```sh
wget http://vault.centos.org/centos/7/cloud/x86_64/openstack-rocky/Packages/q/qpid-proton-c-0.22.0-1.el7.x86_64.rpm
rpm -Uvh qpid-proton-c-0.22.0-1.el7.x86_64.rpm
rpm -Uvh python2-qpid-proton-0.22.0-1.el7.x86_64.rpm
yum -y install openstack-keystone httpd mod_wsgi
```

#### 编辑“/etc/keystone/keystone.conf”文件

<PASSWORD>为用户为数据库设置的密码

```sh
vim /etc/keystone/keystone.conf
# 在[database]部分添加如下内容：
connection = mysql+pymysql://keystone:123456@kz/keystone
# 在[token]部分添加如下内容：
provider = fernet
```

#### 填充Identity服务数据库

```sh
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

#### 初始化Fernet密钥存储库

```sh
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

#### 引导身份服务

注意修改为用户admin的密码。123456

```sh
keystone-manage bootstrap \
  --bootstrap-password 123456 \
  --bootstrap-admin-url http://kz:5000/v3/ \
  --bootstrap-internal-url http://kz:5000/v3/ \
  --bootstrap-public-url http://kz:5000/v3/ \
  --bootstrap-region-id RegionOne
```

## 16.配置Apache HTTP 服务(控制节点)

#### 配置ServerName选项为控制节点

```sh
vim /etc/httpd/conf/httpd.conf

# ServerName gives the name and port that the server uses to identify itself.
# This can often be determined automatically, but we recommend you specify
# it explicitly to prevent problems during startup.
#
# If your host doesn't have a registered DNS name, enter its IP address here.
#
ServerName kz
```

####   创建“/usr/share/keystone/wsgi-keystone.conf”文件的链接

```sh
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```

####  启动 Apache HTTP 服务并配置其随系统启动

```sh
systemctl enable httpd.service
systemctl start httpd.service
```

####  配置管理帐户

注意修改为用户admin的密码

```sh
export OS_USERNAME=admin
export OS_PASSWORD=123456
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://kz:5000/v3
export OS_IDENTITY_API_VERSION=3
```

## 17.创建域、项目、用户和角色(控制节点)

#### 创建新域

```sh
openstack domain create --description "An Example Domain" example
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | An Example Domain                |
| enabled     | True                             |
| id          | 1090cd74994e436389ad6cc49d4ed534 |
| name        | example                          |
| tags        | []                               |
+-------------+----------------------------------+
```

