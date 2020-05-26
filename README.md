# mysql_health_check
Series of scripts to check the states of mysql instances.
## 1、下载安装consul
```sh
wget https://releases.hashicorp.com/consul/1.7.3/consul_1.7.3_linux_amd64.zip
```
解压之后为一个二进制文件，将其拷贝到/usr/local/bin目录下

创建如下目录：放配置文件；数据；及redis或mysql的健康检查脚本。
```sh
mkdir /etc/consul.d/ -p 
mkdir /data/consul/ -p
mkdir /data/consul/shell -p
```
## 2、创建配置文件

Consul-server 172.18.0.150
```sh
vim /etc/consul.d/server.json
{
  "data_dir": "/data/consul",
  "datacenter": "dc1",
  "log_level": "INFO",
  "server": true,
  "advertise_addr":"172.18.0.150",
  "bootstrap_expect": 1,
  "bind_addr": "172.18.0.150",
  "client_addr": "172.18.0.150",
  "ui":true
}
```
注："bootstrap_expect": 1,为server节点个数，测试时server节点为一个，生产环境推荐3、5
```sh
Consul-client  172.18.0.151/172.18.0.152/172.18.0.160
vim  /etc/consul.d/client.json
{
  "data_dir": "/data/consul",
  "enable_script_checks": true,
  "bind_addr": "172.18.0.151",
  "retry_join": ["172.18.0.150"],
  "retry_interval": "30s",
  "rejoin_after_leave": true,
  "start_join": ["172.18.0.150"]
}
vim /etc/consul.d/client.json
{
  "data_dir": "/data/consul",
  "enable_script_checks": true,
  "bind_addr": "172.18.0.152",
  "retry_join": ["172.18.0.150"],
  "retry_interval": "30s",
  "rejoin_after_leave": true,
  "start_join": ["172.18.0.150"]
}
vim /etc/consul.d/client.json
{
  "data_dir": "/data/consul",
  "enable_script_checks": true,
  "bind_addr": "172.18.0.160",
  "retry_join": ["172.18.0.150"],
  "retry_interval": "30s",
  "rejoin_after_leave": true,
  "start_join": ["172.18.0.150"]
}
```
## 3 、启动agent

启动consul server 在172.18.0.150上
```sh
nohup consul agent -config-dir=/etc/consul.d > /data/consul/consul.log &
```
启动consul client 在172.18.0.151/172.18.0.152/172.18.0.160
```sh
nohup consul agent -config-dir=/etc/consul.d > /data/consul/consul.log &
```
观察consul server的log日志3个client自动注册到了consul上了

