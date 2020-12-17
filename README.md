## 一、前言
`Docker Compose`是 `docker` 提供的一个命令行工具，用来定义和运行由多个容器组成的应用。使用 `compose`，我们可以通过 `YAML` 文件声明式的定义应用程序的各个服务，并由单个命令完成应用的创建和启动。

## 二、docker-compose安装
### 2.1 pip方式安装

```bash
pip install docker-compose 
```
### 2.2 查看版本

```bash
docker-compose version
```
## 三、集群架构
### 3.1 结构图及解释
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201217134147993.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)

- `Master`节点作为`Master`节点与协调节点，为防止脑裂问题，降低负载，不存数据

- `Node1~Node3`为数据节点，不参与`Master`竞选

- `TribeNode`节点不存数据，不参与`Master`竞选

### 3.2 集群划分
> 因`Docker`中已部署单节点的`ElasticSearch` ，9200和9300端口已被占用。

| 节点目录         | 节点名称 | 协调端口号 | 说明                         | 查询端口号 | 节点IP |
| ---------------- | -------- | ---------- | ---------------------------- | ---------- | ---------- |
| docker-es-master | master   | 9301       | master节点，非数据节点       | 9201       | 192.168.3.27|
| docker-es-data01 | data01   | 9302       | 数据节点1，非master节点      | 9202       | 192.168.3.27|
| docker-es-data02 | data02   | 9303      | 数据节点2，非master节点      | 9203       | 192.168.3.27|
| docker-es-data03 | data03   | 9304       | 数据节点3，非master节点      | 9204       | 192.168.3.27|
| docker-es-tribe  | tribe    | 9305       | 协调节点，非master非数据节点 | 9205      | 192.168.3.27|

