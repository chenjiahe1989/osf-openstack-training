#OpenStack 手动安装手册（Icehouse）

##部署架构

为了更好的展现OpenStack各组件分布式部署的特点，以及逻辑网络配置的区别，本实验不采用All in One 的部署模式，而是采用多节点分开部署的方式，方便后续学习研究。

![architecture](/installation/images/architecture.png)

##网络拓扑

![networking](/installation/images/networking.png)

##环境准备

本实验采用Virtualbox Windows 版作为虚拟化平台，模拟相应的物理网络和物理服务器，如果需要部署到真实的物理环境，此步骤可以直接替换为在物理机上相应的配置，其原理相同。


Virtualbox 下载地址：https://www.virtualbox.org/wiki/Downloads

###虚拟网络

需要新建3个虚拟网络Net0、Net1和Net2，其在virtual box 中对应配置如下。

	Net0:
		Network name: VirtualBox  host-only Ethernet Adapter#2
		Purpose: administrator / management network
		IP block: 10.20.0.0/24
		DHCP: disable
		Linux device: eth0

	Net1:
		Network name: VirtualBox  host-only Ethernet Adapter#3
		Purpose: public network
		DHCP: disable
		IP block: 172.16.0.0/24
		Linux device: eth1

	Net2：
		Network name: VirtualBox  host-only Ethernet Adapter#4
		Purpose: Storage/private network
		DHCP: disable
		IP block: 192.168.4.0/24
		Linux device: eth2

###虚拟机

需要新建3个虚拟机VM0、VM1和VM2，其对应配置如下。

	VM0：
		Name: controller0
		vCPU:1
		Memory :1G
		Disk:30G
		Networks: net1

	VM1：
		Name : network0
		vCPU:1
		Memory :1G
		Disk:30G
		Network:net1,net2,net3

	VM2：
		Name: compute0
		vCPU:2
		Memory :2G
		Disk:30G
		Networks:net1,net3


###网络设置

	controller0 
	     eth0:10.20.0.10   (management network)
	     eht1:(disabled)
	     eht2:(disabled)

	network0
	     eth0:10.20.0.20    (management network)
	     eht1:172.16.0.20   (public/external network)
	     eht2:192.168.4.20  (private network)

	compute0
	     eth0:10.20.0.30   (management network)
	     eht1:(disabled)
	     eht2:192.168.4.30  (private network)

	compute1  (optional)
	     eth0:10.20.0.31   (management network)
	     eht1:(disabled)
	     eht2:192.168.4.31  (private network)

###操作系统准备

本实验使用Linux 发行版 CentOS 6.5 x86_64，在安装操作系统过程中，选择的初始安装包为“基本”安装包，安装完成系统以后还需要额外配置如下YUM 仓库。

ISO文件下载：http://mirrors.163.com/centos/6.5/isos/x86_64/CentOS-6.5-x86_64-bin-DVD1.iso 

EPEL源: http://dl.fedoraproject.org/pub/epel/6/x86_64/

RDO源:  http://repos.fedorapeople.org/repos/openstack/openstack-icehouse/

自动配置执行如此命令即可，源安装完成后更新所有RPM包，由于升级了kernel 需要重新启动操作系统。

	yum install -y http://repos.fedorapeople.org/repos/openstack/openstack-icehouse/rdo-release-icehouse-4.noarch.rpm
	yum install -y http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
	yum update -y
	reboot -h 0

接下来可以开始安装配置啦！

###公共配置（all nodes）

以下命令需要在每一个节点都执行。

修改hosts 文件

	vi /etc/hosts

	127.0.0.1    localhost
	::1          localhost 
	10.20.0.10   controller0 
	10.20.0.20   network0
	10.20.0.30   compute0


禁用 selinux 

	vi /etc/selinux/config
	SELINUX=disabled


安装NTP 服务

	yum install ntp -y
	service ntpd start
	chkconfig ntpd on

修改NTP配置文件，配置从controller0时间同步。(除了controller0以外)


	vi /etc/ntp.conf

	server 10.20.0.10
	fudge  192.168.0.10 stratum 10  # LCL is unsynchronized


立即同步并检查时间同步配置是否正确。(除了controller0以外)

	ntpdate -u 10.20.0.10
	service ntpd restart
	ntpq -p

