## zookeeper 配置

### 主机一

#### 目录

```shell
# tree np-zk-01
np-zk-01
├── d
│   └── zookeeper
│       ├── conf
│       │   ├── configuration.xsl
│       │   ├── java.env
│       │   ├── log4j.properties
│       │   ├── zk_server_jaas.conf
│       │   └── zoo.cfg
│       ├── data
│       │   └── myid
│       └── logs
└── start.sh
```

#### 配置文件

```shell
# cat np-zk-01/d/zookeeper/conf/zoo.cfg
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/data
dataLogDir=/logs
clientPort=2181
server.1=np-zk-01:2888:3888
server.2=np-zk-02:2888:3888
server.3=np-zk-03:2888:3888
authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
requireClientAuthScheme=sasl
jaasLoginRenew=3600000
#zookeeper.sasl.client=true
quorum.auth.enableSasl=true
quorum.auth.learnerRequireSasl=true
quorum.auth.serverRequireSasl=true
quorum.auth.learner.saslLoginContext=QuorumLearner
quorum.auth.server.saslLoginContext=QuorumServer
quorum.cnxn.threads.size=6
4lw.commands.whitelist=*
```

#### 认证文件

```shell
# cat np-zk-01/d/zookeeper/conf/zk_server_jaas.conf
Server {
    org.apache.zookeeper.server.auth.DigestLoginModule required
    username=admin
    password=111111
    user_admin=111111
    user_kafkaDev=111111;
};
QuorumServer {
       org.apache.zookeeper.server.auth.DigestLoginModule required
       user_admin="111111";
};
QuorumLearner {
       org.apache.zookeeper.server.auth.DigestLoginModule required
       username="admin"
       password="111111";
};
```

#### 启动参数

```shell
# cat d/zookeeper/conf/java.env
SERVER_JVMFLAGS="-Djava.security.auth.login.config=/conf/zk_server_jaas.conf -Xms1024m -Xmx1024m"
```

#### 启动脚本

```shell
# cat np-zk-01/start.sh
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
        sudo chmod g+rwx -R $(pwd)/d
        sudo chgrp 0 -R $(pwd)/d
        eval $(weave env)
        docker run -t -d -i --name ${C_NAME} --hostname ${C_NAME}.weave.local \
                --privileged \
                --restart=unless-stopped \
                --ulimit nofile=262144:262144 \
                --ulimit memlock=-1:-1 \
                --cpus 1 \
                --memory 1G \
                --env JVMFLAGS=" -Xms4G -Xmx4G" \
                -p ${C_HOSTIP}12181:2181 \
                -v /etc/localtime:/etc/localtime \
                -v $(pwd)/d/zookeeper/data:/data \
                -v $(pwd)/d/zookeeper/logs:/logs \
                -v $(pwd)/d/zookeeper/conf:/conf \
                zookeeper:3.6.2
        eval $(weave env --restore)
fi
```

zk 总共部署在三台机器上，除了myid 和容器名称变量`C_NAME` 不同，其它的均相同。

#### 验证集群

```shell
# echo stat |nc 10.10.10.232 12181
```

## kafka 配置

### 服务端配置

#### 目录

```shell
# tree np-kafka-01
np-kafka-01
├── d
│   └── kafka
│       ├── config
│       │   ├── connect-console-sink.properties
│       │   ├── connect-console-source.properties
│       │   ├── connect-distributed.properties
│       │   ├── connect-file-sink.properties
│       │   ├── connect-file-source.properties
│       │   ├── connect-log4j.properties
│       │   ├── connect-mirror-maker.properties
│       │   ├── connect-standalone.properties
│       │   ├── consumer.properties
│       │   ├── kafka_server_jaas.conf
│       │   ├── log4j.properties
│       │   ├── producer.properties
│       │   ├── server.properties
│       │   ├── tools-log4j.properties
│       │   ├── trogdor.conf
│       │   └── zookeeper.properties
│       └── logs
└── start.sh
```

#### 配置文件

