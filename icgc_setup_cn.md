[TOC]
### 1. 操作系统要求

系统要求:
* Ubunutu 14.04 
* 充足的磁盘空间
* 充足的 swap 空间

主机信息:
节点名称| 主机名 | IP
----|-----|----|----
Portal Server|lxv-icgc-stg2-portal01|10.50.10.40
Portal Server|lxv-icgc-stg2-portal02|10.50.10.54
Postgres|lxv-icgc-stg2-postgres01|10.50.10.66
Nginx Server|lxv-icgc-stg2-nginx01|10.50.10.61
Download Server|lxv-icgc-stg2-download01|10.50.10.14
mgmt server|lxv-icgc-stg2-mgmt01|10.50.10.51
Varnish Cache Server|lxv-icgc-stg2-varnish01|10.50.10.76 
Elasticsearch Nodes|lxv-icgc-stg2-elastic01|10.50.10.80
Elasticsearch Nodes|lxv-icgc-stg2-elastic02|10.50.10.81
Elasticsearch Nodes|lxv-icgc-stg2-elastic03|10.50.10.18 
Elasticsearch Nodes|lxv-icgc-stg2-elastic04|10.50.10.39 
Elasticsearch Nodes|lxv-icgc-stg2-elastic05|10.50.10.44 
HDFS Nodes|lxv-icgc-stg2-hdfs01|10.50.10.49
HDFS Nodes|lxv-icgc-stg2-hdfs02|10.50.10.37
HDFS Nodes|lxv-icgc-stg2-hdfs03|10.50.10.57 
HDFS Nodes|lxv-icgc-stg2-hdfs04|10.50.10.56 
HDFS Nodes|lxv-icgc-stg2-hdfs05|10.50.10.43 

hosts信息（存入文件 hosts.txt):
112.124.140.210 mirrors.aliyun.com
10.50.10.40 portal-server-01
10.50.10.54 portal-server-02
10.50.10.66 db-server-pgsql
10.50.10.61 nginx-server
10.50.10.14 download-server
10.50.10.51 mgmt-server
10.50.10.76 varnish-server
10.50.10.80 elastic-server-01
10.50.10.81 elastic-server-02
10.50.10.18 elastic-server-03
10.50.10.39 elastic-server-04
10.50.10.44 elastic-server-05
10.50.10.49 hdfs-01
10.50.10.37 hdfs-02
10.50.10.57 hdfs-03
10.50.10.56 hdfs-04
10.50.10.43 hdfs-05

&ensp;
### 2. 备注

阿里云的虚拟机需要调整 host 文件，将 ubuntu 的软件库调整为阿里云的本地库 （ 可在后面修改 host 文件时一并处理）
```bash
112.124.140.210 mirrors.aliyun.com
```
&ensp;
### 3. 前期准备工作（系统环境配置）

#### 3.1 挂载数据文件分区到对应的节点
对应 elasticsearch 节点，挂载数据分区 **/dev/vdb2** 到 **/var/lib/elasticsearch** 目录。目录属主:属组为 root:root, 权限为 755。配置 fstab, 启用开机挂载。
```bash
#mkdir -p /var/lib/elasticsearch
#vim /etc/fstab
    Modify =>  /dev/vdb2          /var/lib/elasticsearch           ext4    defaults        0 2
```
对应 HDFS 节点，挂载数据分区 **/dev/vdb2** 到 **/dfs** 目录。权限与 elasticsearch 相同。配置 fstab, 启用开机挂载。
```bash
#mkdir /dfs
#mount /dev/vdb2 /dfs
#vim /etc/fstab
    Modify => /dev/vdb2          /dfs           ext4    defaults        0 2
```

#### 3.2 设置配置账户的免密登陆

在管理节点(mgmt-node)上面，新建配置账户。暂用配置账户为 mid, gid / uid 均为 650。调整管理节点的主机名
```bash
#groupadd -g 650 mid
#useradd -g 650 -u 650 mid
#mkdir /home/mid
#cp -rfp /etc/skel/* /home/mid/
#chown -R mid:mid /home/mid
#passwd mid
#echo 'mgmt-server' > /etc/hostname
#hostname 'mgmt-server'
#su - mid
$ssh-keygen -t rsa #生成免密证书
```

