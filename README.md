## OpenStack-Install
## 一、安装环境
## 1、配置hosts文件，配置好网卡(all nodes)
 HOST       |    IP
 ----------|----------
 controller |192.168.100.10 #192.168.200.10
 compute    |192.168.100.20 #192.168.200.20
## 2、关闭防火墙、SELinux（略）
 
## 3、安装NTP服务
 yum install chrony -y
 controller控制节点编辑/etc/chrony.conf
 allow 192.168.100.0/24
 systemctl enable chronyd.service && systemctl start chronyd.service
 其他节点节点编辑/etc/chrony.conf
 server controller iburst
 systemctl enable chronyd.service && systemctl start chronyd.service
## 4、准备OpenStack安装包(all nodes)
 yum install centos-release-openstack-queens -y && yum upgrade -y && yum install python-openstackclient openstack-selinux openstack-utils

## 5、安装Database服务（MySQL）（controller）
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

## 6、安装消息队列服务（controller）
 yum install rabbitmq-server -y
 systemctl enable rabbitmq-server.service && systemctl start rabbitmq-server.service
 rabbitmqctl add_user openstack 123456
 rabbitmqctl set_permissions openstack ".*" ".*" ".*"

## 7、安装缓存服务（controller）
 yum install memcached python-memcached -y
 sed -i 's/OPTIONS="-l 127.0.0.1,::1"/OPTIONS="-l 127.0.0.1,::1,controller"/g' /etc/sysconfig/memcached 
 systemctl enable memcached.service && systemctl start memcached.service

## 8、安装Etcd数据库（controller）
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
 
# 二、Mini最小安装
## 1、认证服务（Identity service）
 mysql -e "CREATE DATABASE keystone;"
 mysql -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY '123456'"
 mysql -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY '123456'"
 yum install openstack-keystone httpd mod_wsgi -y
 cp /etc/keystone/keystone.conf{,.bak}
 crudini --set /etc/keystone/keystone.conf database connection  mysql+pymysql://keystone:123456@controller/keystone
 crudini --set /etc/keystone/keystone.conf token provider  fernet
 su -s /bin/sh -c "keystone-manage db_sync" keystone
#初始化Fernet key库
 keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
 keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
 #Bootstrap the Identity service:
 keystone-manage bootstrap --bootstrap-password 123456 \
   --bootstrap-admin-url http://controller:5000/v3/ \
   --bootstrap-internal-url http://controller:5000/v3/ \
   --bootstrap-public-url http://controller:5000/v3/ \
   --bootstrap-region-id RegionOne
 sed -i "s/#ServerName www.example.com:80/ServerName controller/g" /etc/httpd/conf/httpd.conf 
 ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
 systemctl enable httpd.service && systemctl start httpd.service
 cat <<EOF > ./adminrc.sh
 export OS_USERNAME=admin
 export OS_PASSWORD=123456
 export OS_PROJECT_NAME=admin
 export OS_USER_DOMAIN_NAME=Default
 export OS_PROJECT_DOMAIN_NAME=Default
 export OS_AUTH_URL=http://controller:35357/v3
 export OS_IDENTITY_API_VERSION=3
 EOF
 . adminrc.sh
 openstack domain create --description "An Example Domain" example
 openstack project create --domain default --description "Service Project" service
 openstack project create --domain default --description "Demo Project" demo
 openstack user create --domain default --password-prompt demo
 openstack role create user
 openstack role add --project demo --user demo user
 unset OS_AUTH_URL OS_PASSWORD
 openstack --os-auth-url http://controller:35357/v3 \
   --os-project-domain-name Default --os-user-domain-name Default \
   --os-project-name admin --os-username admin token issue
   输入密码123456
 openstack --os-auth-url http://controller:5000/v3 \
   --os-project-domain-name Default --os-user-domain-name Default \
   --os-project-name demo --os-username demo token issue
   输入密码123456
