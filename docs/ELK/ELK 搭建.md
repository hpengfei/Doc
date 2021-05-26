## 环境搭建

### 安装说明

| 主机名 | IP             | 安装用户 | 系统            | 软件版本 |
| ------ | -------------- | -------- | --------------- | -------- |
| elk    | 192.168.122.70 | elk      | CentOS 7.4.1708 | 7.1.0    |

### elasticsearch 安装

#### 下载

```shell
# wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.1.0-linux-x86_64.tar.gz
# tar -xf elasticsearch-7.1.0-linux-x86_64.tar.gz -C /opt
```

#### 文件句柄设置

```shell
echo "session    required     pam_limits.so">>/etc/pam.d/login
echo "*                hard    nofile          65536">>/etc/security/limits.conf
echo "*                soft    nofile          65536">>/etc/security/limits.conf
echo "*                hard    nproc          4096">>/etc/security/limits.conf
echo "*                soft    nproc          4096">>/etc/security/limits.conf
```

#### 系统参数设置

```shell
echo "vm.max_map_count = 262144">> /etc/sysctl.conf 
sysctl -p
```

#### 主机名解析

```shell
# tail -1 /etc/hosts
192.168.122.70 elk
```

#### 软件配置

```shell
$ cd /opt/elasticsearch-7.1.0
$ egrep -v "^$|^#" config/elasticsearch.yml   
network.host: 192.168.122.70
discovery.seed_hosts: ["elk","127.0.0.1","192.168.122.70"]
```

#### 启动报错说明

CentOS 7 默认启动 elasticsearch  会报如下的错误，按照报错提示并通过上述的配置则可以正常启动。

```
[2019-10-20T17:07:12,083][INFO ][o.e.b.BootstrapChecks    ] [elk] bound or publishing to a non-loopback address, enforcing bootstrap checks
ERROR: [3] bootstrap checks failed
[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65535]
[2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
[3]: the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured
```

#### 验证服务启动

```shell
# curl http://192.168.122.70:9200
{
  "name" : "elk",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "KxUotVAGRBqJK8bUNcJ3NA",
  "version" : {
    "number" : "7.1.0",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "606a173",
    "build_date" : "2019-05-16T00:43:15.323135Z",
    "build_snapshot" : false,
    "lucene_version" : "8.0.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

#### 安装插件

```shell
$ ./bin/elasticsearch-plugin list
$ ./bin/elasticsearch-plugin install analysis-icu
-> Downloading analysis-icu from elastic
[=================================================] 100%   
$ ./bin/elasticsearch-plugin list
analysis-icu
```

安装插件之后重启 elasticsearch 服务，可以通过接口查看插件安装情况：

```shell
# curl http://192.168.122.70:9200/_cat/plugins
elk analysis-icu 7.1.0
```

#### 多实例安装

修改配置文件：

```shell
$ grep 'cluster.initial_master_nodes' config/elasticsearch.yml 
cluster.initial_master_nodes: ["node0", "node1", "node2", "node3"]
```

启动多实例：

```shell
$ pwd
/opt/elasticsearch-7.1.0
./bin/elasticsearch -E node.name=node0 -E cluster.name=elk-cluster -E path.data=node0_data -d
./bin/elasticsearch -E node.name=node1 -E cluster.name=elk-cluster -E path.data=node1_data -d
./bin/elasticsearch -E node.name=node2 -E cluster.name=elk-cluster -E path.data=node2_data -d 
./bin/elasticsearch -E node.name=node3 -E cluster.name=elk-cluster -E path.data=node3_data -d
```

验证多实例：

```shell
$ ls -d  node*
node0_data  node1_data  node2_data  node3_data
$ curl http://192.168.122.70:9200/_cat/nodes
192.168.122.70 16 84 0 0.82 1.31 0.91 mdi * node0
192.168.122.70 11 84 0 0.82 1.31 0.91 mdi - node2
192.168.122.70 12 84 0 0.82 1.31 0.91 mdi - node1
192.168.122.70 32 84 0 0.82 1.31 0.91 mdi - node3
```

### kibana 安装

#### 下载

```shell
wget https://artifacts.elastic.co/downloads/kibana/kibana-7.1.0-linux-x86_64.tar.gz
tar -xf kibana-7.1.0-linux-x86_64.tar.gz -C /opt/
```

#### 服务配置

```shell
$ cd /opt/kibana-7.1.0-linux-x86_64/
$ egrep -v "^$|^#" config/kibana.yml 
server.host: "192.168.122.70"
elasticsearch.hosts: ["http://192.168.122.70:9200"]
```

#### 启动服务

```shell
$ ./bin/kibana &
```

#### 登录

启动服务没报错之后可以在浏览器输入 `http://192.168.122.70:5601` 即可进入 kibana 管理界面。