清空防火墙规则

	vi /etc/sysconfig/iptables
	*filter
	:INPUT ACCEPT [0:0]
	:FORWARD ACCEPT [0:0]
	:OUTPUT ACCEPT [0:0]
	COMMIT

重启防火墙，查看是否生效

	service iptables restart
	iptables -L

安装openstack-utils,方便后续直接可以通过命令行方式修改配置文件


	yum install -y openstack-utils


###基本服务安装与配置（controller0 node）

基本服务包括NTP 服务、MySQL数据库服务和AMQP服务，本实例采用MySQL 和Qpid 作为这两个服务的实现。


修改NTP配置文件，配置从127.127.1.0 时间同步。

	vi /etc/ntp.conf
	server 127.127.1.0

重启ntp service

	service ntpd restart

MySQL 服务安装

	yum install -y mysql mysql-server MySQL-python

修改MySQL配置

	vi /etc/my.cnf
	[mysqld]
	bind-address = 0.0.0.0
	default-storage-engine = innodb
	innodb_file_per_table
	collation-server = utf8_general_ci
	init-connect = 'SET NAMES utf8'
	character-set-server = utf8

启动MySQL服务

	service mysqld start
	chkconfig mysqld on

交互式配置MySQL root 密码，设置密码为“openstack”

	mysql_secure_installation


Qpid 安装消息服务，设置客户端不需要验证使用服务

	yum install -y qpid-cpp-server

	vi /etc/qpidd.conf
	auth=no


配置修改后，重启Qpid后台服务

	service qpidd start
	chkconfig qpidd on


##控制节点安装（controller0）

主机名设置

	vi /etc/sysconfig/network
	HOSTNAME=controller0

网卡配置

	vi /etc/sysconfig/network-scripts/ifcfg-eth0

	DEVICE=eth0
	TYPE=Ethernet
	ONBOOT=yes
	NM_CONTROLLED=yes
	BOOTPROTO=static
	IPADDR=10.20.0.10
	NETMASK=255.255.255.0


网络配置文件修改完后重启网络服务

	serice network restart

###Keyston 安装与配置

安装keystone 包

	yum install openstack-keystone python-keystoneclient -y

为keystone 设置admin 账户的 tokn


	ADMIN_TOKEN=$(openssl rand -hex 10)
	echo $ADMIN_TOKEN
	openstack-config --set /etc/keystone/keystone.conf DEFAULT admin_token $ADMIN_TOKEN


配置数据连接


	openstack-config --set /etc/keystone/keystone.conf sql connection mysql://keystone:openstack@controller0/keystone


设置Keystone 用 PKI tokens 


	keystone-manage pki_setup --keystone-user keystone --keystone-group keystone
	chown -R keystone:keystone /etc/keystone/ssl
	chmod -R o-rwx /etc/keystone/ssl


为Keystone 建表

	mysql -uroot -popenstack -e "CREATE DATABASE keystone;"
	mysql -uroot -popenstack -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'openstack';"
	mysql -uroot -popenstack -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'controller0' IDENTIFIED BY 'openstack';"
	mysql -uroot -popenstack -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'openstack';"

初始化Keystone数据库

	su -s /bin/sh -c "keystone-manage db_sync" 

也可以直接用openstack-db 工具初始数据库

	openstack-db --init --service keystone --password openstack

启动keystone 服务

	service openstack-keystone start
	chkconfig openstack-keystone on

设置认证信息

	export OS_SERVICE_TOKEN=`echo $ADMIN_TOKEN`
	export OS_SERVICE_ENDPOINT=http://controller0:35357/v2.0


创建管理员和系统服务使用的租户

	keystone tenant-create --name=admin --description="Admin Tenant"
	keystone tenant-create --name=service --description="Service Tenant"


创建管理员用户

	keystone user-create --name=admin --pass=admin --email=admin@example.com

创建管理员角色


	keystone role-create --name=admin


为管理员用户分配"管理员"角色


	keystone user-role-add --user=admin --tenant=admin --role=admin


为keystone 服务建立 endpoints


	keystone service-create --name=keystone --type=identity --description="Keystone Identity Service"