#配置admin认证脚本避免每次加--os选项
 cat <<EOF > admin-openrc.sh
 export OS_PROJECT_DOMAIN_NAME=Default
 export OS_USER_DOMAIN_NAME=Default
 export OS_PROJECT_NAME=admin
 export OS_USERNAME=admin
 export OS_PASSWORD=123456
 export OS_AUTH_URL=http://controller:5000/v3
 export OS_IDENTITY_API_VERSION=3
 export OS_IMAGE_API_VERSION=2
 EOF
 cat <<EOF > demon-openrc.sh
 export OS_PROJECT_DOMAIN_NAME=Default
 export OS_USER_DOMAIN_NAME=Default
 export OS_PROJECT_NAME=demo
 export OS_USERNAME=demo
 export OS_PASSWORD=123456
 export OS_AUTH_URL=http://controller:5000/v3
 export OS_IDENTITY_API_VERSION=3
 export OS_IMAGE_API_VERSION=2
 . admin-openrc
 openstack token issue
## 2、镜像服务（Image service）
 mysql -e "CREATE DATABASE glance;"
 mysql -e "GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY '123456'";
 mysql -e "GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY '123456'";
 openstack user create --domain default --password-prompt glance
 输入密码123456
 openstack role add --project service --user glance admin
 openstack service create --name glance --description "OpenStack Image" image
 openstack endpoint create --region RegionOne image public http://controller:9292
 openstack endpoint create --region RegionOne image internal http://controller:9292
 openstack endpoint create --region RegionOne image admin http://controller:9292
 yum install openstack-glance -y
 cp /etc/glance/glance-api.conf{,.bak}
 crudini --set /etc/glance/glance-api.conf database connection  mysql+pymysql://glance:123456@controller/glance
 crudini --set /etc/glance/glance-api.conf keystone_authtoken auth_uri  http://controller:5000
 crudini --set /etc/glance/glance-api.conf keystone_authtoken auth_url  http://controller:5000
 crudini --set /etc/glance/glance-api.conf keystone_authtoken memcached_servers  controller:11211
 crudini --set /etc/glance/glance-api.conf keystone_authtoken auth_type  password
 crudini --set /etc/glance/glance-api.conf keystone_authtoken project_domain_name  Default
 crudini --set /etc/glance/glance-api.conf keystone_authtoken user_domain_name  Default
 crudini --set /etc/glance/glance-api.conf keystone_authtoken project_name service
 crudini --set /etc/glance/glance-api.conf keystone_authtoken username glance
 crudini --set /etc/glance/glance-api.conf keystone_authtoken password 123456
 crudini --set /etc/glance/glance-api.conf paste_deploy flavor keystone
 crudini --set /etc/glance/glance-api.conf glance_store stores file,http
 crudini --set /etc/glance/glance-api.conf glance_store default_store file
 crudini --set /etc/glance/glance-api.conf glance_store filesystem_store_datadir /var/lib/glance/images/
 
 #Edit /etc/glance/glance-registry.conf
 crudini --set /etc/glance/glance-registry.conf database connection mysql+pymysql://glance:123456@controller/glance
 crudini --set /etc/glance/glance-registry.conf keystone_authtoken auth_uri  http://controller:5000
 crudini --set /etc/glance/glance-registry.conf keystone_authtoken auth_url  http://controller:5000
 crudini --set /etc/glance/glance-registry.conf keystone_authtoken memcached_servers  controller:11211
 crudini --set /etc/glance/glance-registry.conf keystone_authtoken auth_type  password
 crudini --set /etc/glance/glance-registry.conf keystone_authtoken project_domain_name  Default
 crudini --set /etc/glance/glance-registry.conf keystone_authtoken user_domain_name  Default
 crudini --set /etc/glance/glance-registry.conf keystone_authtoken project_name service
 crudini --set /etc/glance/glance-registry.conf keystone_authtoken username glance
 crudini --set /etc/glance/glance-registry.conf keystone_authtoken password 123456
 crudini --set /etc/glance/glance-registry.conf paste_deploy flavor keystone
 
 su -s /bin/sh -c "glance-manage db_sync" glance
 systemctl enable openstack-glance-api.service openstack-glance-registry.service
 systemctl start openstack-glance-api.service openstack-glance-registry.service
