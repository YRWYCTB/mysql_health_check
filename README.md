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

配置文件分别在各个节点的/etc/consul.d目录下创建。

server-agent的文件名为server.json

client-agent的文件名为client.json

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
注："bootstrap_expect": 1,为server节点个数，测试时server节点为一个，生产环境推荐3、5；

至少需要三个server节点保证服务的高可用性。
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

主节点运行如下sql，sys库下创建视图，用监控MGR 从节点延时 ：
```sql
USE sys;

DELIMITER $$

CREATE FUNCTION IFZERO(a INT, b INT)
RETURNS INT
DETERMINISTIC
RETURN IF(a = 0, b, a)$$

CREATE FUNCTION LOCATE2(needle TEXT(10000), haystack TEXT(10000), offset INT)
RETURNS INT
DETERMINISTIC
RETURN IFZERO(LOCATE(needle, haystack, offset), LENGTH(haystack) + 1)$$

CREATE FUNCTION GTID_NORMALIZE(g TEXT(10000))
RETURNS TEXT(10000)
DETERMINISTIC
RETURN GTID_SUBTRACT(g, '')$$

CREATE FUNCTION GTID_COUNT(gtid_set TEXT(10000))
RETURNS INT
DETERMINISTIC
BEGIN
  DECLARE result BIGINT DEFAULT 0;
  DECLARE colon_pos INT;
  DECLARE next_dash_pos INT;
  DECLARE next_colon_pos INT;
  DECLARE next_comma_pos INT;
  SET gtid_set = GTID_NORMALIZE(gtid_set);
  SET colon_pos = LOCATE2(':', gtid_set, 1);
  WHILE colon_pos != LENGTH(gtid_set) + 1 DO
     SET next_dash_pos = LOCATE2('-', gtid_set, colon_pos + 1);
     SET next_colon_pos = LOCATE2(':', gtid_set, colon_pos + 1);
     SET next_comma_pos = LOCATE2(',', gtid_set, colon_pos + 1);
     IF next_dash_pos < next_colon_pos AND next_dash_pos < next_comma_pos THEN
       SET result = result +
         SUBSTR(gtid_set, next_dash_pos + 1,
                LEAST(next_colon_pos, next_comma_pos) - (next_dash_pos + 1)) -
         SUBSTR(gtid_set, colon_pos + 1, next_dash_pos - (colon_pos + 1)) + 1;
     ELSE
       SET result = result + 1;
     END IF;
     SET colon_pos = next_colon_pos;
  END WHILE;
  RETURN result;
END$$

CREATE FUNCTION gr_applier_queue_length()
RETURNS INT
DETERMINISTIC
BEGIN
  RETURN (SELECT sys.gtid_count( GTID_SUBTRACT( (SELECT
Received_transaction_set FROM performance_schema.replication_connection_status
WHERE Channel_name = 'group_replication_applier' ), (SELECT
@@global.GTID_EXECUTED) )));
END$$

CREATE FUNCTION gr_member_in_primary_partition()
RETURNS VARCHAR(3)
DETERMINISTIC
BEGIN
  RETURN (SELECT IF( MEMBER_STATE='ONLINE' AND ((SELECT COUNT(*) FROM
performance_schema.replication_group_members WHERE MEMBER_STATE != 'ONLINE') >=
((SELECT COUNT(*) FROM performance_schema.replication_group_members)/2) = 0),
'YES', 'NO' ) FROM performance_schema.replication_group_members JOIN
performance_schema.replication_group_member_stats USING(member_id));
END$$

CREATE VIEW gr_member_routing_candidate_status AS SELECT
sys.gr_member_in_primary_partition() as viable_candidate,
IF( (SELECT (SELECT GROUP_CONCAT(variable_value) FROM
performance_schema.global_variables WHERE variable_name IN ('read_only',
'super_read_only')) != 'OFF,OFF'), 'YES', 'NO') as read_only,
sys.gr_applier_queue_length() as transactions_behind, Count_Transactions_in_queue as 'transactions_to_cert' from performance_schema.replication_group_member_stats;$$

DELIMITER ;
```
上述视图可以用来监控从库延时情况，如下为压测过程中从库事务延时情况
```sql
mysql> select * from sys.gr_member_routing_candidate_status;
+------------------+-----------+---------------------+----------------------+
| viable_candidate | read_only | transactions_behind | transactions_to_cert |
+------------------+-----------+---------------------+----------------------+
| YES              | YES       |                   3 |                    2 |
+------------------+-----------+---------------------+----------------------+
1 row in set (0.00 sec)
```
primary节点如下
```sql
mysql> select * from sys.gr_member_routing_candidate_status;
+------------------+-----------+---------------------+----------------------+
| viable_candidate | read_only | transactions_behind | transactions_to_cert |
+------------------+-----------+---------------------+----------------------+
| YES              | NO        |                   0 |                    0 |
+------------------+-----------+---------------------+----------------------+
1 row in set (0.00 sec)
### 4.1、检查是节点是否为primary，三个节点均添加

在/data/consul/shell目录下创建脚本
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
# 判断节点事务延时 need to be tested 
#find out if secondary is delayed from primary.
delay_trans= `$comm -Nse "select transactions_behind from sys.gr_member_routing_candidate_status"`
if [ $delay_trans -ge 200 ]
then
   echo "This instance has more than 200 transactions behind primary node ...."
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

三个client-agent节点(mysql实例所在节点)均添加服务注册文件

文件内容如下：只需将address改为所部署机器Ip，port该为mysql端口

在/etc/consul.d下创建服务注册配置文件

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
      "port": 3306,
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
      "port": 3306,
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

重新启动consul-client之后（kill consul进程后重新启动）
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
也可以使用curl进行结果的获取
```sh
curl http://172.18.0.150:8500/v1/health/service/write-mysql-primary?passing=true 
```
结果如下
```
[{
	"Node": {
		"ID": "4a8cee76-31da-14f0-5037-e08076b8bd91",
		"Node": "dzst152",
		"Address": "172.18.0.152",
		"Datacenter": "dc1",
		"TaggedAddresses": {
			"lan": "172.18.0.152",
			"lan_ipv4": "172.18.0.152",
			"wan": "172.18.0.152",
			"wan_ipv4": "172.18.0.152"
		},
		"Meta": {
			"consul-network-segment": ""
		},
		"CreateIndex": 3472,
		"ModifyIndex": 3473
	},
	"Service": {
		"ID": "write-mysql-primary",
		"Service": "write-mysql-primary",
		"Tags": ["master-write"],
		"Address": "172.18.0.152",
		"TaggedAddresses": {
			"lan_ipv4": {
				"Address": "172.18.0.152",
				"Port": 3317
			},
			"wan_ipv4": {
				"Address": "172.18.0.152",
				"Port": 3317
			}
		},
		"Meta": null,
		"Port": 3317,
		"Weights": {
			"Passing": 1,
			"Warning": 1
		},
		"EnableTagOverride": false,
		"Proxy": {
			"MeshGateway": {},
			"Expose": {}
		},
		"Connect": {},
		"CreateIndex": 3474,
		"ModifyIndex": 3474
	},
	"Checks": [{
		"Node": "dzst152",
		"CheckID": "serfHealth",
		"Name": "Serf Health Status",
		"Status": "passing",
		"Notes": "",
		"Output": "Agent alive and reachable",
		"ServiceID": "",
		"ServiceName": "",
		"ServiceTags": [],
		"Type": "",
		"Definition": {},
		"CreateIndex": 3472,
		"ModifyIndex": 3472
	}, {
		"Node": "dzst152",
		"CheckID": "service:write-mysql-primary",
		"Name": "Service 'write-mysql-primary' check",
		"Status": "passing",
		"Notes": "",
		"Output": "/bin/bash: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)\nmysql: [Warning] Using a password on the command line interface can be insecure.\nmysql: [Warning] Using a password on the command line interface can be insecure.\nmysql: [Warning] Using a password on the command line interface can be insecure.\nmysql: [Warning] Using a password on the command line interface can be insecure.\nMySQL 3317  Instance is master ........\n",
		"ServiceID": "write-mysql-primary",
		"ServiceName": "write-mysql-primary",
		"ServiceTags": ["master-write"],
		"Type": "script",
		"Definition": {},
		"CreateIndex": 3474,
		"ModifyIndex": 13891
	}]
}]
```
读库检测
```sh
curl http://172.18.0.150:8500/v1/health/service/read-mysql-slave?passing=true 
```
结果如下
```
[{
	"Node": {
		"ID": "2ca5bc56-44b5-c699-9256-887817769e1b",
		"Node": "dzst151",
		"Address": "172.18.0.151",
		"Datacenter": "dc1",
		"TaggedAddresses": {
			"lan": "172.18.0.151",
			"lan_ipv4": "172.18.0.151",
			"wan": "172.18.0.151",
			"wan_ipv4": "172.18.0.151"
		},
		"Meta": {
			"consul-network-segment": ""
		},
		"CreateIndex": 1807,
		"ModifyIndex": 1808
	},
	"Service": {
		"ID": "read-mysql-slave",
		"Service": "read-mysql-slave",
		"Tags": ["slave-read"],
		"Address": "172.18.0.151",
		"TaggedAddresses": {
			"lan_ipv4": {
				"Address": "172.18.0.151",
				"Port": 3317
			},
			"wan_ipv4": {
				"Address": "172.18.0.151",
				"Port": 3317
			}
		},
		"Meta": null,
		"Port": 3317,
		"Weights": {
			"Passing": 1,
			"Warning": 1
		},
		"EnableTagOverride": false,
		"Proxy": {
			"MeshGateway": {},
			"Expose": {}
		},
		"Connect": {},
		"CreateIndex": 1811,
		"ModifyIndex": 1811
	},
	"Checks": [{
		"Node": "dzst151",
		"CheckID": "serfHealth",
		"Name": "Serf Health Status",
		"Status": "passing",
		"Notes": "",
		"Output": "Agent alive and reachable",
		"ServiceID": "",
		"ServiceName": "",
		"ServiceTags": [],
		"Type": "",
		"Definition": {},
		"CreateIndex": 1807,
		"ModifyIndex": 15276
	}, {
		"Node": "dzst151",
		"CheckID": "service:read-mysql-slave",
		"Name": "Service 'read-mysql-slave' check",
		"Status": "passing",
		"Notes": "",
		"Output": "/bin/bash: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)\nmysql: [Warning] Using a password on the command line interface can be insecure.\nmysql: [Warning] Using a password on the command line interface can be insecure.\nmysql: [Warning] Using a password on the command line interface can be insecure.\nmysql: [Warning] Using a password on the command line interface can be insecure.\nMySQL 3317  Instance is slave ........\n",
		"ServiceID": "read-mysql-slave",
		"ServiceName": "read-mysql-slave",
		"ServiceTags": ["slave-read"],
		"Type": "script",
		"Definition": {},
		"CreateIndex": 1811,
		"ModifyIndex": 15282
	}]
}, {
	"Node": {
		"ID": "550130bb-3a92-5624-ec12-b69700def621",
		"Node": "dzst160",
		"Address": "172.18.0.160",
		"Datacenter": "dc1",
		"TaggedAddresses": {
			"lan": "172.18.0.160",
			"lan_ipv4": "172.18.0.160",
			"wan": "172.18.0.160",
			"wan_ipv4": "172.18.0.160"
		},
		"Meta": {
			"consul-network-segment": ""
		},
		"CreateIndex": 4688,
		"ModifyIndex": 4689
	},
	"Service": {
		"ID": "read-mysql-slave",
		"Service": "read-mysql-slave",
		"Tags": ["slave-read"],
		"Address": "172.18.0.160",
		"TaggedAddresses": {
			"lan_ipv4": {
				"Address": "172.18.0.160",
				"Port": 3317
			},
			"wan_ipv4": {
				"Address": "172.18.0.160",
				"Port": 3317
			}
		},
		"Meta": null,
		"Port": 3317,
		"Weights": {
			"Passing": 1,
			"Warning": 1
		},
		"EnableTagOverride": false,
		"Proxy": {
			"MeshGateway": {},
			"Expose": {}
		},
		"Connect": {},
		"CreateIndex": 4693,
		"ModifyIndex": 4693
	},
	"Checks": [{
		"Node": "dzst160",
		"CheckID": "serfHealth",
		"Name": "Serf Health Status",
		"Status": "passing",
		"Notes": "",
		"Output": "Agent alive and reachable",
		"ServiceID": "",
		"ServiceName": "",
		"ServiceTags": [],
		"Type": "",
		"Definition": {},
		"CreateIndex": 4688,
		"ModifyIndex": 4688
	}, {
		"Node": "dzst160",
		"CheckID": "service:read-mysql-slave",
		"Name": "Service 'read-mysql-slave' check",
		"Status": "passing",
		"Notes": "",
		"Output": "/bin/bash: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)\nmysql: [Warning] Using a password on the command line interface can be insecure.\nmysql: [Warning] Using a password on the command line interface can be insecure.\nmysql: [Warning] Using a password on the command line interface can be insecure.\nmysql: [Warning] Using a password on the command line interface can be insecure.\nMySQL 3317  Instance is slave ........\n",
		"ServiceID": "read-mysql-slave",
		"ServiceName": "read-mysql-slave",
		"ServiceTags": ["slave-read"],
		"Type": "script",
		"Definition": {},
		"CreateIndex": 4693,
		"ModifyIndex": 13826
	}]
}]
```
对于一般主从复制的健康检查脚本，放在该项目的shell文件下。