在各节点上面新建 mid 账户，uid / gid 保持一致，且开启 sudo。调整各节点的主机名。
```bash
#groupadd -g 650 mid
#useradd -g 650 -u 650 mid
#mkdir /home/mid
#cp -rfp /etc/skel/* /home/mid/
#chown -R mid:mid /home/mid
#passwd mid
#vim /etc/sudoers
    Add => <username> ALL=(ALL:ALL) ALL
    Add => Defaults:<username>  !authenticate 
#echo <host_name> > /etc/hostname
#hostname <host_name>
```

调整管理节点的 host 解析，增加其他节点的名称解析。将 *#1* 中的 hosts.txt 信息贴入管理节点 /etc/hosts 文件中

复制管理节点上面的证书到其他各节点(以 mid 用户执行， 使用脚本处理)
*add_host_all.sh*
```bash
#!/bin/bash
# add_host_all.sh 用于批量处理其他机器的 host / hostname 信息

all_host=`cat hosts.txt| grep -v 'mgmt-server'| cut -d ' ' -f 2`

for host in $all_host; do
    echo "Working on $host"
    ssh-copy-id [mid@]$host
    scp add_host_one.sh $host:/tmp
    ssh $host /usr/bin/sudo /tmp/add_host_one.sh $host
done
```

*add_host_one.sh*
```bash
#!/bin/bash
# add_host_one.sh 配置单一节点的脚本

cp /etc/hosts /etc/hosts.orig

cat << EOF > /etc/hosts

127.0.0.1 localhost
127.0.1.1       localhost.localdomain   localhost

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

112.124.140.210 mirrors.aliyun.com

10.50.10.40 portal-server-01
10.50.10.54 portal-server-02
10.50.10.66 db-server-pgsql
10.50.10.61 nginx-server
10.50.10.14 download-server
10.50.10.51 mgmt-server
10.50.10.76 varnish-server
10.50.10.80 elastic-server-01
10.50.10.81 elastic-server-02
10.50.10.18 elastic-server-03
10.50.10.39 elastic-server-04
10.50.10.44 elastic-server-05
10.50.10.49 hdfs-01
10.50.10.37 hdfs-02
10.50.10.57 hdfs-03
10.50.10.56 hdfs-04
10.50.10.43 hdfs-05


EOF

host=$1
echo $host > /etc/hostname
hostname $host
```
&ensp;
### 4. 管理节点 ansible 安装

#### 4.1 使用软件库安装

```bash
$sudo apt-get update
$sudo apt-get install ansible
```

查看管理节点（将运行 ansible 的自动部署脚本的节点）安装的 ansible 版本 `ansible --version`

如果安装的版本是1.x, 用命令 `sudo dpkg -r ansible`, 先把旧版本的 ansible 进行卸载

参考 https://community.spiceworks.com/how_to/110622-install-ansible-on-64-bit-ubuntu-14-04-lts , 手动升级软件库版本，安装最新版 ansible

```bash
sudo apt-get update && sudo apt-get install software-properties-common
sudo apt-add-repository ppa:ansible/ansible
sudo apt-get update && sudo apt-get install ansible
```

#### 4.2 使用手动安装

如果出现如下报错信息，可以确定为 ansible 版本过低。如果软件库版本已为最新，则需要手动安装，具体参考 ansible 官方安装教程。

```bash
$ansible-playbook -i config/hosts portal.yml

ERROR: become is not a legal parameter in an Ansible task or handler
```

#### 4.3 ansible 的配置

解压 icgc.tgz 包，进入 icgc/ansible 目录，所有部署命令都会在该目录下执行

编辑 vars/main.yml, 确保 **ansible_ssh_user** 和 **ansible_ssh_private_key_file** 参数为本次配置的免密登陆信息

编辑 config/hosts, 确保各节点的主机名信息匹配

确保 postgresql 节点的 swap 已开启，否则稍后会报错

&ensp;
### 5. 各节点的软件环境配置

#### 5.1 Java 环境配置
由于 portal-server / download-server / elasticsearch-server / hdfs-server 的配置都需要 java_1.8 的环境，先行更新这些节点的软件库信息

