## master my.cnf

```
[mysqld]
character_set_server=utf8
innodb_buffer_pool_size = 128G
innodb_log_file_size = 8G
max_connections = 1000
max_allowed_packet = 64M
log_bin = /data/mysql/binlogs/mysql-bin
relay_log = /data/mysql/relaylogs/relay-bin
datadir = /data/mysql/
port = 3306
server_id = 1
pid-file = /data/mysql/mysql.pid
read-only=0
log-slave-updates
log_error = error.log
slave-skip-errors=all
auto_increment_increment=2
auto_increment_offset=1
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
#sql_mode=NO_ENGINE_SUBSTITUTION,NO_AUTO_CREATE_USER
skip-name-resolve
```

## slave my.cnf

```
[mysqld]
character_set_server=utf8
innodb_buffer_pool_size = 128G
innodb_log_file_size = 8G
max_connections = 1000
max_allowed_packet = 64M
log_bin = /data/mysql/binlogs/mysql-bin
relay_log = /data/mysql/relaylogs/relay-bin
datadir = /data/mysql/
port = 3306
server_id = 98
pid-file = /data/mysql/mysql.pid
read-only=1
log-slave-updates
log_error = error.log
slave-skip-errors=all
auto_increment_increment=2
auto_increment_offset=2
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
#sql_mode=NO_ENGINE_SUBSTITUTION,NO_AUTO_CREATE_USER
skip-name-resolve
#skip-slave-start
```

## master keepalived

```
global_defs {
   router_id KEEPALIVED_MYSQL
}
vrrp_script chk_mysql {
    script "/etc/keepalived/check_mysql.sh"
    interval 5
    weight -20
    fall 3  
    rise 2
}
vrrp_instance VI_1 {
    state BACKUP
    interface bond0
    virtual_router_id 105
    priority 200
    advert_int 1
    nopreempt
    authentication {
        auth_type PASS
        auth_pass live10598
    }
    track_script {
        chk_mysql
    }
    virtual_ipaddress {
        192.168.41.90
    }
}
```

## slave keepalived

```
global_defs {
   router_id KEEPALIVED_MYSQL
}
vrrp_script chk_mysql {
    script "/etc/keepalived/check_mysql.sh"
    interval 5
    weight -20
    fall 3  
    rise 2
}
vrrp_instance VI_1 {
    state BACKUP
    interface bond0
    virtual_router_id 105
    priority 190
    advert_int 1
    nopreempt
    authentication {
        auth_type PASS
        auth_pass live10598
    }
    track_script {
        chk_mysql
    }
    virtual_ipaddress {
        192.168.41.90
    }
}
```

## master check_mysql.sh

```
#!/bin/bash

MYSQL=/usr/local/mysql/bin/mysql
USER='readonly'
PASSWORD='xxxxxxx'
MYSQL_HOST='192.168.41.105'
STATUS=1
CHECK_TIME=3

function check_mysql_helth(){
    $MYSQL -u${USER} -h ${MYSQL_HOST} -p${PASSWORD} -e "show status;" >/dev/null 2>&1
    if [ $? -eq 0 ]; then
        STATUS=0
    else
        STATUS=1
    fi
    return $STATUS
}

while [ $CHECK_TIME -ne 0 ]
do
    let "CHECK_TIME -= 1"
    check_mysql_helth
        if
            [ $STATUS -eq 1 ] && [ $CHECK_TIME -eq 0 ]; then
            /etc/init.d/keepalived stop
            exit 1
        fi
    sleep 1
done
```

## slave check_mysql.sh

```
#!/bin/bash

MYSQL=/usr/local/mysql/bin/mysql
USER='readonly'
PASSWORD='xxxxxx'
MYSQL_HOST='192.168.41.98'
STATUS=1
CHECK_TIME=3

function check_mysql_helth(){
    $MYSQL -u${USER} -h ${MYSQL_HOST} -p${PASSWORD} -e "show status;" >/dev/null 2>&1
    if [ $? -eq 0 ]; then
        STATUS=0
    else
        STATUS=1
    fi
    return $STATUS
}

while [ $CHECK_TIME -ne 0 ]
do
    let "CHECK_TIME -= 1"
    check_mysql_helth
        if
            [ $STATUS -eq 1 ] && [ $CHECK_TIME -eq 0 ]; then
            /etc/init.d/keepalived stop
            exit 1
        fi
    sleep 1
done
```

