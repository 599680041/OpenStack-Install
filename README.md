# OpenStack-Install
一、安装环境
1、配置hosts文件，配置好网卡(all nodes)
controller 192.168.100.10 #192.168.200.10
compute    192.168.100.20 #192.168.200.20
2、关闭防火墙、SELinux（略）

3、安装NTP服务

yum install chrony -y
controller控制节点编辑/etc/chrony.conf
allow 192.168.100.0/24
systemctl enable chronyd.service && systemctl start chronyd.service
其他节点节点编辑/etc/chrony.conf
server controller iburst
systemctl enable chronyd.service && systemctl start chronyd.service
4、准备OpenStack安装包(all nodes)
yum install centos-release-openstack-queens -y && yum upgrade -y && yum install python-openstackclient openstack-selinux openstack-utils

5、安装Database服务（MySQL）（controller）

yum install mariadb mariadb-server python2-PyMySQL -y
cat <<EOF > /etc/my.cnf.d/openstack.cnf
[mysqld]
bind-address = 192.168.100.10
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
EOF
systemctl enable mariadb.service && systemctl start mariadb.service
mysql_secure_installation

6、安装消息队列服务（controller）
yum install rabbitmq-server -y
systemctl enable rabbitmq-server.service && systemctl start rabbitmq-server.service
rabbitmqctl add_user openstack 123456
rabbitmqctl set_permissions openstack ".*" ".*" ".*"

7、安装缓存服务（controller）
yum install memcached python-memcached -y
sed -i 's/OPTIONS="-l 127.0.0.1,::1"/OPTIONS="-l 127.0.0.1,::1,controller"/g' /etc/sysconfig/memcached 
systemctl enable memcached.service && systemctl start memcached.service

8、安装Etcd数据库（controller）
yum install etcd -y
cat <<EOF >/etc/etcd/etcd.conf
#[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.0.0.11:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.0.0.11:2379"
ETCD_NAME="controller"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.0.0.11:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.11:2379"
ETCD_INITIAL_CLUSTER="controller=http://10.0.0.11:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
systemctl enable etcd && systemctl start etcd
 
二、Mini最小安装
1、认证服务（Identity service）
mysql -e "CREATE DATABASE keystone;"
mysql -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY '123456'"
mysql -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY '123456'"
yum install openstack-keystone httpd mod_wsgi -y
cp /etc/keystone/keystone.conf{,.bak}
sed -i 's/\[database\]/a\connection = mysql+pymysql://keystone:123456@controller/keystone /etc/keystone/keystone.conf
sed -i 's/\[token\]/a\provider = fernet /etc/keystone/keystone.conf
su -s /bin/sh -c "keystone-manage db_sync" keystone
# 初始化Fernet key库
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
# Bootstrap the Identity service:

2、镜像服务（Image service）

3、计算服务（Compute service）

4、网络服务（Network service）

5、Horizon（Dashboard）

6、块存储服务（Block Storage service）
参考官方文档/附地址：https://docs.openstack.org/install-guide