登陆以上各节点，更新 Java 源
```bash
$sudo dpkg --configure -a
$sudo apt-get install software-properties-common
$sudo add-apt-repository ppa:webupd8team/java
$sudo apt-get update
```

#### 5.2 数据库节点 Postgresql 安装
```bash
$sudo apt-get install python-software-properties libpq-dev python-pip python-psycopg2
```
&ensp;
### 6. 安装 Portal Server

#### 6.1 Portal Serve 项目代码部署 
```bash
$ansible-playbook -i config/hosts portal.yml -c paramiko
```

> 该脚本会同时部署 nginx-server / varnish-server / pgsql-server / portal-server
>
> 去除 '-c' 的选项后，ansible 命令可能会因为 ssh host key 校验而失败。当 CLI 提示 ssh 校验确认的时候，输入 'yes'

当安装 Java SDK 的时候，忽略如下错误：
```
   fatal: [lxv-icgc-elastic04]: FAILED! => {"changed": false, "cmd": "java -version", "failed": true, "msg": "[Errno 2] No such file or directory", "rc": 2}

#脚本会在部署 Java 环境之前，先检查一遍 java 环境

   TASK [portal : make sure to kill the potential previously running instance] ****

   fatal: [lxv-icgc-portal02]: FAILED! => {"changed": true, "cmd": "pkill -IO -f WrapperSimpleApp || true", "delta": "0:00:00.060439", "end": "2016-09-12 22:39:48.332188", "failed": true, "rc": -29, "start": "2016-09-12 22:39:48.271749", "stderr": "", "stdout": "", "stdout_lines": [], "warnings": []}

   ...ignoring

```

在深圳的阿里云环境上，安装 elasticsearch 的 knapsack 插件会失败，部署脚本已把该部分（ansible/roles/elasticsearch/tasks/plugins.yml）进行了注释。

需要手动将 elasticsearch-knapsack-1.4.4.1-plugin.zip 插件包复制到各 elasticsearch 节点，使用如下命令进行安装

```bash
$sudo /usr/share/elasticsearch/bin/plugin -url file:///home/mid/elasticsearch-knapsack-1.4.4.1-plugin.zip -install knapsack

$sudo /etc/init.d/elasticsearch restart

```

#### 6.2 Elasticsearch 集群主从设置

登录 Elasticsearch 各节点，修改配置文件，设置第一个节点为 master 节点

编辑 '/etc/elasticsearch/elasticsearch.yml',新增如下内容：
```
node:

    master: true
    
    data: false
```

其他节点配置为：
```
node:

    master: false
    
    data: true
```
   
修改配置完毕后，重启 Elasticsearch 服务
```
sudo /etc/init.d/elasticsearch restart
```

#### 6.3 确认配置生效

使用命令 
```
curl 'http://<elastic_search_master_node_ip>:9200/_cluster/health?pretty=1'
```

正常回显为：
```
root@mgmt-server:~/icgc/ansible# curl 'http://elastic-server-01:9200/_cluster/health?pretty=1'
{
  "cluster_name" : "elasticsearch",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 5,
  "number_of_data_nodes" : 4,
  "active_primary_shards" : 40,
  "active_shards" : 67,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0
}
```

**Notice：**
1.如果主节点无法发现其他节点，可能是由于内部网络环境导致组播包传输失败。需要修改配置文件，将组播改为单播，并补充单播其他节点的信息。需要在所有 Elasticsearch 节点配置

示例：master -> /etc/elasticsearch/elasticsearch.yml
```
discovery.zen.ping.multicast.enabled: false
discovery.zen.ping.unicast.hosts: ["elastic-server-02", "elastic-server-03", "elastic-server-04", "elastic-server-05"]
```
   
2.调整 Java heap space 参数大小，设置参数为物理内存一半。
对于 64GB 内存的系统，设置 30GB heap space。设置 heap size 超过物理内存，会导致 elasticsearch 无法启动

编辑 /etc/default/elasticsearch
```
ES_HEAP_SIZE=30g
```

调整以上配置后，需要重启 elasticsearch 的服务才能生效

#### 6.4 Postgresql 初始化

下载最新的 **库/表** 结构
```
wget https://raw.githubusercontent.com/icgc-dcc/dcc-portal/develop/dcc-portal-server/src/main/sql/schema.sql
```
  