```shell
# egrep -v '^#|^$' np-kafka-01/d/kafka/config/server.properties
broker.id=1
acks=all
auto.create.topics.enable=false
default.replication.factor=3
min.insync.replicas=2
enable.auto.commit=false
listeners=SASL_PLAINTEXT://:9092
authorizer.class.name=kafka.security.auth.SimpleAclAuthorizer
sasl.enabled.mechanisms=PLAIN
security.inter.broker.protocol=SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol=PLAIN
allow.everyone.if.no.acl.found=true
keeper.set.acl=true
super.users=User:admin
advertised.listeners=SASL_PLAINTEXT://:9092
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/opt/kafka/logs
num.partitions=3
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=np-zk-01:2181,np-zk-02:2181,np-zk-03:2181
zookeeper.connection.timeout.ms=18000
group.initial.rebalance.delay.ms=0
advertised.port=9092
advertised.host.name=np-kafka-01
port=9092
```

#### 认证文件

```shell
# cat np-kafka-01/d/kafka/config/kafka_server_jaas.conf
KafkaServer {
    org.apache.kafka.common.security.plain.PlainLoginModule required
        username="admin"
        password="111111"
        user_admin="111111"
        user_kafkaDev="111111";
};
Client {
    org.apache.kafka.common.security.plain.PlainLoginModule required
    username="admin"
    password="111111";
};
```

#### 启动脚本

```shell
# cat np-kafka-01/start.sh
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
        sudo chmod g+rwx -R $(pwd)/d
        sudo chgrp 0 -R $(pwd)/d
        eval $(weave env)
        docker run -t -d -i --name ${C_NAME} --hostname ${C_NAME}.weave.local \
                --privileged \
                --restart=unless-stopped \
                --ulimit nofile=262144:262144 \
                --ulimit memlock=-1:-1 \
                --cpus 1 \
                --memory 2G \
                -v /etc/localtime:/etc/localtime \
                -v $(pwd)/d/kafka/config:/opt/kafka/config \
                -v $(pwd)/d/kafka/logs:/opt/kafka/logs \
                --env KAFKA_HEAP_OPTS=" -Xmx1024m -Xms1024m" \
                --env KAFKA_ZOOKEEPER_CONNECT=np-zk-01:2181,np-zk-02:2181,np-zk-03:2181 \
                --env KAFKA_ADVERTISED_HOST_NAME=${C_NAME} \
                --env KAFKA_ADVERTISED_PORT=9092 \
                --env KAFKA_BROKER_ID=1 \
                --env KAFKA_OPTS="-Djava.security.auth.login.config=/opt/kafka/config/kafka_server_jaas.conf" \
                --env KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.rmi.server.hostname=${C_NAME} -Dcom.sun.management.jmxremote.rmi.port=9999" \
                --env JMX_PORT=9999 \
                --env KAFKA_LOG_DIRS=/opt/kafka/logs \
                -p ${C_HOSTIP}9092:9092 \
                -p ${C_HOSTIP}9999:9999 \
                wurstmeister/kafka:2.13-2.7.0
        eval $(weave env --restore)
fi
```

#### 剩余主机配置

```shell
# grep broker.id np-kafka-02/d/kafka/config/server.properties
broker.id=2
# grep KAFKA_BROKER_ID np-kafka-02/start.sh
                --env KAFKA_BROKER_ID=2 \
```

```shell
# grep broker.id np-kafka-03/d/kafka/config/server.properties
broker.id=3
# grep KAFKA_BROKER_ID np-kafka-03/start.sh
                --env KAFKA_BROKER_ID=3 \
```

注意剩余两台主机的broker id 的不同。

### 客户端配置

注意：上面启动的kafka都是在容器中，客户端的配置在宿主上，程序解压放在/opt/ 目录下，并作软连接。

```shell
# ls /opt/kafka
bin  config  libs  LICENSE  licenses  NOTICE  site-docs
```

生产者和消费者的配置文件修改：

```shell
# tail -2 /opt/kafka/config/producer.properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
# tail -2 /opt/kafka/config/consumer.properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
```

认证用户配置文件：

```shell
# cat  /opt/kafka/config/kafka_client_jaas.conf
KafkaClient {
    org.apache.kafka.common.security.plain.PlainLoginModule required
    username="hicoreDev"
    password="111111";
};
# cat  /opt/kafka/config/kafka_server_jaas.conf
KafkaServer {
    org.apache.kafka.common.security.plain.PlainLoginModule required
        username="admin"
        password="111111"
        user_admin="111111"
        user_kafkaDev="111111";
};
Client {
    org.apache.kafka.common.security.plain.PlainLoginModule required
    username="admin"
    password="11111";
};
```

在使用的脚本中添加认证相关的配置：