#### 安装插件

安装插件的方式和前面 elasticsearch 安装的方式一样，如下：

```
./bin/kibana-plugin install plugin_name
```

## 通过 docker 安装ELK

### 安装docker

```shell
# cat  >/etc/sysctl.d/k8s.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
# sysctl -p
# yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# yum install -y docker-ce-18.09.8
# systemctl start docker
# systemctl enable docker
# cat  >/etc/docker/daemon.json<<EOF
{
  "registry-mirrors": ["https://xxxxxx.mirror.aliyuncs.com"]
}
EOF
# systemctl daemon-reload
# systemctl restart docker
```

### 编辑 compose 文件

```yaml
version: '2.2'
services:
  cerebro:
    image: lmenezes/cerebro:0.8.3
    container_name: cerebro
    ports:
      - "9000:9000"
    command:
      - -Dhosts.0.host=http://elasticsearch:9200
    networks:
      - es7net
  kibana:
    image: kibana:7.1.0
    container_name: kibana7
    environment:
      - I18N_LOCALE=zh-CN
      - XPACK_GRAPH_ENABLED=true
      - TIMELION_ENABLED=true
      - XPACK_MONITORING_COLLECTION_ENABLED="true"
    ports:
      - "5601:5601"
    networks:
      - es7net
  elasticsearch:
    image: elasticsearch:7.1.0
    container_name: es7_01
    environment:
      - cluster.name=geektime
      - node.name=es7_01
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - discovery.seed_hosts=es7_01,es7_02
      - cluster.initial_master_nodes=es7_01,es7_02
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es7data1:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - es7net
  elasticsearch2:
    image: elasticsearch:7.1.0
    container_name: es7_02
    environment:
      - cluster.name=geektime
      - node.name=es7_02
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - discovery.seed_hosts=es7_01,es7_02
      - cluster.initial_master_nodes=es7_01,es7_02
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es7data2:/usr/share/elasticsearch/data
    networks:
      - es7net

volumes:
  es7data1:
    driver: local
  es7data2:
    driver: local

networks:
  es7net:
    driver: bridge
```

### 启动 docker-compose

需要在上面yaml 文件目录中执行如下的命令：

```shell
# docker-compose up -d
```

### 验证

在浏览器输入 `http://192.168.122.70:5601/` 地址登录到kibana，输入 `http://192.168.122.70:9000/` 登录到 cerebro 后台。也可以在命令行通关curl 命令进行验证 elasticsearch 节点。

```shell
# curl http://192.168.122.70:9200/_cat/nodes
172.18.0.2 35 82 1 0.14 0.44 0.53 mdi - es7_01
172.18.0.5 28 82 1 0.14 0.44 0.53 mdi * es7_02
```

### 多机容器部署

多主机部署容器的前提是weave网络配置好，不然跨主机之间容器无法通信。在执行脚本 `./start.sh 0.0.0.0:`  创建容器之前，需要执行`eval $(weave env)` 命令才能确保新建的容器在weave网络中。

```
# tree np-es-01/
np-es-01/
├── d
│   └── usr
│       └── share
│           └── elasticsearch
│               ├── config
│               │   ├── config
│               │   ├── elasticsearch.keystore
│               │   ├── elasticsearch.yml
│               │   ├── jvm.options
│               │   ├── jvm.options.d
│               │   ├── log4j2.properties
│               │   ├── role_mapping.yml
│               │   ├── roles.yml
│               │   ├── users
│               │   └── users_roles
│               ├── data
│               └── logs
└── start.sh
# cat np-es-01/start.sh
#!/bin/bash
if [[ $# == 0 ]]; then
        echo "会启动一套系统"
        echo "格式:"
        echo "          $0 192.168.2.100: 5000"
        echo "      容器名为当前目录名"
else
        C_HOSTIP=$1
        PORTSTART=$2
        DIR_NAME=$(pwd)
        C_NAME=$(basename ${DIR_NAME})
        mkdir -p $(pwd)/d/usr/share/elasticsearch/data $(pwd)/d/usr/share/elasticsearch/logs
        sudo chmod g+rwx -R $(pwd)/d
        sudo chgrp 0 -R $(pwd)/d
        docker run -t -d -i --name ${C_NAME} --hostname ${C_NAME}.weave.local \
                --privileged \
                --restart=unless-stopped \
                --ulimit nofile=262144:262144 \
                --ulimit memlock=-1:-1 \
                -e "ES_JAVA_OPTS=-Xms2048m -Xmx2048m" \
                --cpus 1 \
                --memory 2G \
                -p ${C_HOSTIP}9201:9200 \
                -p ${C_HOSTIP}9301:9300 \
                -v $(pwd)/d/usr/share/elasticsearch/data:/usr/share/elasticsearch/data \
                -v $(pwd)/d/usr/share/elasticsearch/logs:/usr/share/elasticsearch/logs \
                -v $(pwd)/d/usr/share/elasticsearch/config:/usr/share/elasticsearch/config \
                docker.elastic.co/elasticsearch/elasticsearch:7.9.2
fi
# cat np-es-01/d/usr/share/elasticsearch/config/elasticsearch.yml
bootstrap.memory_lock: false
bootstrap.system_call_filter: false
cluster.initial_master_nodes: ["np-es-01","np-es-02","np-es-03"]
cluster.name: "np-es"
discovery.seed_hosts: ["np-es-01:9300","np-es-02:9300","np3-es-03:9300"]
discovery.zen.minimum_master_nodes: 2
# discovery.zen.ping.unicast.hosts: ["np3-es00-00:9300","np3-es00-02:9300"]
http.cors.allow-origin: "*"
http.cors.enabled: true
http.port: 9200
network.host: 0.0.0.0
node.data: true
node.master: true
node.name: "np-es-01"
transport.tcp.port: 9300
```