该sql 文件中，新建库的代码已被注释。手动新建数据库账户 'dcc'，密码 'dcc'
   
* 手动建库 dcc_portal
* 使用 psql dcc_portal -f scheme.sql 导入表结构(su - postgres 切换为 postgres 用户)


#### 6.5 Nginx 配置

默认部署完成后 Nginx 已配置，配置文件位于 /etc/nginx/sites-avaliable

nginx 重定向 http 的请求到 https
```
   # HTTPS

    server {

            listen  443;

            server_name icgcportal.icgc.genomics.cn;

            ssl                 on;

            ssl_certificate     /etc/ssl/dcc/portal.crt;

            ssl_certificate_key /etc/ssl/dcc/portal.key;



            ssl_session_timeout  5m;

            ssl_protocols  TLSv1 TLSv1.1 TLSv1.2;

            ssl_ciphers  HIGH:!RC4:!MD5:!aNULL:!eNULL:!NULL:!DH:!EDH:!EXP:+MEDIUM;

            ssl_prefer_server_ciphers   on;



            location / {

                    proxy_pass http://web-cluster;

            }

    }



    # HTTP

    server {

            listen 80;

            server_name icgcportal.icgc.genomics.cn;

            return 301 https://$server_name$request_uri;

    }
```

#### 6.6 调整 Portal Server 中项目的配置信息

登录各 Portal Server 节点，调整配置文件 **/srv/dcc-portal-server/conf/application.yml**，修改里面 elasticsearch / pgsql / download_server 等相关信息

```
## database
spring.datasource:
  driver-class-name: org.postgresql.Driver
  url: jdbc:postgresql://db-server-pgsql/dcc_portal
  username: dcc
  password: dcc
  max-active: 10
  max-idle: 1
  min-idle: 1

## nginx 反向代理
web:
  # Defines an external URL when the portal is behind a reverse proxy / load balancer. E.g. shortUrl resource uses it for generation of valid URLs
  baseUrl: https://nginx-server

## elasticsearch (master 节点)
elastic:
  indexName: icgc22-13
  repoIndexName: icgc-repository-20160830
  nodeAddresses:
     - host: elastic-server-01
       port: 9300
  client:
    "client.transport.sniff": true

## download
## The "sharedSecret" and "aesKey" must match the ones in the jwt section of download server's application.yml.
download:
  enabled: true
  # ignore login ? add by Felix
  requestLoggingEnabled: true
  serverUrl: "https://download-server"  #内部ip
  publicServerUrl: "https://10.50.10.14"  #外部ip，用户可直接访问
  sharedSecret: "deadbeefdeadbeefdeadbeefdeadbeef"
  aesKey: "deadbeefdeadbeef"

## release information
release:
  releaseDate: "August 23rd, 2016"
  dataVersion: ICGC22

## ignore authentication
auth:
  enabled: false
```

**Notice：**没有看到 */srv* 目录时，则确定为 ansible 自动部署失败

&ensp;
### 7. HDFS 节点部署

* roles/hdfs/defaults/main.yml `hdfs_namenode_host` 设定对应的集群主机
* roles/download/vars/main.yml `download_name` `download_dist_name` `hdfs_uri` 设定对应的主机名

部署 HDFS 节点：
```
ansible-playbook -i config/hosts hdfs.yml -c paramiko
```

常见问题为 Java 环境安装失败，参考 *#12/Issue-01* 处理

查看 HDFS 集群状态
```
# 使用 links 命令，在命令行浏览网页
apt-get install links

links http://<hdfs-master-node>:50070
```
&ensp;
### 8. Download 节点部署

#### 8.1 项目代码部署

```
ansible-playbook -i config/hosts download.yml -c paramiko
```

查看 Portal Server 和 Download Server 的连接状态 
```
curl -k -v https://download-server/srv-info/health
```

#### 8.2 确保 Portal Server 和Download Server 使用 https 交换数据

生成 Download Server 的自验证证书
```
keytool -genkey -keyalg RSA -alias selfsigned -keystore keystore.jks -storepass icgcbgi -validity 3600 -keysize 2048
```

复制证书文件 **keystore.jks** 到 **/srv/dcc-download-server/conf/**，配置文件**/srv/dcc-download-server/conf/application.yml**
```
server:
    port: 443
    ssl:
        keyStore: "../conf/keystore.ks"
        keyStorePassword: "icgcbgi"
```