## 四、集群配置
### 4.1 目录结构
下载地址：[https://github.com/gm19900510/docker-es-cluster](https://github.com/gm19900510/docker-es-cluster)
```bash
.
├── docker-es-data01
│   ├── data01
│   ├── data01-logs
│   ├── docker-compose.yml
│   ├── .env
│   └── es-config
│       └── elasticsearch.yml
├── docker-es-data02
│   ├── data02
│   ├── data02-logs
│   ├── docker-compose.yml
│   ├── .env
│   └── es-config
│       └── elasticsearch.yml
├── docker-es-data03
│   ├── data03
│   ├── data03-logs
│   ├── docker-compose.yml
│   ├── .env
│   └── es-config
│       └── elasticsearch.yml
├── docker-es-master
│   ├── docker-compose.yml
│   ├── .env
│   ├── es-config
│   │   └── elasticsearch.yml
│   ├── master-data
│   └── master-logs
└── docker-es-tribe
    ├── docker-compose.yml
    ├── .env
    ├── es-config
    │   └── elasticsearch.yml
    ├── tribe-data
    └── tribe-logs
```
### 4.2 集群配置说明
#### 4.2.1 master节点docker-compose.yml配置说明
`docker-compose.yml` 是`docker-compose`的配置文件
```bash
version: "3"
services:
    es-master:
        image: elasticsearch:7.9.3
        container_name: es-master
        environment: # setting container env
            - ES_JAVA_OPTS=${ES_JVM_OPTS}   # set es bootstrap jvm args
        restart: always
        volumes:
            - ./es-config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
            - ${MASTER_DATA_DIR}:/usr/share/elasticsearch/data:rw
            - ${MASTER_LOGS_DIR}:/usr/share/elasticsearch/logs:rw
        network_mode: "host"
```
> 修改`pull`的镜像，替换其中的变量与配置文件，挂载数据与日志目录，最后用的`host`主机模式，让节点服务占用到实体机端口

>启动`ElasticSearch` 时如果提示无法访问日志或数据目录的问题可以通过,在`docker-compose.yml`的`environment`节点下添加`- TAKE_FILE_OWNERSHIP=true`

#### 4.2.2 master节点elaticsearch.yml配置说明
`elaticsearch.yml` 是`ElasticSearch`的配置文件，搭建集群最关键的文件之一

```bash
# ======================== Elasticsearch Configuration =========================
cluster.name: es-cluster
node.name: master 
node.master: true
node.data: false
node.attr.rack: r1 
bootstrap.memory_lock: true 
http.port: 9201
network.host: 192.168.3.27
transport.tcp.port: 9301
discovery.seed_hosts: ["192.168.3.27:9302","192.168.3.27:9303","192.168.3.27:9304","192.168.3.27:9305"] 
cluster.initial_master_nodes: ["master"] 
gateway.recover_after_nodes: 2
```
> - `transport.tcp.port` 设置`Elaticsearch`多节点协调的端口号
>- `discovery.seed_hosts` 设置当前节点启动后要发现的协调节点位置，当然自己不需要发现自己，推荐使用`ip:port`形式，集群形成快
>- `cluster.initial_master_nodes` 集群中可以成为`master`节点的节点名，这里指定唯一的一个，防止脑裂

#### 4.2.4 master节点.env配置说明
`.env` 这个文件为`docker-compose.yml`提供默认参数，方便修改

```bash
# the default environment for es-master
# set es node jvm args
ES_JVM_OPTS=-Xms2048m -Xmx2048m
# set master node data folder
MASTER_DATA_DIR=./master-data
# set master node logs folder
MASTER_LOGS_DIR=./master-logs
```


## 五、使用说明
### 5.1 多服务器环境使用说明
1. 若想将此脚本使用到生产上，需要修改每个节点下的`.env文件`，`将挂载数据`、`日志目录`修改为启动`Elaticsearch`的集群的用户可读写的位置，可以通过`sudo chmod 777 -R 目录` 或 `sudo chown -R 当前用户名:用户组 目录` 来修改被挂载的目录权限。

2. 修改`.env`下的`JVM`参数，扩大堆内存，启动与最大值最好相等，以减少`gc`次数，提高效率。

3. 修改所有节点下的`docker-compose.yml` 中的`network.host`地址 为当前所放置的主机的`ip`，`discovery.seed_hosts`需要填写具体各待发现节点的实体机`ip`，以确保可以组成集群。

4. 确保各端口在其宿主机上没有被占用，如有占用需确认是否有用，无用`kill`，有用则更新`docker-compose.yml`的`http.port`或`transport.tcp.port`，注意与此同时要更新其它节点的`discovery.seed_hosts`对应的`port`。

5. `docker-compose up -d`后台启动命令。

6. `docker-compose down`关闭同时移除容器与多余虚拟网卡。

7. `docker stop contains_name`根据容器名称关闭容器，不移除容器。

### 5.2 单服务环境使用说明
1. `sh docker-es-cluster-up.sh`创建并启动集群
2. `shdocker-es-cluster-stop.sh`停止集群
3. `docker-es-cluster-down.sh`停止并移除集群
>- 如果你想让这些脚本有执行权限，不妨试试sudo chmod +x *.sh
>- 这些脚本中没有使用sudo，如需要使用sudo才能启动docker,请添加当前用户到docker组

## 六、启动服务的常见问题
### 6.1 `max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]`
问题翻译过来就是：`Elasticsearch`用户拥有的内存权限太小，至少需要`262144`；

解决办法：

执行命令：

```bash
sudo sysctl -w vm.max_map_count=262144
```

查看结果：

```bash
sudo  sysctl -a|grep vm.max_map_count
```

显示：

```bash
vm.max_map_count = 262144
```

上述方法修改之后，如果重启虚拟机将失效，所以：

在   `/etc/sysctl.conf`文件最后添加一行，`vm.max_map_count=262144`，即可永久修改

### 6.2 `memory locking requested for elasticsearch process but memory is not locked`
解决方法一，关闭`bootstrap.memory_lock`，会影响性能

```bash
vim elasticsearch.yml          // 设置成false就正常运行了。
bootstrap.memory_lock: false
```
解决方法二，开启`bootstrap.memory_lock`
1. 修改文件`elasticsearch.yml`，上面那个报错就是开启后产生的，如果开启还要修改其它系统配置文件 

```bash
vim elasticsearch.yml
bootstrap.memory_lock: true
```

2. 修改文件`/etc/security/limits.conf`，最后添加以下内容。      

```bash
* soft nofile 65536

* hard nofile 65536

* soft nproc 32000

* hard nproc 32000

* hard memlock unlimited

* soft memlock unlimited
```

3. 修改文件 `/etc/systemd/system.conf` ，分别修改以下内容。

```bash
DefaultLimitNOFILE=65536

DefaultLimitNPROC=32000

DefaultLimitMEMLOCK=infinity
```

改好后**重启系统**。再启动`Elasticsearch`就没报错了 。

## 七、效果验证

```bash
docker ps 
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201217142846306.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
`curl http://localhost:9200/_cat/health?v` 查看集群状态，出现如下信息则集群搭建成功![在这里插入图片描述](https://img-blog.csdnimg.cn/20201217144226540.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201217141810858.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)