为keystone 建立 servie 和 endpoint 关联


	keystone endpoint-create \
	--service-id=$(keystone service-list | awk '/ identity / {print $2}') \
	--publicurl=http://controller0:5000/v2.0 \
	--internalurl=http://controller0:5000/v2.0 \
	--adminurl=http://controller0:35357/v2.0


验证keystone 安装的正确性

取消先前的Token变量，不然会干扰新建用户的验证。

	unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT

先用命令行方式验证

	keystone --os-username=admin --os-password=admin --os-auth-url=http://controller0:35357/v2.0 token-get
	keystone --os-username=admin --os-password=admin --os-tenant-name=admin --os-auth-url=http://controller0:35357/v2.0 token-get


让后用设置环境变量认证,保存认证信息

	vi ~/keystonerc

	export OS_USERNAME=admin
	export OS_PASSWORD=admin
	export OS_TENANT_NAME=admin
	export OS_AUTH_URL=http://controller0:35357/v2.0


source 该文件使其生效

	source keystonerc
	keystone token-get


Keystone 安装结束。

###Glance 安装与配置 

安装Glance 的包

	yum install openstack-glance python-glanceclient -y

配置Glance 连接数据库

	openstack-config --set /etc/glance/glance-api.conf DEFAULT sql_connection mysql://glance:openstack@controller0/glance
	openstack-config --set /etc/glance/glance-registry.conf DEFAULT sql_connection mysql://glance:openstack@controller0/glance

初始化Glance数据库

	openstack-db --init --service glance --password openstack

创建glance 用户


	keystone user-create --name=glance --pass=glance --email=glance@example.com


并分配service角色

	keystone user-role-add --user=glance --tenant=service --role=admin

创建glance 服务

	keystone service-create --name=glance --type=image --description="Glance Image Service"


创建keystone 的endpoint 

	keystone endpoint-create \
	--service-id=$(keystone service-list | awk '/ image / {print $2}')  \
	--publicurl=http://controller0:9292 \
	--internalurl=http://controller0:9292 \
	--adminurl=http://controller0:9292


用openstack util 修改glance api 和 register 配置文件


	openstack-config --set /etc/glance/glance-api.conf keystone_authtoken auth_uri http://controller0:5000
	openstack-config --set /etc/glance/glance-api.conf keystone_authtoken auth_host controller0
	openstack-config --set /etc/glance/glance-api.conf keystone_authtoken auth_port 35357
	openstack-config --set /etc/glance/glance-api.conf keystone_authtoken auth_protocol http
	openstack-config --set /etc/glance/glance-api.conf keystone_authtoken admin_tenant_name service
	openstack-config --set /etc/glance/glance-api.conf keystone_authtoken admin_user glance
	openstack-config --set /etc/glance/glance-api.conf keystone_authtoken admin_password glance
	openstack-config --set /etc/glance/glance-api.conf paste_deploy flavor keystone

	openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken auth_uri http://controller0:5000
	openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken auth_host controller0
	openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken auth_port 35357
	openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken auth_protocol http
	openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken admin_tenant_name service
	openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken admin_user glance
	openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken admin_password glance
	openstack-config --set /etc/glance/glance-registry.conf paste_deploy flavor keystone



启动glance 相关的两个服务

	service openstack-glance-api start
	service openstack-glance-registry start
	chkconfig openstack-glance-api on
	chkconfig openstack-glance-registry on


下载最Cirros镜像验证glance 安装是否成功

	wget http://cdn.download.cirros-cloud.net/0.3.1/cirros-0.3.1-x86_64-disk.img
	glance image-create --progress --name="CirrOS 0.3.1" --disk-format=qcow2  --container-format=ovf --is-public=true < cirros-0.3.1-x86_64-disk.img



查看刚刚上传的image 

	glance  image-list

如果显示相应的image 信息说明安装成功。


###Nova 安装与配置

	yum install -y openstack-nova-api openstack-nova-cert openstack-nova-conductor \
	openstack-nova-console openstack-nova-novncproxy openstack-nova-scheduler python-novaclient

在keystone中创建nova相应的用户和服务

	keystone user-create --name=nova --pass=nova --email=nova@example.com
	keystone user-role-add --user=nova --tenant=service --role=admin