```shell
# cat /opt/kafka/bin/kafka-topics.sh
...
export KAFKA_OPTS="-Djava.security.auth.login.config=/opt/kafka/config/kafka_server_jaas.conf"
...
```

```shell
# cat /opt/kafka/bin/kafka-console-producer.sh
...
export KAFKA_OPTS="-Djava.security.auth.login.config=/opt/kafka/config/kafka_client_jaas.conf"
...
```

```shell
# cat /opt/kafka/bin/kafka-console-consumer.sh
...
export KAFKA_OPTS="-Djava.security.auth.login.config=/opt/kafka/config/kafka_client_jaas.conf"
...
```

### 验证

#### 创建topic

```shell
# ./kafka-topics.sh --create --zookeeper 10.10.10.232:12181,10.10.10.233:22181,10.10.10.234:32181 --replication-factor 3 --partitions 3 --topic kafka_topic_test
WARNING: Due to limitations in metric names, topics with a period ('.') or underscore ('_') could collide. To avoid issues it is best to use either, but not both.
Created topic kafka_topic_test.
```

#### 用户权限管理

查询所有组：

```
# egrep -v "^#|^$" /opt/kafka/config/consumer.properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
```

```
# ./kafka-consumer-groups.sh --bootstrap-server 10.10.10.232:9092,10.10.10.233:9092,10.10.10.234:9092 --list --command-config /opt/kafka/config/consumer.properties
```

查询指定组详细信息：

```
# ./kafka-consumer-groups.sh --bootstrap-server 10.10.10.232:9092,10.10.10.233:9092,10.10.10.234:9092 --command-config /opt/kafka/config/consumer.properties   --describe --group gray-sender
```

添加读写权限：

```shell
# ./kafka-acls.sh --authorizer-properties zookeeper.connect=10.10.10.232:12181,10.10.10.233:22181,10.10.10.234:32181 --add --allow-principal User:hicoreDev --operation Write --operation Read --topic kafka_topic_test
Adding ACLs for resource `ResourcePattern(resourceType=TOPIC, name=kafka_topic_test, patternType=LITERAL)`:
        (principal=User:hicoreDev, host=*, operation=WRITE, permissionType=ALLOW)
        (principal=User:hicoreDev, host=*, operation=READ, permissionType=ALLOW)

Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=kafka_topic_test, patternType=LITERAL)`:
        (principal=User:hicoreDev, host=*, operation=WRITE, permissionType=ALLOW)
        (principal=User:hicoreDev, host=*, operation=READ, permissionType=ALLOW)
```

添加读权限：

```shell
# ./kafka-acls.sh --authorizer-properties zookeeper.connect=10.10.10.232:12181,10.10.10.233:22181,10.10.10.234:32181 --add --allow-principal User:hicoreDev --operation Read --topic kafka_topic_test
```

添加写权限：

```shell
# ./kafka-acls.sh --authorizer-properties zookeeper.connect=10.10.10.232:12181,10.10.10.233:22181,10.10.10.234:32181 --add --allow-principal User:hicoreDev --operation Write --topic kafka_topic_test
```

查看权限：

```shell
# ./kafka-acls.sh --list --authorizer-properties zookeeper.connect=10.10.10.232:12181,10.10.10.233:22181,10.10.10.234:32181 --topic  kafka_topic_test
Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=kafka_topic_test, patternType=LITERAL)`:
        (principal=User:hicoreDev, host=*, operation=WRITE, permissionType=ALLOW)
        (principal=User:hicoreDev, host=*, operation=READ, permissionType=ALLOW)
```

移除权限：

```shell
./kafka-acls.sh --authorizer-properties zookeeper.connect=10.10.10.232:12181,10.10.10.233:22181,10.10.10.234:32181 --remove --allow-principal User:hicoreDev --operation Read --topic kafka_topic_test
```

注意：验证完消息发送与接收以后在做权限的移除操作。

#### 消息发送接收

生产者发送消息：

```shell
# ./kafka-console-producer.sh --bootstrap-server 10.10.10.232:9092,10.10.10.233:9092,10.10.10.234:9092 --topic kafka_topic_test --producer.config /opt/kafka/config/producer.properties
>hello
>world
```

消费者接收消息：

```shell
# ./kafka-console-consumer.sh --bootstrap-server 10.10.10.232:9092,10.10.10.233:9092,10.10.10.234:9092 --topic kafka_topic_test --from-beginning --consumer.config /opt/kafka/config/consumer.properties
hello
world
```

## 使用遇到的问题

