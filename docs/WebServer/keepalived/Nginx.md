## master keepalived

```
global_defs {
   router_id KEEPALIVED_NGINX
}
vrrp_script chk_mysql {
    script "/etc/keepalived/check_nginx.sh"
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
        chk_nginx
    }
    virtual_ipaddress {
        192.168.41.90
    }
}
```

## slave keepalived

```
global_defs {
   router_id KEEPALIVED_NGINX
}
vrrp_script chk_mysql {
    script "/etc/keepalived/check_nginx.sh"
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

## master check_nginx.sh

```
#!/bin/bash

NGINX_HOST='localhost'
STATUS=1
CHECK_TIME=3

function check_nginx_helth(){
    code=`curl -sIL -w "%{http_code}\n" -o /dev/null http://localhost`
    if [ ${code} -eq 200 ]; then
        STATUS=0
    else
        STATUS=1
    fi
    return $STATUS
}

while [ $CHECK_TIME -ne 0 ]
do
    let "CHECK_TIME -= 1"
    check_nginx_helth
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

NGINX_HOST='localhost'
STATUS=1
CHECK_TIME=3

function check_nginx_helth(){
    code=`curl -sIL -w "%{http_code}\n" -o /dev/null http://localhost`
    if [ ${code} -eq 200 ]; then
        STATUS=0
    else
        STATUS=1
    fi
    return $STATUS
}

while [ $CHECK_TIME -ne 0 ]
do
    let "CHECK_TIME -= 1"
    check_nginx_helth
        if
            [ $STATUS -eq 1 ] && [ $CHECK_TIME -eq 0 ]; then
            /etc/init.d/keepalived stop
            exit 1
        fi
    sleep 1
done
```