keystone 注册服务

	keystone service-create --name=nova --type=compute --description="Nova Compute Service"

keystone 注册endpoint

	keystone endpoint-create \
	--service-id=$(keystone service-list | awk '/ compute / {print $2}')  \
	--publicurl=http://controller0:8774/v2/%\(tenant_id\)s \
	--internalurl=http://controller0:8774/v2/%\(tenant_id\)s \
	--adminurl=http://controller0:8774/v2/%\(tenant_id\)s

配置nova MySQL 连接

	openstack-config --set /etc/nova/nova.conf database connection mysql://nova:openstack@controller0/nova

初始化数据库

	openstack-db --init --service nova --password openstack

配置nova.conf

	openstack-config --set /etc/nova/nova.conf DEFAULT rpc_backend qpid
	openstack-config --set /etc/nova/nova.conf DEFAULT qpid_hostname controller0

	openstack-config --set /etc/nova/nova.conf DEFAULT my_ip 10.20.0.10
	openstack-config --set /etc/nova/nova.conf DEFAULT vncserver_listen 10.20.0.10
	openstack-config --set /etc/nova/nova.conf DEFAULT vncserver_proxyclient_address 10.20.0.10

	openstack-config --set /etc/nova/nova.conf DEFAULT auth_strategy keystone
	openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_uri http://controller0:5000
	openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_host controller0
	openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_protocol http
	openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_port 35357
	openstack-config --set /etc/nova/nova.conf keystone_authtoken admin_user nova
	openstack-config --set /etc/nova/nova.conf keystone_authtoken admin_tenant_name service
	openstack-config --set /etc/nova/nova.conf keystone_authtoken admin_password nova



添加api-paste.ini 的 Keystone认证信息

	openstack-config --set /etc/nova/api-paste.ini filter:authtoken paste.filter_factory keystoneclient.middleware.auth_token:filter_factory
	openstack-config --set /etc/nova/api-paste.ini filter:authtoken auth_host controller0
	openstack-config --set /etc/nova/api-paste.ini filter:authtoken admin_tenant_name service
	openstack-config --set /etc/nova/api-paste.ini filter:authtoken admin_user nova
	openstack-config --set /etc/nova/api-paste.ini filter:authtoken admin_password nova

启动服务

	service openstack-nova-api start
	service openstack-nova-cert start
	service openstack-nova-consoleauth start
	service openstack-nova-scheduler start
	service openstack-nova-conductor start
	service openstack-nova-novncproxy start

添加到系统服务

	chkconfig openstack-nova-api on
	chkconfig openstack-nova-cert on
	chkconfig openstack-nova-consoleauth on
	chkconfig openstack-nova-scheduler on
	chkconfig openstack-nova-conductor on
	chkconfig openstack-nova-novncproxy on


检查服务是否正常

	nova-manage service list

	root@controller0 ~]# nova-manage service list
	Binary           Host                                 Zone             Status     State Updated_At
	nova-consoleauth controller0                          internal         enabled    :-)   2013-11-12 11:14:56
	nova-cert        controller0                          internal         enabled    :-)   2013-11-12 11:14:56
	nova-scheduler   controller0                          internal         enabled    :-)   2013-11-12 11:14:56
	nova-conductor   controller0                          internal         enabled    :-)   2013-11-12 11:14:56

检查进程

	[root@controller0 ~]# ps -ef|grep nova
	nova      7240     1  1 23:11 ?        00:00:02 /usr/bin/python /usr/bin/nova-api --logfile /var/log/nova/api.log
	nova      7252     1  1 23:11 ?        00:00:01 /usr/bin/python /usr/bin/nova-cert --logfile /var/log/nova/cert.log
	nova      7264     1  1 23:11 ?        00:00:01 /usr/bin/python /usr/bin/nova-consoleauth --logfile /var/log/nova/consoleauth.log
	nova      7276     1  1 23:11 ?        00:00:01 /usr/bin/python /usr/bin/nova-scheduler --logfile /var/log/nova/scheduler.log
	nova      7288     1  1 23:11 ?        00:00:01 /usr/bin/python /usr/bin/nova-conductor --logfile /var/log/nova/conductor.log
	nova      7300     1  0 23:11 ?        00:00:00 /usr/bin/python /usr/bin/nova-novncproxy --web /usr/share/novnc/
	nova      7336  7240  0 23:11 ?        00:00:00 /usr/bin/python /usr/bin/nova-api --logfile /var/log/nova/api.log
	nova      7351  7240  0 23:11 ?        00:00:00 /usr/bin/python /usr/bin/nova-api --logfile /var/log/nova/api.log
	nova      7352  7240  0 23:11 ?        00:00:00 /usr/bin/python /usr/bin/nova-api --logfile /var/log/nova/api.log