### 开发配置账号密码还是无法使用

1、在程序配置中确认是否有如下的配置

```shell
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
```

2、将kafka的主机名解析写入到本地hosts文件中。

### kafka 使用遇到如下的报错

```shell
[2021-05-19 07:12:57,779] ERROR [ReplicaManager broker=2] Error processing append operation on partition __consumer_offsets-5 (kafka.server.ReplicaManager)
org.apache.kafka.common.errors.NotEnoughReplicasException: The size of the current ISR Set(2) is insufficient to satisfy the min.isr requirement of 2 for partition __consumer_offsets-5
[2021-05-19 07:12:57,782] INFO [GroupCoordinator 2]: Preparing to rebalance group KMOffsetCache-np-kafkamanager.weave.local in state PreparingRebalance with old generation 9 (__consumer_offsets-5) (reason: error when storing group assignment during SyncGroup (member: consumer-KMOffsetCache-np-kafkamanager.weave.local-1651-685ff611-281a-40d1-84e0-7c00ba4a818a)) (kafka.coordinator.group.GroupCoordinator)
[2021-05-19 07:12:57,969] INFO [GroupCoordinator 2]: Stabilized group KMOffsetCache-np-kafkamanager.weave.local generation 10 (__consumer_offsets-5) (kafka.coordinator.group.GroupCoordinator)
[2021-05-19 07:12:57,973] INFO [GroupCoordinator 2]: Assignment received from leader for group KMOffsetCache-np-kafkamanager.weave.local for generation 10 (kafka.coordinator.group.GroupCoordinator)
[2021-05-19 07:12:57,974] ERROR [ReplicaManager broker=2] Error processing append operation on partition __consumer_offsets-5 (kafka.server.ReplicaManager)
org.apache.kafka.common.errors.NotEnoughReplicasException: The size of the current ISR Set(2) is insufficient to satisfy the min.isr requirement of 2 for partition __consumer_offsets-5
```

因为我这边kafka最初启动的参数又做了部分修改，offsets.topic.replication.factor 参数默认为1，我修改为3重启过之后报错的。在做了删除 zookeeper 中的  /brokers/topics/__consumer_offsets ，并重启 kafka 之后不在报上面的错误。具体参考如下：

https://stackoverflow.com/questions/50160474/kafka-transactionlog-fails-with-notenoughreplicasexception-despite-correct-conf/56055714



问题汇总：

https://blog.csdn.net/warrah/article/details/78909798

```
[2021-05-20 08:10:08,342] ERROR [ReplicaManager broker=1] Error processing append operation on partition hicore-gray-0 (kafka.server.ReplicaManager)
org.apache.kafka.common.errors.NotEnoughReplicasException: The size of the current ISR Set(1) is insufficient to satisfy the min.isr requirement of 2 for partition hicore-gray-0
[2021-05-20 08:10:08,449] ERROR [ReplicaManager broker=1] Error processing append operation on partition hicore-gray-0 (kafka.server.ReplicaManager)
org.apache.kafka.common.errors.NotEnoughReplicasException: The size of the current ISR Set(1) is insufficient to satisfy the min.isr requirement of 2 for partition hicore-gray-0
[2021-05-20 08:10:08,559] ERROR [ReplicaManager broker=1] Error processing append operation on partition hicore-gray-0 (kafka.server.ReplicaManager)
org.apache.kafka.common.errors.NotEnoughReplicasException: The size of the current ISR Set(1) is insufficient to satisfy the min.isr requirement of 2 for partition hicore-gray-0
[2021-05-20 08:10:08,669] ERROR [ReplicaManager broker=1] Error processing append operation on partition hicore-gray-0 (kafka.server.ReplicaManager)
org.apache.kafka.common.errors.NotEnoughReplicasException: The size of the current ISR Set(1) is insufficient to satisfy the min.isr requirement of 2 for partition hicore-gray-0
[2021-05-20 08:10:08,780] ERROR [ReplicaManager broker=1] Error processing append operation on partition hicore-gray-0 (kafka.server.ReplicaManager)
org.apache.kafka.common.errors.NotEnoughReplicasException: The size of the current ISR Set(1) is insufficient to satisfy the min.isr requirement of 2 for partition hicore-gray-0
[2021-05-20 08:10:08,890] ERROR [ReplicaManager broker=1] Error processing append operation on partition hicore-gray-0 (kafka.server.ReplicaManager)

```