修改配置文件后，重启 download server
```
cd /srv/dcc-download-server/bin; sudo ./dcc-download-server restart
```

#### 8.4 导入 HDFS 数据 

解压 data.open.tar 文件，将解压后的文件复制到 HDFS 节点本地磁盘或使用 NFS 挂载

导入 HDFS 原始数据：
```
#由于权限原因，需要使用 hdfs 用户执行数据导入命令
$sudo su hdfs

$hdfs dfs -copyFromLocal release_22 /icgc/input
```
&ensp;
### 9. 导入 Release 数据的索引

从任一内存充足的节点（eg. hdfs-01）执行以下命令，把 release 的数据索引导入 hdfs
```
# 测试用例只导入了一个项目（LAML-CN）的数据
java -Xmx12g -jar dcc-download-import.jar -i release.tar -es es://lxv-icgc-elastic01:9300 -p LAML-CN      
```

> 默认导入所有项目的数据
> 只能选择导入单个或所有项目
> 使用 '-p' 参数导入单个项目
>
> 如果输出为 "cluster is in red" 或者 "cluster is in yellow"，请检查 elasticsearch 是否配置了足够的 heap space

&ensp;
### 10. 导入 Dialy Repository Index
在安装了 knapsack 插件的 elasticsearch 服务器上，执行如下命令：
```
java -jar dcc-download-import.jar -i repository.tar.gz -es es://localhost:9300
#curl -XPOST "http://elastic-server-01:9200/_import?path=<path_to_repository.tar.gz>"
```
&ensp;
### 11. 重启Portal Server
```
sudo /svr/dcc-portal-server/bin/dcc-portal-server restart
```
&ensp;
### 12. 常见问题处理
*Issue-01：* **Java 环境在自动化部署时安装失败（用 java -version 检查），一般为网络异常导致**
先把已安装的 Java 包进行卸载
```bash
$sudo dpkg --configure -a

$sudo apt-get remove oracle-java8-installer oracle-java8-set-default
```

然后复制已完成安装节点的缓存数据 **/var/cache/oracle-jdk8-installer/** 到失败节点对应目录。重新运行部署命令

&ensp;
### 13. 附加信息

1.测试环境下，Elasticsearch 导入需要 2.5h, HDFS 导入需要 1h

2.使用命令获取 Elasticsearch 索引列表
```
curl 'http://<elasticsearch-master:9200/_cat/indices?pretty=1'
```

3.删除 Elasticsearch 索引
```
curl -XDELETE <elasticsearch-master>:9200/icgc21-0-3
```

4.获取最新的数据导入 java 包
```
# 若不确定版本信息，可用浏览器访问

wget 'https://artifacts.oicr.on.ca/artifactory/dcc-release/org/icgc/dcc/dcc-download-import/[RELEASE]/dcc-download-import-[RELEASE].jar'
```

5.下载最新的数据
访问链接 https://download.icgc.org/exports， 获取各数据的下载链接

> data.open.tar => 导入 HDFS 的原始数据（每季度更新）
> release.tar => 原始数据的完整索引信息（每季度更新）
> repository.tar.gz => 每日索引信息（每季度更新）

6.使用 screen 执行长时间命令（代理节点必须一直在线）
```
Example .screenrc:



deflogin on

login on

hardstatus alwayslastline

hardstatus string '%{= kG}%-Lw%{= kW}%50> %n%f* %t%{= kG}%+Lw%< %{= kG}%-=%c:%s%{-}'

defscrollback 30000



screen -t bash

screen -t bash

screen -t bash

screen -t bash

screen -t portal01

screen -t portal02 ssh lxv-icgc-portal02

screen -t varnish01 ssh lxv-icgc-varnish01

screen -t elastic01 ssh lxv-icgc-elastic01

screen -t postgres01 ssh lxv-icgc-postgres01

screen -t hdfs01 ssh lxv-icgc-hdfs01

screen -t nginx01 ssh lxv-icgc-nginx01



term linux

termcapinfo xterm* ti@:te@



bindkey -k k5 prev # F5 for previous window

bindkey -k k6 next # F6 for next window
```