###Neutron server安装与配置

安装Neutron server 相关包

	yum install -y openstack-neutron openstack-neutron-ml2 python-neutronclient

在keystone中创建 Neutron 相应的用户和服务

	keystone user-create --name neutron --pass neutron --email neutron@example.com

	keystone user-role-add --user neutron --tenant service --role admin

	keystone service-create --name neutron --type network --description "OpenStack Networking"

	keystone endpoint-create \
	--service-id $(keystone service-list | awk '/ network / {print $2}') \
	--publicurl http://controller0:9696 \
	--adminurl http://controller0:9696 \
	--internalurl http://controller0:9696

为Neutron 在MySQL建数据库

	mysql -uroot -popenstack -e "CREATE DATABASE neutron;"
	mysql -uroot -popenstack -e "GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'openstack';"
	mysql -uroot -popenstack -e "GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'openstack';"
	mysql -uroot -popenstack -e "GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'controller0' IDENTIFIED BY 'openstack';"

配置MySQL

	openstack-config --set /etc/neutron/neutron.conf database connection mysql://neutron:openstack@controller0/neutron

配置Neutron Keystone 认证

	openstack-config --set /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_uri http://controller0:5000
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_host controller0
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_protocol http
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_port 35357
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken admin_tenant_name service
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken admin_user neutron
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken admin_password neutron

配置Neutron qpid
	openstack-config --set /etc/neutron/neutron.conf DEFAULT rpc_backend neutron.openstack.common.rpc.impl_qpid
	openstack-config --set /etc/neutron/neutron.conf DEFAULT qpid_hostname controller0


	openstack-config --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_status_changes True
	openstack-config --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_data_changes True
	openstack-config --set /etc/neutron/neutron.conf DEFAULT nova_url http://controller0:8774/v2

	openstack-config --set /etc/neutron/neutron.conf DEFAULT nova_admin_username nova
	openstack-config --set /etc/neutron/neutron.conf DEFAULT nova_admin_tenant_id $(keystone tenant-list | awk '/ service / { print $2 }')
	openstack-config --set /etc/neutron/neutron.conf DEFAULT nova_admin_password nova
	openstack-config --set /etc/neutron/neutron.conf DEFAULT nova_admin_auth_url http://controller0:35357/v2.0

配置Neutron ml2 plugin 用openvswitch

	ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
	cp /etc/init.d/neutron-openvswitch-agent /etc/init.d/neutronopenvswitch-agent.orig
	sed -i 's,plugins/openvswitch/ovs_neutron_plugin.ini,plugin.ini,g' /etc/init.d/neutron-openvswitch-agent

	openstack-config --set /etc/neutron/neutron.conf DEFAULT core_plugin ml2
	openstack-config --set /etc/neutron/neutron.conf DEFAULT service_plugins router

	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 type_drivers gre
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 tenant_network_types gre
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 mechanism_drivers openvswitch
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_gre tunnel_id_ranges 1:1000
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup firewall_driver neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup enable_security_group True

配置nova 使用Neutron 作为network 服务

	openstack-config --set /etc/nova/nova.conf DEFAULT network_api_class nova.network.neutronv2.api.API
	openstack-config --set /etc/nova/nova.conf DEFAULT neutron_url http://controller0:9696
	openstack-config --set /etc/nova/nova.conf DEFAULT neutron_auth_strategy keystone
	openstack-config --set /etc/nova/nova.conf DEFAULT neutron_admin_tenant_name service
	openstack-config --set /etc/nova/nova.conf DEFAULT neutron_admin_username neutron
	openstack-config --set /etc/nova/nova.conf DEFAULT neutron_admin_password neutron
	openstack-config --set /etc/nova/nova.conf DEFAULT neutron_admin_auth_url http://controller0:35357/v2.0
	openstack-config --set /etc/nova/nova.conf DEFAULT linuxnet_interface_driver nova.network.linux_net.LinuxOVSInterfaceDriver
	openstack-config --set /etc/nova/nova.conf DEFAULT firewall_driver nova.virt.firewall.NoopFirewallDriver
	openstack-config --set /etc/nova/nova.conf DEFAULT security_group_api neutron