## 3、计算服务（Compute service）
#### 先配置controller
 mysql -e "CREATE DATABASE nova_api;"
 mysql -e "CREATE DATABASE nova;"
 mysql -e "CREATE DATABASE nova_cell0;"
 mysql -e "GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY '123456';"
 mysql -e "GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY '123456';"
 mysql -e "GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY '123456';"
 mysql -e "GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY '123456';"
 mysql -e "GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY '123456';"
 mysql -e "GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY '123456';"
 openstack user create --domain default --password-prompt nova
  输入密码123456
 openstack role add --project service --user nova admin
 openstack service create --name nova --description "OpenStack Compute" compute
 openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
 openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
 openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
 openstack user create --domain default --password-prompt placement
  输入密码123456
 openstack role add --project service --user placement admin
 openstack service create --name placement --description "Placement API" placement
 openstack endpoint create --region RegionOne placement public http://controller:8778
 openstack endpoint create --region RegionOne placement internal http://controller:8778
 openstack endpoint create --region RegionOne placement admin http://controller:8778
 yum install openstack-nova-api openstack-nova-conductor \
  openstack-nova-console openstack-nova-novncproxy \
  openstack-nova-scheduler openstack-nova-placement-api
 #修改NOVA配置文件
 crudini --set /etc/nova/nova.conf DEFAULT enabled_apis osapi_compute,metadata
 crudini --set /etc/nova/nova.conf api_database connection mysql+pymysql://nova:123456@controller/nova_api
 crudini --set /etc/nova/nova.conf database connection mysql+pymysql://nova:123456@controller/nova
 crudini --set /etc/nova/nova.conf DEFAULT transport_url rabbit://openstack:RABBIT_PASS@controller
 crudini --set /etc/nova/nova.conf api auth_strategy keystone
 crudini --set /etc/nova/nova.conf keystone_authtoken auth_url http://controller:5000/v3
 crudini --set /etc/nova/nova.conf keystone_authtoken memcached_servers controller:11211
 crudini --set /etc/nova/nova.conf keystone_authtoken auth_type password
 crudini --set /etc/nova/nova.conf keystone_authtoken project_domain_name default
 crudini --set /etc/nova/nova.conf keystone_authtoken euser_domain_name default
 crudini --set /etc/nova/nova.conf keystone_authtoken project_name service
 crudini --set /etc/nova/nova.conf keystone_authtoken username nova
 crudini --set /etc/nova/nova.conf keystone_authtoken password 123456
 crudini --set /etc/nova/nova.conf DEFAULT my_ip 192.168.100.10
 crudini --set /etc/nova/nova.conf use_neutron True
 crudini --set /etc/nova/nova.conf firewall_driver nova.virt.firewall.NoopFirewallDriver
 crudini --set /etc/nova/nova.conf vnc enabled true
 crudini --set /etc/nova/nova.conf vnc server_listen $my_ip
 crudini --set /etc/nova/nova.conf vnc server_proxyclient_address $my_ip
 crudini --set /etc/nova/nova.conf glance api_servers http://controller:9292
 crudini --set /etc/nova/nova.conf oslo_concurrency lock_path /var/lib/nova/tmp
 crudini --set /etc/nova/nova.conf placement os_region_name RegionOne
 crudini --set /etc/nova/nova.conf placement project_domain_name Default
 crudini --set /etc/nova/nova.conf placement project_name service
 crudini --set /etc/nova/nova.conf placement auth_type password
 crudini --set /etc/nova/nova.conf placement user_domain_name Default
 crudini --set /etc/nova/nova.conf placement auth_url http://controller:5000/v3
 crudini --set /etc/nova/nova.conf placement username placement
 crudini --set /etc/nova/nova.conf placement password 123456

cat>>/etc/httpd/conf.d/00-nova-placement-api.conf<<EOF

