## master Redis.conf

```
bind 0.0.0.0
protected-mode yes
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize yes
supervised no
pidfile /var/run/redis_6379.pid
loglevel notice
logfile "/data/logs/redis/redis.log"
databases 16
always-show-logo yes
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /data/redis
replica-serve-stale-data yes
replica-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
replica-priority 100
lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no
replica-lazy-flush no
appendonly no
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
aof-use-rdb-preamble yes
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
stream-node-max-bytes 4096
stream-node-max-entries 100
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
dynamic-hz yes
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes
```

## slave redis.conf

```
bind 0.0.0.0
protected-mode yes
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize yes
supervised no
pidfile /var/run/redis_6379.pid
loglevel notice
logfile "/data/logs/redis/redis.log"
databases 16
always-show-logo yes
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /data/redis
replicaof 10.1.1.32 6379
replica-serve-stale-data yes
replica-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
replica-priority 100
lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no
replica-lazy-flush no
appendonly no
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
aof-use-rdb-preamble yes
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
stream-node-max-bytes 4096
stream-node-max-entries 100
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
dynamic-hz yes
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes
```

## master keepalived 10.1.1.32

```
global_defs {
  router_id KEEPALIVED_REDIS
}
 
vrrp_script chk_redis {
  script "/etc/keepalived/redis_check.sh 127.0.0.1 6379"
  interval 3
  timeout 2
  fall 3
}
 
vrrp_instance redis {
  state BACKUP
  interface eth0
  lvs_sync_daemon_interface eth0
  virtual_router_id 202
  priority 150
  nopreempt
  advert_int 1
 
  authentication {
    auth_type PASS
    auth_pass GAOJIANREDIS
  }
 
  virtual_ipaddress {
    10.1.1.197
  }
 
  track_script {
    chk_redis
  }
 
notify_master "/etc/keepalived/redis_master.sh 127.0.0.1 10.1.1.33 6379"
notify_backup "/etc/keepalived/redis_backup.sh 127.0.0.1 10.1.1.33 6379"
notify_fault /etc/keepalived/redis_fault.sh
notify_stop /etc/keepalived/redis_stop.sh
}
```

## slave keepalived 10.1.1.33

```
global_defs {
  router_id KEEPALIVED_REDIS
}
 
vrrp_script chk_redis {
  script "/etc/keepalived/redis_check.sh 127.0.0.1 6379"
  interval 3
  timeout 2
  fall 3
}

vrrp_instance redis {
  state BACKUP                             
  interface eth0
  lvs_sync_daemon_interface eth0
  virtual_router_id 202
  priority  100             
  nopreempt
  advert_int 1
 
  authentication {
    auth_type PASS
    auth_pass GAOJIANREDIS
  }
 
  virtual_ipaddress {
    10.1.1.197
  }
 
  track_script {
  chk_redis
  }
 
notify_master "/etc/keepalived/redis_master.sh 127.0.0.1 10.1.1.32 6379"
notify_backup "/etc/keepalived/redis_backup.sh 127.0.0.1 10.1.1.32 6379"
notify_fault /etc/keepalived/redis_fault.sh
notify_stop /etc/keepalived/redis_stop.sh
}
```

## master scripts

## redis_check.sh

```
#!/bin/bash
ALIVE=`/usr/local/bin/redis-cli -h $1 -p $2 PING`
LOGFILE="/data/logs/redis/keepalived-redis-check.log"
echo "[CHECK]" >> $LOGFILE
date >> $LOGFILE
if [ $ALIVE == "PONG" ]; then :
   echo "Success: redis-cli -h $1 -p $2 PING $ALIVE" >> $LOGFILE 2>&1
    exit 0
else
    echo "Failed:redis-cli -h $1 -p $2 PING $ALIVE " >> $LOGFILE 2>&1
    exit 1
fi
```

### redis_master.sh

```
#!/bin/bash
REDISCLI="/usr/local/bin/redis-cli -h $1 -p $3"
LOGFILE="/data/logs/redis/keepalived-redis-state.log"
echo "[master]" >> $LOGFILE
date >> $LOGFILE
echo "Being master...." >> $LOGFILE 2>&1
echo "Run SLAVEOF cmd ... " >> $LOGFILE
$REDISCLI SLAVEOF $2 $3 >> $LOGFILE  2>&1
 
#echo "SLAVEOF $2 cmd can't excute ... " >> $LOGFILE
sleep 10                                               #延迟10秒以后待数据同步完成后再取消同步状态
echo "Run SLAVEOF NO ONE cmd ..." >> $LOGFILE
$REDISCLI SLAVEOF NO ONE >> $LOGFILE 2>&1
```

### redis_backup.sh

```
#!/bin/bash
REDISCLI="/usr/local/bin/redis-cli"
LOGFILE="/data/logs/redis/keepalived-redis-state.log"
echo "[BACKUP]" >> $LOGFILE
date >> $LOGFILE
echo "Being slave...." >> $LOGFILE 2>&1
echo "Run SLAVEOF cmd ..." >> $LOGFILE 2>&1
$REDISCLI SLAVEOF $2 $3 >> $LOGFILE
sleep 100                                             #延迟100秒以后待数据同步完成后再取消同步状态
exit(0)
```

### redis_fault.sh

```
#!/bin/bash
LOGFILE="/data/logs/redis/keepalived-redis-state.log"
echo "[fault]" >> $LOGFILE
date >> $LOGFILE
```

### redis_stop.sh

```
#!/bin/bash
LOGFILE="/data/logs/redis/keepalived-redis-state.log"
echo "[stop]" >> $LOGFILE
date >> $LOGFILE
```

## slave scripts

### redis_check.sh

```
#!/bin/bash
ALIVE=`/usr/local/bin/redis-cli -h $1 -p $2 PING`
LOGFILE="/data/logs/redis/keepalived-redis-check.log"
echo "[CHECK]" >> $LOGFILE
date >> $LOGFILE
if [ $ALIVE == "PONG" ]; then :
   echo "Success: redis-cli -h $1 -p $2 PING $ALIVE" >> $LOGFILE 2>&1
    exit 0
else
    echo "Failed:redis-cli -h $1 -p $2 PING $ALIVE " >> $LOGFILE 2>&1
    exit 1
fi
```

### redis_master.sh

```
#!/bin/bash
REDISCLI="/usr/local/bin/redis-cli -h $1 -p $3"
LOGFILE="/data/logs/redis/keepalived-redis-state.log"
echo "[master]" >> $LOGFILE
date >> $LOGFILE
echo "Being master...." >> $LOGFILE 2>&1
echo "Run SLAVEOF cmd ... " >> $LOGFILE
$REDISCLI SLAVEOF $2 $3 >> $LOGFILE  2>&1
 
#echo "SLAVEOF $2 cmd can't excute ... " >> $LOGFILE
sleep 10                                               #延迟10秒以后待数据同步完成后再取消同步状态
echo "Run SLAVEOF NO ONE cmd ..." >> $LOGFILE
$REDISCLI SLAVEOF NO ONE >> $LOGFILE 2>&1
```

### redis_fault.sh

```
#!/bin/bash
LOGFILE="/data/logs/redis/keepalived-redis-state.log"
echo "[fault]" >> $LOGFILE
date >> $LOGFILE
```

### redis_stop.sh 

```
#!/bin/bash
LOGFILE="/data/logs/redis/keepalived-redis-state.log"
echo "[stop]" >> $LOGFILE
date >> $LOGFILE
```