重启nova controller 上的服务

	service openstack-nova-api restart
	service openstack-nova-scheduler restart
	service openstack-nova-conductor restart


启动Neutron server

	service neutron-server start
	chkconfig neutron-server on
	
##安装network节点

安装network 相关的包
	yum install -y openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch

允许ip forward

	vi /etc/sysctl.conf 
	net.ipv4.ip_forward=1
	net.ipv4.conf.all.rp_filter=0
	net.ipv4.conf.default.rp_filter=0

配置Neutron keysone 认证

	openstack-config --set /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_uri http://controller0:5000
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_host controller0
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_protocol http
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_port 35357
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken admin_tenant_name service
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken admin_user neutron
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken admin_password neutron

配置qpid 

	openstack-config --set /etc/neutron/neutron.conf DEFAULT rpc_backend neutron.openstack.common.rpc.impl_qpid
	openstack-config --set /etc/neutron/neutron.conf DEFAULT qpid_hostname controller0

配置Neutron 使用ml + openvswitch +gre

	openstack-config --set /etc/neutron/neutron.conf DEFAULT core_plugin ml2
	openstack-config --set /etc/neutron/neutron.conf DEFAULT service_plugins router

	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 type_drivers gre
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 tenant_network_types gre
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 mechanism_drivers openvswitch
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_gre tunnel_id_ranges 1:1000
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ovs local_ip 192.168.4.20
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ovs tunnel_type gre
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ovs enable_tunneling True
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup firewall_driver neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup enable_security_group True

	ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
	cp /etc/init.d/neutron-openvswitch-agent /etc/init.d/neutronopenvswitch-agent.orig
	sed -i 's,plugins/openvswitch/ovs_neutron_plugin.ini,plugin.ini,g' /etc/init.d/neutron-openvswitch-agent

配置l3

	openstack-config --set /etc/neutron/l3_agent.ini DEFAULT interface_driver neutron.agent.linux.interface.OVSInterfaceDriver
	openstack-config --set /etc/neutron/l3_agent.ini DEFAULT use_namespaces True

配置dhcp agent

	openstack-config --set /etc/neutron/dhcp_agent.ini DEFAULT interface_driver neutron.agent.linux.interface.OVSInterfaceDriver
	openstack-config --set /etc/neutron/dhcp_agent.ini DEFAULT dhcp_driver neutron.agent.linux.dhcp.Dnsmasq
	openstack-config --set /etc/neutron/dhcp_agent.ini DEFAULT use_namespaces True

配置metadata agent

	openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT auth_url http://controller0:5000/v2.0
	openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT auth_region regionOne
	openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT admin_tenant_name service
	openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT admin_user neutron
	openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT admin_password neutron
	openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT nova_metadata_ip controller0
	openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT metadata_proxy_shared_secret METADATA_SECRET

service openvswitch start
chkconfig openvswitch on

ovs-vsctl add-br br-int
ovs-vsctl add-br br-ex
ovs-vsctl add-port br-ex eth1

启动Neutron 服务

	service neutron-openvswitch-agent start
	service neutron-l3-agent start
	service neutron-dhcp-agent start
	service neutron-metadata-agent start

	chkconfig neutron-openvswitch-agent on
	chkconfig neutron-l3-agent on
	chkconfig neutron-dhcp-agent on
	chkconfig neutron-metadata-agent on


## 安装compute 服务

安装nova 相关包

	yum install -y openstack-nova-compute