<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>
EOF
systemctl restart httpd
#同步数据库报错，好像不影响，数据表都在
#/usr/lib/python2.7/site-packages/oslo_db/sqlalchemy/enginefacade.py:332: NotSupportedWarning: Configuration option(s) ['use_tpool'] not #supported
#exception.NotSupportedWarning
#如果说看着不爽，可以到源码中将这段判断注释掉/usr/lib/python2.7/site-packages/oslo_db/sqlalchemy/enginefacade.py的325:333行
 su -s /bin/sh -c "nova-manage api_db sync" nova
 su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
 su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
 su -s /bin/sh -c "nova-manage db sync" nova
 nova-manage cell_v2 list_cells
 systemctl enable openstack-nova-api.service \
   openstack-nova-consoleauth.service openstack-nova-scheduler.service \
   openstack-nova-conductor.service openstack-nova-novncproxy.service
 systemctl start openstack-nova-api.service \
   openstack-nova-consoleauth.service openstack-nova-scheduler.service \
   openstack-nova-conductor.service openstack-nova-novncproxy.service
 
 #配置（compute）
 yum install openstack-nova-compute -y
 crudini --set /etc/nova/nova.conf DEFAULT enabled_apis osapi_compute,metadata
 crudini --set /etc/nova/nova.conf DEFAULT transport_url rabbit://openstack:123456@controller
 crudini --set /etc/nova/nova.conf api auth_strategy keystone
 crudini --set /etc/nova/nova.conf keystone_authtoken auth_url http://controller:5000/v3
 crudini --set /etc/nova/nova.conf keystone_authtoken memcached_servers controller:11211
 crudini --set /etc/nova/nova.conf keystone_authtoken auth_type password
 crudini --set /etc/nova/nova.conf keystone_authtoken project_domain_name default
 crudini --set /etc/nova/nova.conf keystone_authtoken user_domain_name default
 crudini --set /etc/nova/nova.conf keystone_authtoken project_name service
 crudini --set /etc/nova/nova.conf keystone_authtoken enabled_apis username nova
 crudini --set /etc/nova/nova.conf keystone_authtoken password 123456
 crudini --set /etc/nova/nova.conf DEFAULT my_ip = 192.168.100.20
 crudini --set /etc/nova/nova.conf DEFAULT use_neutron True
 crudini --set /etc/nova/nova.conf DEFAULT firewall_driver nova.virt.firewall.NoopFirewallDriver
 crudini --set /etc/nova/nova.conf vnc enabled True
 crudini --set /etc/nova/nova.conf vnc server_listen 0.0.0.0
 crudini --set /etc/nova/nova.conf vnc server_proxyclient_address $my_ip
 crudini --set /etc/nova/nova.conf vnc novncproxy_base_url http://controller:6080/vnc_auto.html
 crudini --set /etc/nova/nova.conf glance api_servers http://controller:9292
 crudini --set /etc/nova/nova.conf oslo_concurrency lock_path = /var/lib/nova/tmp 
 crudini --set /etc/nova/nova.conf placement os_region_name RegionOne
 crudini --set /etc/nova/nova.conf placement project_domain_name Default
 crudini --set /etc/nova/nova.conf placement project_name service
 crudini --set /etc/nova/nova.conf placement auth_type password
 crudini --set /etc/nova/nova.conf placement user_domain_name Default
 crudini --set /etc/nova/nova.conf placement auth_url http://controller:5000/v3
 crudini --set /etc/nova/nova.conf placement username placement
 crudini --set /etc/nova/nova.conf placement password 123456
 [[ `egrep -c '(vmx|svm)' /proc/cpuinfo` == 0 ]]  && crudini --set /etc/nova/nova.conf libvirt virt_type qemu
 crudini --set /etc/nova/nova.conf scheduler discover_hosts_in_cells_interval 300
 systemctl enable libvirtd.service openstack-nova-compute.service && systemctl start libvirtd.service openstack-nova-compute.service
 
 . admin-openrc
 openstack compute service list --service nova-compute
 su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
## 4、网络服务（Network service）
### 配置controller
5、Horizon（Dashboard）

6、块存储服务（Block Storage service）
参考官方文档/附地址：https://docs.openstack.org/install-guide