## 4、MySQL检查脚本（MGR）
### 4.1、检查是节点是否为primary，三个节点均添加
```sh
vim /data/consul/shell/check_mysql_mgr_master.sh
#!/bin/bash
port=3306
user="root"
passwod="passwd"

comm="/usr/local/mysql57/bin/mysql -u$user -h127.0.0.1 -P$port -p$passwod"
value=`$comm -Nse "select 1"`
primary_member=`$comm -Nse "select variable_value from performance_schema.global_status WHERE VARIABLE_NAME= 'group_replication_primary_member'"`
server_uuid=`$comm -Nse "select variable_value from performance_schema.global_variables where VARIABLE_NAME='server_uuid';"`


# 判断mysql是否存活
if [ -z $value ]
then
   echo "mysql $port is down....."
   exit 2
fi


# 判断节点状态
node_state=`$comm -Nse "select MEMBER_STATE from performance_schema.replication_group_members where MEMBER_ID='$server_uuid'"`
if [ $node_state != "ONLINE" ]
then
   echo "MySQL $port state is not online...."
   exit 2
fi


# 判断是不是主节点
if [[ $server_uuid == $primary_member ]]
then
   echo "MySQL $port  Instance is primary node  ........"
   exit 0
else
   echo "MySQL $port  Instance is secondary node ........"
   exit 2
fi
```
### 4.2、检查是节点是否为secondary，三个节点均添加
```sh
vim /data/consul/shell/check_mysql_mgr_slave.sh

#!/bin/bash
port=3317
user="tian"
passwod="8085782"

comm="/usr/local/mysql57/bin/mysql -u$user -h127.0.0.1 -P$port -p$passwod"
value=`$comm -Nse "select 1"`
primary_member=`$comm -Nse "select variable_value from performance_schema.global_status WHERE VARIABLE_NAME= 'group_replication_primary_member'"`
server_uuid=`$comm -Nse "select variable_value from performance_schema.global_variables where VARIABLE_NAME='server_uuid';"`


# 判断mysql是否存活
if [ -z $value ]
then
   echo "mysql $port is down....."
   exit 2
fi

# 判断节点状态
node_state=`$comm -Nse "select MEMBER_STATE from performance_schema.replication_group_members where MEMBER_ID='$server_uuid'"`
if [ $node_state != "ONLINE" ]
then
   echo "MySQL $port state is not online...."
   exit 2
fi


# 判断是不是主节点
if [[ $server_uuid != $primary_member ]]
then
   echo "MySQL $port  Instance is slave ........"
   exit 0
else
   node_num=`$comm -Nse "select count(*) from  performance_schema.replication_group_members"`
   # 判断如果没有任何从节点，主节点也注册从角色服务。
   if [ $node_num -eq 1 ]
   then
       echo "MySQL $port  Instance is slave ........"
       exit 0
   else
       echo "MySQL $port  Instance is master ........"
       exit 2
   fi
fi
```
## 5、增加服务注册配置：

三个节点均添加，只将address改为所部署机器Ip，port为mysql端口
```sh
vim /etc/consul.d/master.json
{
  "services": [
    {
      "name": "write-mysql-primary",
      "tags": [
        "master-write"
      ],
      "address": "172.18.0.151",
      "port": 3317,
      "checks": [
        {
           "Args":["/data/consul/shell/check_mysql_mgr_master.sh"],
          "Shell": "/bin/bash",
          "interval": "15s"
        }
      ]
    }
  ]
}
```
```sh
vim /etc/consul.d/slave.json
{
  "services": [
    {
      "name": "read-mysql-slave",
      "tags": [
        "slave-read"
      ],
      "address": "172.18.0.151",
      "port": 3317,
      "checks": [
        {
           "Args":["/data/consul/shell/check_mysql_mgr_slave.sh"],
           "Shell": "/bin/bash",
           "interval": "15s"
        }
      ]
    }
  ]
}
```
## 6、查看服务注册是否正常：

重新启动consul-client之后
```sh
[zhaofeng.tian@l-betadb2.ops.p1 ~]$ dig @172.18.0.150 -p 8600 read-mysql-slave.service.consul

; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7_5.1 <<>> @172.18.0.150 -p 8600 read-mysql-slave.service.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 7123
;; flags: qr aa rd; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;read-mysql-slave.service.consul. IN	A

;; ANSWER SECTION:
read-mysql-slave.service.consul. 0 IN	A	172.18.0.160
read-mysql-slave.service.consul. 0 IN	A	172.18.0.152

;; Query time: 0 msec
;; SERVER: 172.18.0.150#8600(172.18.0.150)
;; WHEN: Tue May 26 12:21:31 CST 2020
;; MSG SIZE  rcvd: 92
```
```sh
[zhaofeng.tian@l-betadb2.ops.p1 ~]$ dig @172.18.0.150 -p 8600 write-mysql-primary.service.consul

; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7_5.1 <<>> @172.18.0.150 -p 8600 write-mysql-primary.service.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 65057
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;write-mysql-primary.service.consul. IN	A

;; ANSWER SECTION:
write-mysql-primary.service.consul. 0 IN A	172.18.0.151

;; Query time: 1 msec
;; SERVER: 172.18.0.150#8600(172.18.0.150)
;; WHEN: Tue May 26 12:24:09 CST 2020
;; MSG SIZE  rcvd: 79
```
根据上述信息，可以区分当前主节点及从节点IP