```
# tree np-es-02/
np-es-02/
├── d
│   └── usr
│       └── share
│           └── elasticsearch
│               ├── config
│               │   ├── config
│               │   ├── elasticsearch.keystore
│               │   ├── elasticsearch.yml
│               │   ├── jvm.options
│               │   ├── jvm.options.d
│               │   ├── log4j2.properties
│               │   ├── role_mapping.yml
│               │   ├── roles.yml
│               │   ├── users
│               │   └── users_roles
│               ├── data
│               └── logs
└── start.sh
# cat np-es-02/start.sh
#!/bin/bash
if [[ $# == 0 ]]; then
        echo "会启动一套系统"
        echo "格式:"
        echo "          $0 192.168.2.100: 5000"
        echo "      容器名为当前目录名"
else
        C_HOSTIP=$1
        PORTSTART=$2
        DIR_NAME=$(pwd)
        C_NAME=$(basename ${DIR_NAME})
        mkdir -p $(pwd)/d/usr/share/elasticsearch/data $(pwd)/d/usr/share/elasticsearch/logs
        sudo chmod g+rwx -R $(pwd)/d
        sudo chgrp 0 -R $(pwd)/d
        docker run -t -d -i --name ${C_NAME} --hostname ${C_NAME}.weave.local \
                --privileged \
                --restart=unless-stopped \
                --ulimit nofile=262144:262144 \
                --ulimit memlock=-1:-1 \
                -e "ES_JAVA_OPTS=-Xms2048m -Xmx2048m" \
                --cpus 1 \
                --memory 2G \
                -p ${C_HOSTIP}9202:9200 \
                -p ${C_HOSTIP}9302:9300 \
                -v $(pwd)/d/usr/share/elasticsearch/data:/usr/share/elasticsearch/data \
                -v $(pwd)/d/usr/share/elasticsearch/logs:/usr/share/elasticsearch/logs \
                -v $(pwd)/d/usr/share/elasticsearch/config:/usr/share/elasticsearch/config \
                docker.elastic.co/elasticsearch/elasticsearch:7.9.2
fi
# cat np-es-02/d/usr/share/elasticsearch/config/elasticsearch.yml
bootstrap.memory_lock: false
bootstrap.system_call_filter: false
cluster.initial_master_nodes: ["np-es-01","np-es-02","np-es-03"]
cluster.name: "np-es"
discovery.seed_hosts: ["np-es-01:9300","np-es-02:9300","np-es-03:9300"]
discovery.zen.minimum_master_nodes: 2
# discovery.zen.ping.unicast.hosts: ["np3-es00-00:9300","np3-es00-01:9300"]
http.cors.allow-origin: "*"
http.cors.enabled: true
http.port: 9200
network.host: 0.0.0.0
node.data: true
node.master: true
node.name: "np-es-02"
transport.tcp.port: 9300
```