配置nova

	openstack-config --set /etc/nova/nova.conf database connection mysql://nova:openstack@controller0/nova

	openstack-config --set /etc/nova/nova.conf DEFAULT auth_strategy keystone
	openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_uri http://controller0:5000
	openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_host controller0
	openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_protocol http
	openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_port 35357
	openstack-config --set /etc/nova/nova.conf keystone_authtoken admin_user nova
	openstack-config --set /etc/nova/nova.conf keystone_authtoken admin_tenant_name service
	openstack-config --set /etc/nova/nova.conf keystone_authtoken admin_password nova

	openstack-config --set /etc/nova/nova.conf DEFAULT rpc_backend qpid
	openstack-config --set /etc/nova/nova.conf DEFAULT qpid_hostname controller0

	openstack-config --set /etc/nova/nova.conf DEFAULT my_ip 192.168.4.30
	openstack-config --set /etc/nova/nova.conf DEFAULT vnc_enabled True
	openstack-config --set /etc/nova/nova.conf DEFAULT vncserver_listen 0.0.0.0
	openstack-config --set /etc/nova/nova.conf DEFAULT vncserver_proxyclient_address 10.20.0.10
	openstack-config --set /etc/nova/nova.conf DEFAULT novncproxy_base_url http://controller0:6080/vnc_auto.html
	openstack-config --set /etc/nova/nova.conf libvirt virt_type qemu

	openstack-config --set /etc/nova/nova.conf DEFAULT glance_host controller0


启动compute 节点服务

	service libvirtd start
	service messagebus start
	service openstack-nova-compute start

	chkconfig libvirtd on
	chkconfig messagebus on
	chkconfig openstack-nova-compute on

在controller 节点检查compute服务是否启动

	nova-manage service list

多出计算节点服务

	[root@controller0 ~]# nova-manage service list
	Binary           Host                                 Zone             Status     State Updated_At
	nova-consoleauth controller0                          internal         enabled    :-)   2014-07-19 09:04:18
	nova-cert        controller0                          internal         enabled    :-)   2014-07-19 09:04:19
	nova-conductor   controller0                          internal         enabled    :-)   2014-07-19 09:04:20
	nova-scheduler   controller0                          internal         enabled    :-)   2014-07-19 09:04:20
	nova-compute     compute0                             nova             enabled    :-)   2014-07-19 09:04:19

安装neutron ml2 和openvswitch agent 

	yum install openstack-neutron-ml2 openstack-neutron-openvswitch


配置Neutron Keystone 认证

	openstack-config --set /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_uri http://controller0:5000
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_host controller0
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_protocol http
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_port 35357
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken admin_tenant_name service
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken admin_user neutron
	openstack-config --set /etc/neutron/neutron.conf keystone_authtoken admin_password neutron

配置Neutron qpid
	openstack-config --set /etc/neutron/neutron.conf DEFAULT rpc_backend neutron.openstack.common.rpc.impl_qpid
	openstack-config --set /etc/neutron/neutron.conf DEFAULT qpid_hostname controller0

配置Neutron 使用 ml2 for ovs and gre

	openstack-config --set /etc/neutron/neutron.conf DEFAULT core_plugin ml2
	openstack-config --set /etc/neutron/neutron.conf DEFAULT service_plugins router

	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 type_drivers gre
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 tenant_network_types gre
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 mechanism_drivers openvswitch
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_gre tunnel_id_ranges 1:1000
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ovs local_ip 192.168.4.30
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ovs tunnel_type gre
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ovs enable_tunneling True
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup firewall_driver neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
	openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup enable_security_group True

	ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
	cp /etc/init.d/neutron-openvswitch-agent /etc/init.d/neutronopenvswitch-agent.orig
	sed -i 's,plugins/openvswitch/ovs_neutron_plugin.ini,plugin.ini,g' /etc/init.d/neutron-openvswitch-agent