```
# tree np-es-03/
np-es-03/
├── d
│   └── usr
│       └── share
│           └── elasticsearch
│               ├── config
│               │   ├── config
│               │   ├── elasticsearch.keystore
│               │   ├── elasticsearch.yml
│               │   ├── jvm.options
│               │   ├── jvm.options.d
│               │   ├── log4j2.properties
│               │   ├── role_mapping.yml
│               │   ├── roles.yml
│               │   ├── users
│               │   └── users_roles
│               ├── data
│               └── logs
└── start.sh
# cat  np-es-03/start.sh
#!/bin/bash
if [[ $# == 0 ]]; then
        echo "会启动一套系统"
        echo "格式:"
        echo "          $0 192.168.2.100: 5000"
        echo "      容器名为当前目录名"
else
        C_HOSTIP=$1
        PORTSTART=$2
        DIR_NAME=$(pwd)
        C_NAME=$(basename ${DIR_NAME})
        mkdir -p $(pwd)/d/usr/share/elasticsearch/data $(pwd)/d/usr/share/elasticsearch/logs
        sudo chmod g+rwx -R $(pwd)/d
        sudo chgrp 0 -R $(pwd)/d
        docker run -t -d -i --name ${C_NAME} --hostname ${C_NAME}.weave.local \
                --privileged \
                --restart=unless-stopped \
                --ulimit nofile=262144:262144 \
                --ulimit memlock=-1:-1 \
                -e "ES_JAVA_OPTS=-Xms2048m -Xmx2048m" \
                --cpus 1 \
                --memory 2G \
                -p ${C_HOSTIP}9203:9200 \
                -p ${C_HOSTIP}9303:9300 \
                -v $(pwd)/d/usr/share/elasticsearch/data:/usr/share/elasticsearch/data \
                -v $(pwd)/d/usr/share/elasticsearch/logs:/usr/share/elasticsearch/logs \
                -v $(pwd)/d/usr/share/elasticsearch/config:/usr/share/elasticsearch/config \
                docker.elastic.co/elasticsearch/elasticsearch:7.9.2
fi
# cat np-es-03/d/usr/share/elasticsearch/config/elasticsearch.yml
bootstrap.memory_lock: false
bootstrap.system_call_filter: false
cluster.initial_master_nodes: ["np-es-01","np-es-02","np-es-03"]
cluster.name: "np-es"
discovery.seed_hosts: ["np-es-01:9300","np-es-02:9300","np-es-03:9300"]
discovery.zen.minimum_master_nodes: 2
# discovery.zen.ping.unicast.hosts: ["np3-es00-00:9300","np3-es00-01:9300"]
http.cors.allow-origin: "*"
http.cors.enabled: true
http.port: 9200
network.host: 0.0.0.0
node.data: true
node.master: true
node.name: "np-es-03"
transport.tcp.port: 9300
```

```
# tree np-kba01/
np-kba01/
├── d
│   └── usr
│       └── share
│           └── kibana
│               ├── config
│               │   └── kibana.yml
│               └── data
└── start.sh
# cat np-kba01/start.sh
#!/bin/bash
if [[ $# == 0 ]]; then
        echo "会启动一套系统"
        echo "格式:"
        echo "          $0 192.168.2.100: 5000"
        echo "      容器名为当前目录名"
else
        C_HOSTIP=$1
        PORTSTART=$2
        DIR_NAME=$(pwd)
        C_NAME=$(basename ${DIR_NAME})
        mkdir -p $(pwd)/d
        docker run -t -d -i --name ${C_NAME} --hostname ${C_NAME}.weave.local \
                --privileged \
                --restart=unless-stopped \
                --ulimit nofile=262144:262144 \
                --cpus 1 \
                --memory 1G \
                -p ${C_HOSTIP}5601:5601 \
                -v $(pwd)/d/usr/share/kibana/config:/usr/share/kibana/config \
                -v $(pwd)/d/usr/share/kibana/data:/usr/share/kibana/data \
                kibana:7.9.2
fi
# cat np-kba01/d/usr/share/kibana/config/kibana.yml
server.host: "0.0.0.0"
server.port: 5601
server.name: "kba-np3"
elasticsearch.hosts: ["http://np-es-01:9200","http://np-es-02:9200","http://np-es-03:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "123.com"
kibana.index: ".kibana"
i18n.locale: "zh-CN"
```

## logstash 导入测试数据

### 安装 logstash

```shell
# tar -xf logstash-7.1.0.tar.gz -C /opt/
# cd /opt/logstash-7.1.0/
```

### 配置 logstash

```
# cat config/logstash.conf 
input {
  file {
    path => "/data/logstash/movies.csv"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}
filter {
  csv {
    separator => ","
    columns => ["id","content","genre"]
  }

  mutate {
    split => { "genre" => "|" }
    remove_field => ["path", "host","@timestamp","message"]
  }

  mutate {

    split => ["content", "("]
    add_field => { "title" => "%{[content][0]}"}
    add_field => { "year" => "%{[content][1]}"}
  }

  mutate {
    convert => {
      "year" => "integer"
    }
    strip => ["title"]
    remove_field => ["path", "host","@timestamp","message","content"]
  }

}
output {
   elasticsearch {
     hosts => "http://localhost:9200"
     index => "movies"
     document_id => "%{id}"
   }
  stdout {}
}
```

### 导入数据

```shell
# head -2 /data/logstash/movies.csv  
movieId,title,genres
1,Toy Story (1995),Adventure|Animation|Children|Comedy|Fantasy
# ./bin/logstash -f config/logstash.conf 
```