配置 Nova 使用Neutron 提供网络服务

	openstack-config --set /etc/nova/nova.conf DEFAULT network_api_class nova.network.neutronv2.api.API
	openstack-config --set /etc/nova/nova.conf DEFAULT neutron_url http://controller0:9696
	openstack-config --set /etc/nova/nova.conf DEFAULT neutron_auth_strategy keystone
	openstack-config --set /etc/nova/nova.conf DEFAULT neutron_admin_tenant_name service
	openstack-config --set /etc/nova/nova.conf DEFAULT neutron_admin_username neutron
	openstack-config --set /etc/nova/nova.conf DEFAULT neutron_admin_password neutron
	openstack-config --set /etc/nova/nova.conf DEFAULT neutron_admin_auth_url http://controller0:35357/v2.0
	openstack-config --set /etc/nova/nova.conf DEFAULT linuxnet_interface_driver nova.network.linux_net.LinuxOVSInterfaceDriver
	openstack-config --set /etc/nova/nova.conf DEFAULT firewall_driver nova.virt.firewall.NoopFirewallDriver
	openstack-config --set /etc/nova/nova.conf DEFAULT security_group_api neutron

	openstack-config --set /etc/nova/nova.conf DEFAULT service_neutron_metadata_proxy true
	openstack-config --set /etc/nova/nova.conf DEFAULT neutron_metadata_proxy_shared_secret METADATA_SECRET

	service openvswitch start
	chkconfig openvswitch on
	ovs-vsctl add-br br-int

	service openstack-nova-compute restart
	service neutron-openvswitch-agent start
	chkconfig neutron-openvswitch-agent on

检查agent 是否启动正常

	neutron agent-list

启动正常显示

	[root@controller0 ~]# neutron agent-list
	+--------------------------------------+--------------------+----------+-------+----------------+
	| id                                   | agent_type         | host     | alive | admin_state_up |
	+--------------------------------------+--------------------+----------+-------+----------------+
	| 2c5318db-6bc2-4d09-b728-bbdd677b1e72 | L3 agent           | network0 | :-)   | True           |
	| 4a79ff75-6205-46d0-aec1-37f55a8d87ce | Open vSwitch agent | network0 | :-)   | True           |
	| 5a5bd885-4173-4515-98d1-0edc0fdbf556 | Open vSwitch agent | compute0 | :-)   | True           |
	| 5c9218ce-0ebd-494a-b897-5e2df0763837 | DHCP agent         | network0 | :-)   | True           |
	| 76f2069f-ba84-4c36-bfc0-3c129d49cbb1 | Metadata agent     | network0 | :-)   | True           |
	+--------------------------------------+--------------------+----------+-------+----------------+

初始网络

创建外部网络
	neutron net-create ext-net --shared --router:external=True

为外部网络添加subnet

	neutron subnet-create ext-net --name ext-subnet \
	--allocation-pool start=172.16.0.100,end=172.16.0.200 \
	--disable-dhcp --gateway 172.16.0.1 172.16.0.0/24

创建住户网络

首先创建demo用户、租户已经分配角色关系
	keystone user-create --name=demo --pass=demo --email=demo@example.com
	keystone tenant-create --name=demo --description="Demo Tenant"
	keystone user-role-add --user=demo --role=_member_ --tenant=demo

创建租户网络demo-net

	neutron net-create demo-net

为租户网络添加subnet
	neutron subnet-create demo-net --name demo-subnet --gateway 192.168.1.1 192.168.1.0/24


为租户网络创建路由，并连接到外部网络

	neutron router-create demo-router

将demo-net 连接到路由器

	neutron router-interface-add demo-router $(neutron net-show demo-net|awk '/ subnets / { print $4 }')

设置demo-router 默认网关

	neutron router-gateway-set demo-router ext-net


启动一个instance

nova boot --flavor m1.tiny --image $(nova image-list|awk '/ CirrOS / { print $2 }') --nic net-id=$(neutron net-list|awk '/ demo-net / { print $2 }') --security-group default demo-instance1


sudo ip addr add 172.16.0.20/24 dev br-ex;service network restart

sudo ip link set br-ex up


安装Dashboard

安装Dashboard 相关包
	yum install memcached python-memcached mod_wsgi openstack-dashboard

配置mencached

	CACHES = {
	'default': {
	'BACKEND' : 'django.core.cache.backends.memcached.MemcachedCache',
	'LOCATION' : '127.0.0.1:11211'
	}
	}

配置Keystone hostname

	OPENSTACK_HOST = "controller0"

启动Dashboard 相关服务

	service httpd start
	service memcached start
	chkconfig httpd on
	chkconfig memcached on

打开浏览器验证,用户名：admin 密码：admin

	http://10.20.0.10/dashboard


其他服务安装步骤，待续...
