## 基本概念

* Pod/Pod 控制器
* Name/Namespace
* Label/Label 选择器
* Service/Ingress





## DNS

```
yum install -y wget net-tools telnet tree nmap sysstat lrzsz doc2unix bind-utils bind
```



```
...
options {
        listen-on port 53 { 192.168.23.192; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query     { any; };         # 修改
        forwarders      { 172.32.0.2; };  # 修改
        recursion yes;
        dnssec-enable no;                 # 修改
        dnssec-validation no;             # 修改
        bindkeys-file "/etc/named.root.key";
        managed-keys-directory "/var/named/dynamic";
        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};
]# named-checkconf
```



```
]# vim /etc/named.rfc1912.zones 
...
zone "host.com" IN {
        type  master;
        file  "host.com.zone";
        allow-update { 192.168.23.192; };

};

zone "k8s.com" IN {
        type  master;
        file  "k8s.com.zone";
        allow-update { 192.168.23.192; };
};
```

```
]# vim /var/named/host.com.zone
$ORIGIN host.com.
$TTL 600  ; 10 minutes
@       IN SOA  dns.host.com. dnsadmin.host.com. (
        2020111601 ; serial
        10800      ; refresh (3 hours)
        900        ; retry (15 minutes)
        604800     ; expire (1 week)
        86400      ; minimum (1 day)
        )
      NS   dns.host.com.
$TTL 60 ; 1 minute
dns                A    192.168.23.192
ctrl-122-81        A    192.168.122.81
ctrl-122-82        A    192.168.122.82
ctrl-122-83        A    192.168.122.83
node-122-84        A    192.168.122.84
node-122-85        A    192.168.122.85
```



```
]# vim /var/named/k8s.com.zone
$ORIGIN k8s.com.
$TTL 600  ; 10 minutes
@       IN SOA  dns.k8s.com. dnsadmin.k8s.com. (
        2020111601 ; serial
        10800      ; refresh (3 hours)
        900        ; retry (15 minutes)
        604800     ; expire (1 week)
        86400      ; minimum (1 day)
        )
        NS   dns.k8s.com.
$TTL 60 ; 1 minute
dns                A    192.168.23.192
```

```
]# named-checkconf 
]# systemctl start named
]# ss -tnlp |grep named
LISTEN     0      10     192.168.23.192:53                       *:*                   users:(("named",pid=99781,fd=21))
LISTEN     0      128    127.0.0.1:953                      *:*                   users:(("named",pid=99781,fd=23))
LISTEN     0      10         ::1:53                      :::*                   users:(("named",pid=99781,fd=22))
LISTEN     0      128        ::1:953                     :::*                   users:(("named",pid=99781,fd=24))
]# dig -t A ctrl-122-81.host.com @192.168.23.192  +short 
192.168.122.81
```



```
]# cat >/etc/resolv.conf<<EOF
search host.com
nameserver 192.168.23.192
EOF
```



```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -O /usr/local/bin/cfssl
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -O /usr/local/bin/cfssl-json
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -O /usr/local/bin/cfssl-certinfo
chmod u+x /usr/local/bin/cfssl*
```



```
{
    "CN": "Kubernetes",
    "hosts": [
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Shanghai",
            "L": "Shanghai",
            "O": "k8s",
            "OU": "ops"
        }
    ],
    "ca": {
        "expiry": "175200h"
    }
}
```

CN：Common Name，浏览器使用该字段验证网站是否合法，一般写的是域名。

C：Country，国家

ST：State，州，省

L：Locality，地区，城市

O：Organization Name，组织名称或者公司名称

OU：Organization Unit Name：组织单位名称或者公司部门

expiry：证书有效时间

```
]# cd /opt/certs/
certs]# cfssl gencert -initca ca-csr.json  |cfssl-json -bare ca
certs]# ll
total 16
-rw-r--r-- 1 root root 1001 Nov 17 22:46 ca.csr
-rw-r--r-- 1 root root  332 Nov 17 22:46 ca-csr.json
-rw------- 1 root root 1675 Nov 17 22:46 ca-key.pem
-rw-r--r-- 1 root root 1354 Nov 17 22:46 ca.pem
```

安装docker

```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum -y install docker-ce
```

```
{
  "graph": "/data/docker",
  "storage-driver": "overlay2",
  "insecure-registries": ["registry.access.redhat.com","quay.io","harbor.k8s.com"],
  "registry-mirrors": ["https://registry.docker-cn.com"],
  "bip": "172.7.81.1/24",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "live-restore": true
}
```







问题：

```
---> Package docker-ce-cli.x86_64 1:19.03.13-3.el7 will be installed
---> Package libcgroup.x86_64 0:0.41-20.el7 will be installed
--> Finished Dependency Resolution
Error: Package: 3:docker-ce-19.03.13-3.el7.x86_64 (docker-ce-stable)
           Requires: container-selinux >= 2:2.74
Error: Package: containerd.io-1.3.7-3.1.el7.x86_64 (docker-ce-stable)
           Requires: container-selinux >= 2:2.74
 You could try using --skip-broken to work around the problem
 You could try running: rpm -Va --nofiles --nodigest
```

解决：

```
wget https://mirrors.aliyun.com/centos/7.8.2003/extras/x86_64/Packages/container-selinux-2.119.2-1.911c772.el7_8.noarch.rpm
yum install -y container-selinux-2.119.2-1.911c772.el7_8.noarch.rpm
```





etcd

```
{
    "signing": {
        "default": {
            "expiry": "175200h"
        },
        "profiles": {
            "server": {
                "expiry": "175200h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "175200h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
            "peer": {
                "expiry": "175200h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
```



```
{
    "CN": "k8s-etcd",
    "hosts": [
        "192.168.122.81",
        "192.168.122.82",
        "192.168.122.83",
        "192.168.122.84"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Shanghai",
            "L": "Shanghai",
            "O": "k8s",
            "OU": "ops"
        }
    ]
}
```

```
certs]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer etcd-peer-csr.json |cfssl-json  -bare etcd-peer
```

```
[root@hdss7-12 ~]# vim /opt/etcd/etcd-server-startup.sh
#!/bin/sh

/opt/etcd/etcd --name etcd-server-122-81 \
    --data-dir /data/etcd/etcd-server \
    --listen-peer-urls https://192.168.122.81:2380 \
    --listen-client-urls https://192.168.122.81:2379,http://127.0.0.1:2379 \
    --quota-backend-bytes 8000000000 \
    --initial-advertise-peer-urls https://192.168.122.81:2380 \
    --advertise-client-urls https://192.168.122.81:2379,http://127.0.0.1:2379 \
    --initial-cluster  etcd-server-122-81=https://192.168.122.81:2380,etcd-server-122-82=https://192.168.122.82:2380,etcd-server-122-83=https://192.168.122.83:2380 \
    --ca-file ./certs/ca.pem \
    --cert-file ./certs/etcd-peer.pem \
    --key-file ./certs/etcd-peer-key.pem \
    --client-cert-auth  \
    --trusted-ca-file ./certs/ca.pem \
    --peer-ca-file ./certs/ca.pem \
    --peer-cert-file ./certs/etcd-peer.pem \
    --peer-key-file ./certs/etcd-peer-key.pem \
    --peer-client-cert-auth \
    --peer-trusted-ca-file ./certs/ca.pem \
    --log-output stdout
```



```
yum install -y supervisor
```

```
]# vim /etc/supervisord.d/etcd-server.ini
[program:etcd-server-122-81]
command=/opt/etcd/etcd-server-startup.sh              ; the program (relative uses PATH, can take args)
numprocs=1                                            ; number of processes copies to start (def 1)
directory=/opt/etcd                                   ; directory to cwd to before exec (def no cwd)
autostart=true                                        ; start at supervisord start (default: true)
autorestart=true                                      ; retstart at unexpected quit (default: true)
startsecs=30                                          ; number of secs prog must stay running (def. 1)
startretries=3                                        ; max # of serial start failures (default 3)
exitcodes=0,2                                         ; 'expected' exit codes for process (default 0,2)
stopsignal=QUIT                                       ; signal used to kill process (default TERM)
stopwaitsecs=10                                       ; max num secs to wait b4 SIGKILL (default 10)
user=etcd                                             ; setuid to this UNIX account to run the program
redirect_stderr=true                                  ; redirect proc stderr to stdout (default false)
stdout_logfile=/data/logs/etcd-server/etcd.stdout.log ; stdout log path, NONE for none; default AUTO
stdout_logfile_maxbytes=64MB                          ; max # logfile bytes b4 rotation (default 50MB)
stdout_logfile_backups=5                              ; # of stdout logfile backups (default 10)
stdout_capture_maxbytes=1MB                           ; number of bytes in 'capturemode' (default 0)
stdout_events_enabled=false                           ; emit events on stdout writes (default false)

]# tar -xf etcd-v3.1.20-linux-amd64.tar.gz -C /opt/
]# cd /opt/
]# ln -sv etcd-v3.1.20-linux-amd64 etcd
]# useradd -s /sbin/nologin -M etcd
]# mkdir -p /data/etcd /data/logs/etcd-server /opt/etcd/certs
]# chown -R etcd.etcd /opt/etcd/* /data/etcd/
]# ll /opt/etcd/certs/
total 12
-rw-r--r-- 1 etcd etcd 1354 Nov 18 17:27 ca.pem
-rw------- 1 etcd etcd 1675 Nov 18 17:27 etcd-peer-key.pem
-rw-r--r-- 1 etcd etcd 1436 Nov 18 17:27 etcd-peer.pem
]# systemctl start supervisord
]# supervisorctl update
etcd-server-122-81: added process group
```



```
]# /opt/etcd/etcdctl cluster-health
member 2d04c59b080372f6 is healthy: got healthy result from http://127.0.0.1:2379
member 4a8a64920aff7586 is healthy: got healthy result from http://127.0.0.1:2379
member d0d72fe80decc78d is healthy: got healthy result from http://127.0.0.1:2379
cluster is healthy
]# /opt/etcd/etcdctl member list
2d04c59b080372f6: name=etcd-server-122-82 peerURLs=https://192.168.122.82:2380 clientURLs=http://127.0.0.1:2379,https://192.168.122.82:2379 isLeader=false
4a8a64920aff7586: name=etcd-server-122-81 peerURLs=https://192.168.122.81:2380 clientURLs=http://127.0.0.1:2379,https://192.168.122.81:2379 isLeader=true
d0d72fe80decc78d: name=etcd-server-122-83 peerURLs=https://192.168.122.83:2380 clientURLs=http://127.0.0.1:2379,https://192.168.122.83:2379 isLeader=false
```

api-server

```
certs]# vim client-csr.json
{
    "CN": "k8s-node",
    "hosts": [
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Shanghai",
            "L": "Shanghai",
            "O": "k8s",
            "OU": "ops"
        }
    ]
}
certs]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client client-csr.json |cfssl-json -bare client 
certs]# ll client*
-rw-r--r-- 1 root root  997 Dec  1 13:48 client.csr
-rw-r--r-- 1 root root  283 Dec  1 13:42 client-csr.json
-rw------- 1 root root 1675 Dec  1 13:48 client-key.pem
-rw-r--r-- 1 root root 1375 Dec  1 13:48 client.pem
```

```
certs]# vim apiserver-csr.json
{
    "CN": "k8s-apiserver",
    "hosts": [
        "127.0.0.1",
        "192.168.0.1",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster",
        "kubernetes.default.svc.cluster.local",
        "192.168.122.10",
        "192.168.122.81",
        "192.168.122.82",
        "192.168.122.83"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Shanghai",
            "L": "Shanghai",
            "O": "k8s",
            "OU": "ops"
        }
    ]
}
certs]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server apiserver-csr.json |cfssl-json -bare apiserver 
certs]# ls apiserver* -l
-rw-r--r-- 1 root root 1253 Dec  1 14:01 apiserver.csr
-rw-r--r-- 1 root root  589 Dec  1 13:59 apiserver-csr.json
-rw------- 1 root root 1679 Dec  1 14:01 apiserver-key.pem
-rw-r--r-- 1 root root 1606 Dec  1 14:01 apiserver.pem
```



```
]# tar -xf /tmp/kubernetes-server-linux-amd64.tar.gz -C /opt/
]# chown -R root.root /opt/kubernetes/
]# cd /opt/kubernetes/server/bin/
bin]# rm -f *.tar *_tag
bin]# mkdir certs
```



```
certs]# for i in 81 82 83;do echo ctrl-122-$i;ssh ctrl-122-$i "mkdir /opt/kubernetes/server/bin/certs";scp ca.pem  ca-key.pem client.pem client-key.pem apiserver.pem apiserver-key.pem ctrl-122-$i:/opt/kubernetes/server/bin/certs/;done
certs]# ssh ctrl-122-81 'ls -l  /opt/kubernetes/server/bin/certs '      
total 24
-rw------- 1 root root 1679 Dec  1 15:23 apiserver-key.pem
-rw-r--r-- 1 root root 1606 Dec  1 15:23 apiserver.pem
-rw-r--r-- 1 root root 1675 Dec  1 15:23 ca-key.pem
-rw-r--r-- 1 root root 1354 Dec  1 15:23 ca.pem
-rw------- 1 root root 1675 Dec  1 15:23 client-key.pem
-rw-r--r-- 1 root root 1375 Dec  1 15:23 client.pem
```



```
]# vi audit.yaml 
apiVersion: audit.k8s.io/v1beta1 # This is required.
kind: Policy
# Don't generate audit events for all requests in RequestReceived stage.
omitStages:
  - "RequestReceived"
rules:
  # Log pod changes at RequestResponse level
  - level: RequestResponse
    resources:
    - group: ""
      # Resource "pods" doesn't match requests to any subresource of pods,
      # which is consistent with the RBAC policy.
      resources: ["pods"]
  # Log "pods/log", "pods/status" at Metadata level
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log", "pods/status"]

  # Don't log requests to a configmap called "controller-leader"
  - level: None
    resources:
    - group: ""
      resources: ["configmaps"]
      resourceNames: ["controller-leader"]

  # Don't log watch requests by the "system:kube-proxy" on endpoints or services
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: "" # core API group
      resources: ["endpoints", "services"]

  # Don't log authenticated requests to certain non-resource URL paths.
  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
    - "/api*" # Wildcard matching.
    - "/version"

  # Log the request body of configmap changes in kube-system.
  - level: Request
    resources:
    - group: "" # core API group
      resources: ["configmaps"]
    # This rule only applies to resources in the "kube-system" namespace.
    # The empty string "" can be used to select non-namespaced resources.
    namespaces: ["kube-system"]

  # Log configmap and secret changes in all other namespaces at the Metadata level.
  - level: Metadata
    resources:
    - group: "" # core API group
      resources: ["secrets", "configmaps"]

  # Log all other resources in core and extensions at the Request level.
  - level: Request
    resources:
    - group: "" # core API group
    - group: "extensions" # Version of group should NOT be included.

  # A catch-all rule to log all other requests at the Metadata level.
  - level: Metadata
    # Long-running requests like watches that fall under this rule will not
    # generate an audit event in RequestReceived.
    omitStages:
      - "RequestReceived"
]# ansible k8s_ctrl -m copy -a "src=audit.yaml dest=/opt/kubernetes/server/bin/conf/"  
```



```
]# vim kube-apiserver.sh
#!/bin/bash
./kube-apiserver \
    --apiserver-count 2 \
    --audit-log-path /data/logs/kubernetes/kube-apiserver/audit-log \
    --audit-policy-file ./conf/audit.yaml \
    --authorization-mode RBAC \
    --client-ca-file ./certs/ca.pem \
    --requestheader-client-ca-file ./certs/ca.pem \
    --enable-admission-plugins NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota \
    --etcd-cafile ./certs/ca.pem \
    --etcd-certfile ./certs/client.pem \
    --etcd-keyfile ./certs/client-key.pem \
    --etcd-servers https://192.168.122.81:2379,https://192.168.122.82:2379,https://192.168.122.83:2379 \
    --service-account-key-file ./certs/ca-key.pem \
    --service-cluster-ip-range 192.168.0.0/16 \
    --service-node-port-range 3000-29999 \
    --target-ram-mb=1024 \
    --kubelet-client-certificate ./certs/client.pem \
    --kubelet-client-key ./certs/client-key.pem \
    --log-dir  /data/logs/kubernetes/kube-apiserver \
    --tls-cert-file ./certs/apiserver.pem \
    --tls-private-key-file ./certs/apiserver-key.pem \
    --v 2
]# ansible k8s_ctrl -m copy -a "src=kube-apiserver.sh dest=/opt/kubernetes/server/bin/"
]# ansible k8s_ctrl -m file -a "path=/opt/kubernetes/server/bin/kube-apiserver.sh mode=755"
```

```
]# ansible k8s_ctrl -m file -a "path=/data/logs/kubernetes/kube-apiserver state=directory mode='0755'"
```

```
[root@ctrl-122-81 ~]# vim /etc/supervisord.d/kube-apiserver.ini
[program:kube-apiserver-122-81]
command=/opt/kubernetes/server/bin/kube-apiserver.sh
numprocs=1
directory=/opt/kubernetes/server/bin
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kube-apiserver/apiserver.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=5
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
[root@ctrl-122-81 ~]# supervisorctl update
kube-apiserver-122-81: added process group
[root@ctrl-122-81 ~]# supervisorctl status
etcd-server-122-81               RUNNING   pid 21002, uptime 6 days, 1:24:30
kube-apiserver-122-81            RUNNING   pid 1619, uptime 0:00:56
```

nginx+keepalived

```
stream {
    log_format proxy '$time_local|$remote_addr|$upstream_addr|$protocol|$status|'
                     '$session_time|$upstream_connect_time|$bytes_sent|$bytes_received|'
                     '$upstream_bytes_sent|$upstream_bytes_received' ;

    upstream kube-apiserver {
        server 192.168.122.81:6443     max_fails=3 fail_timeout=30s;
        server 192.168.122.82:6443     max_fails=3 fail_timeout=30s;
        server 192.168.122.83:6443     max_fails=3 fail_timeout=30s;
    }
    server {
        listen 7443;
        proxy_connect_timeout 2s;
        proxy_timeout 900s;
        proxy_pass kube-apiserver;
        access_log /var/log/nginx/proxy.log proxy;
    }
}
```

```
keepalived]# cat check_process.sh 
#!/bin/bash
CHECK_PROCESS=nginx
STATUS=1
CHECK_TIME=3
function check_process_helth(){
    status=`ps aux|grep $CHECK_PROCESS | grep -v grep | grep -v bash | wc -l`
    if [ $status -eq 0 ]; then
        STATUS=0
    else
        STATUS=1
    fi
    return $STATUS
}
while [ $CHECK_TIME -ne 0 ]
do
    let "CHECK_TIME -= 1"
    check_process_helth
        if
            [ $STATUS -eq 0 ] && [ $CHECK_TIME -eq 0 ]; then
            systemctl stop keepalived
            exit 1
        fi
    sleep 1
done
```

```
keepalived]# cat keepalived.conf 
! Configuration File for keepalived
global_defs {
   router_id 192.168.122.10
}
vrrp_script monitor {
   script "/etc/keepalived/check_process.sh"
   interval 3
   weight  -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 240
    priority 150
    advert_int 1
    nopreempt
    authentication {
        auth_type PASS
        auth_pass 111qaz
    }
    track_script {
        monitor
    }
    virtual_ipaddress {
        192.168.122.10
    }
}
```

```
keepalived]# cat keepalived.conf 
! Configuration File for keepalived
global_defs {
   router_id 192.168.122.10
}
vrrp_script monitor {
   script "/etc/keepalived/check_process.sh"
   interval 5
   weight  -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 240
    priority 100
    advert_int 1
    nopreempt
    authentication {
        auth_type PASS
        auth_pass 111qaz
    }
    track_script {
        monitor
    }
    virtual_ipaddress {
        192.168.122.10
    }
}
```

### controller-manager

```

bin]# vim /opt/kubernetes/server/bin/kube-controller-manager.sh
#!/bin/sh
./kube-controller-manager \
    --cluster-cidr 172.7.0.0/16 \
    --leader-elect true \
    --log-dir /data/logs/kubernetes/kube-controller-manager \
    --master http://127.0.0.1:8080 \
    --service-account-private-key-file ./certs/ca-key.pem \
    --service-cluster-ip-range 192.168.0.0/16 \
    --root-ca-file ./certs/ca.pem \
    --v 2
```

```
]# vim /etc/supervisord.d/kube-controller-manager.ini
[program:kube-controller-manager-122-81]
command=/opt/kubernetes/server/bin/kube-controller-manager.sh                     ; the program (relative uses PATH, can take args)
numprocs=1                                                                        ; number of processes copies to start (def 1)
directory=/opt/kubernetes/server/bin                                              ; directory to cwd to before exec (def no cwd)
autostart=true                                                                    ; start at supervisord start (default: true)
autorestart=true                                                                  ; retstart at unexpected quit (default: true)
startsecs=30                                                                      ; number of secs prog must stay running (def. 1)
startretries=3                                                                    ; max # of serial start failures (default 3)
exitcodes=0,2                                                                     ; 'expected' exit codes for process (default 0,2)
stopsignal=QUIT                                                                   ; signal used to kill process (default TERM)
stopwaitsecs=10                                                                   ; max num secs to wait b4 SIGKILL (default 10)
user=root                                                                         ; setuid to this UNIX account to run the program
redirect_stderr=true                                                              ; redirect proc stderr to stdout (default false)
stdout_logfile=/data/logs/kubernetes/kube-controller-manager/controller.stdout.log  ; stderr log path, NONE for none; default AUTO
stdout_logfile_maxbytes=64MB                                                      ; max # logfile bytes b4 rotation (default 50MB)
stdout_logfile_backups=4                                                          ; # of stdout logfile backups (default 10)
stdout_capture_maxbytes=1MB                                                       ; number of bytes in 'capturemode' (default 0)
stdout_events_enabled=false                                                       ; emit events on stdout writes (default false)
```

```
bin]# chmod +x /opt/kubernetes/server/bin/kube-controller-manager.sh
bin]# mkdir -p /data/logs/kubernetes/kube-controller-manager/
bin]# supervisorctl update
bin]# supervisorctl status                                   
etcd-server-122-81               RUNNING   pid 21002, uptime 7 days, 1:23:58
kube-apiserver-122-81            RUNNING   pid 1619, uptime 1 day, 0:00:24
kube-controller-manager-122-81   RUNNING   pid 3685, uptime 0:02:10
```

### kube-scheduler

```
bin]# vim /opt/kubernetes/server/bin/kube-scheduler.sh
#!/bin/sh
./kube-scheduler \
    --leader-elect  \
    --log-dir /data/logs/kubernetes/kube-scheduler \
    --master http://127.0.0.1:8080 \
    --v 2
```

```
bin]# vim /etc/supervisord.d/kube-scheduler.ini
[program:kube-scheduler-122-81]
command=/opt/kubernetes/server/bin/kube-scheduler.sh                     
numprocs=1                                                               
directory=/opt/kubernetes/server/bin                                     
autostart=true                                                           
autorestart=true                                                         
startsecs=30                                                             
startretries=3                                                           
exitcodes=0,2                                                            
stopsignal=QUIT                                                          
stopwaitsecs=10                                                          
user=root                                                                
redirect_stderr=true                                                     
stdout_logfile=/data/logs/kubernetes/kube-scheduler/scheduler.stdout.log 
stdout_logfile_maxbytes=64MB                                             
stdout_logfile_backups=4                                                 
stdout_capture_maxbytes=1MB                                              
stdout_events_enabled=false 
```

```
bin]# chmod +x /opt/kubernetes/server/bin/kube-scheduler.sh
bin]# mkdir -p /data/logs/kubernetes/kube-scheduler/
bin]# supervisorctl update
bin]# supervisorctl status
etcd-server-122-81               RUNNING   pid 21002, uptime 7 days, 1:49:11
kube-apiserver-122-81            RUNNING   pid 1619, uptime 1 day, 0:25:37
kube-controller-manager-122-81   RUNNING   pid 3685, uptime 0:27:23
kube-scheduler-122-81            RUNNING   pid 3746, uptime 0:02:11
```

```
# ln -sv /opt/kubernetes/server/bin/kubectl /usr/bin/kubectl
]# kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}  
```



## 部署node

```
[root@node-122-85 ~]# tar -xf /tmp/kubernetes-server-linux-amd64.tar.gz -C /opt/
]# cd /opt/kubernetes/server/bin/
bin]# rm -f *.tar *_tag
bin]# ln -sv /opt/kubernetes/server/bin/kubectl /usr/bin/kubectl
```



```
certs]# vim kubelet-csr.json
{
    "CN": "k8s-kubelet",
    "hosts": [
    "127.0.0.1",
    "192.168.122.71",
    "192.168.122.72",
    "192.168.122.81",
    "192.168.122.82",
    "192.168.122.83",
    "192.168.122.84",
    "192.168.122.85",
    "192.168.122.86",
    "192.168.122.87",
    "192.168.122.88",
    "192.168.122.89",
    "192.168.122.90"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Shanghai",
            "L": "Shanghai",
            "O": "k8s",
            "OU": "ops"
        }
    ]
}
```

```
certs]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server kubelet-csr.json |cfssl-json -bare kubelet
certs]# ll kubelet*
-rw-r--r-- 1 root root 1143 Dec  3 11:08 kubelet.csr
-rw-r--r-- 1 root root  566 Dec  3 10:02 kubelet-csr.json
-rw------- 1 root root 1679 Dec  3 11:08 kubelet-key.pem
-rw-r--r-- 1 root root 1501 Dec  3 11:08 kubelet.pem
certs]# ansible k8s_node -m copy -a "src=./kubelet-key.pem dest=/opt/kubernetes/server/bin/certs/"
```

生成kubelet 配置文件



```
certs]# ansible k8s_node -m copy -a "src=./ca.pem dest=/opt/kubernetes/server/bin/certs/"
certs]# ansible k8s_node -m copy -a "src=./client.pem dest=/opt/kubernetes/server/bin/certs/"
certs]# ansible k8s_node -m copy -a "src=./client-key.pem dest=/opt/kubernetes/server/bin/certs/"
```



```
[root@node-122-84 ~]# kubectl config set-cluster myk8s \
 --certificate-authority=/opt/kubernetes/server/bin/certs/ca.pem \
 --embed-certs=true \
 --server=https://192.168.122.10:7443 \
 --kubeconfig=/opt/kubernetes/conf/kubelet.kubeconfig
Cluster "myk8s" set.
[root@node-122-84 ~]# kubectl config set-credentials k8s-node \
 --client-certificate=/opt/kubernetes/server/bin/certs/client.pem \
 --client-key=/opt/kubernetes/server/bin/certs/client-key.pem \
 --embed-certs=true \
 --kubeconfig=/opt/kubernetes/conf/kubelet.kubeconfig
User "k8s-node" set.
[root@node-122-84 ~]# kubectl config set-context myk8s-context \
 --cluster=myk8s \
 --user=k8s-node \
 --kubeconfig=/opt/kubernetes/conf/kubelet.kubeconfig
Context "myk8s-context" created.
[root@node-122-84 ~]# kubectl config use-context myk8s-context --kubeconfig=/opt/kubernetes/conf/kubelet.kubeconfig
Switched to context "myk8s-context".
```

复制 kubelet.kubeconfig 文件到122.85：

```
[root@node-122-85 ~]# md5sum /opt/kubernetes/conf/kubelet.kubeconfig 
cb33213de4e2dc2b43c03ee03bb76e8e  /opt/kubernetes/conf/kubelet.kubeconfig
```

在122.81授权用户，如果运算节点和控制节点分开，应该不需要这步。

```
# vim k8s-node.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: k8s-node
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:node
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: k8s-node
# kubectl create -f k8s-node.yaml 
# kubectl get clusterrolebinding k8s-node
NAME       AGE
k8s-node   26s
```

pause 镜像

```
# docker pull kubernetes/pause
# docker tag kubernetes/pause:latest harbor.k8s.com:8080/public/pause:latest
# docker login -u admin harbor.k8s.com:8080
# docker push harbor.k8s.com:8080/public/pause:latest 
```

122.84 122.85 配置并启动kubelet

```
certs]# ansible k8s_node -m copy -a "src=./kubelet.pem dest=/opt/kubernetes/server/bin/certs/"
# ansible k8s_node -m file -a "path=/data/kubelet state=directory mode='0755'"
# ansible k8s_node -m file -a "path=/data/logs/kubernetes/kube-kubelet state=directory mode='0755'"
```

```
# vim /opt/kubernetes/server/bin/kubelet.sh
#!/bin/sh
./kubelet \
    --anonymous-auth=false \
    --cgroup-driver systemd \
    --cluster-dns 192.168.0.2 \
    --cluster-domain cluster.local \
    --runtime-cgroups=/systemd/system.slice \
    --kubelet-cgroups=/systemd/system.slice \
    --fail-swap-on="false" \
    --client-ca-file ./certs/ca.pem \
    --tls-cert-file ./certs/kubelet.pem \
    --tls-private-key-file ./certs/kubelet-key.pem \
    --hostname-override node-122-84.host.com \
    --image-gc-high-threshold 20 \
    --image-gc-low-threshold 10 \
    --kubeconfig ../../conf/kubelet.kubeconfig \
    --log-dir /data/logs/kubernetes/kube-kubelet \
    --pod-infra-container-image harbor.k8s.com:8080/public/pause:latest \
    --root-dir /data/kubelet
# vim /etc/supervisord.d/kube-kubelet.ini
[program:kube-kubelet-122-84]
command=/opt/kubernetes/server/bin/kubelet.sh
numprocs=1
directory=/opt/kubernetes/server/bin
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kube-kubelet/kubelet.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=5
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
# chmod +x /opt/kubernetes/server/bin/kubelet.sh
# supervisorctl update
# supervisorctl status
kube-kubelet-122-84              RUNNING   pid 13916, uptime 0:01:09
```

在控制节点（81/82/83）执行如下命令可以查看机器状态

```
# kubectl get nodes
NAME                   STATUS   ROLES    AGE    VERSION
node-122-84.host.com   Ready    <none>   106s   v1.15.12
node-122-85.host.com   Ready    <none>   52s    v1.15.12
```

修改node节点的ROLES：

```
# kubectl label node node-122-84.host.com node-role.kubernetes.io/node=
# kubectl label node node-122-85.host.com node-role.kubernetes.io/node=
```

### kube-proxy部署

```
certs]# vim kube-proxy-csr.json
{
    "CN": "system:kube-proxy",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Shanghai",
            "L": "Shanghai",
            "O": "k8s",
            "OU": "ops"
        }
    ]
}
certs]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client kube-proxy-csr.json |cfssl-json -bare kube-proxy-client 
certs]# ll kube-proxy-*
-rw-r--r-- 1 root root 1009 Dec  7 16:50 kube-proxy-client.csr
-rw------- 1 root root 1679 Dec  7 16:50 kube-proxy-client-key.pem
-rw-r--r-- 1 root root 1387 Dec  7 16:50 kube-proxy-client.pem
-rw-r--r-- 1 root root  270 Dec  7 16:48 kube-proxy-csr.json
certs]# ansible k8s_node -m copy -a "src=./kube-proxy-client.pem dest=/opt/kubernetes/server/bin/certs/"
```

控制节点生成证书

```
# kubectl config set-cluster myk8s \
--certificate-authority=/opt/kubernetes/server/bin/certs/ca.pem \
--embed-certs=true \
--server=https://192.168.122.10:7443 \
--kubeconfig=/opt/kubernetes/conf/kube-proxy.kubeconfig
# kubectl config set-credentials kube-proxy \
--client-certificate=/opt/kubernetes/server/bin/certs/kube-proxy-client.pem \
--client-key=/opt/kubernetes/server/bin/certs/kube-proxy-client-key.pem \
--embed-certs=true \
--kubeconfig=/opt/kubernetes/conf/kube-proxy.kubeconfig
# kubectl config set-context myk8s-context \
--cluster=myk8s \
--user=kube-proxy \
--kubeconfig=/opt/kubernetes/conf/kube-proxy.kubeconfig
# kubectl config use-context myk8s-context --kubeconfig=/opt/kubernetes/conf/kube-proxy.kubeconfig
certs]# cd /opt/kubernetes/conf/
conf]# ansible k8s_node -m copy -a "src=./kube-proxy.kubeconfig dest=/opt/kubernetes/conf/"
```

运算节点加载ipvs模块

```
# for i in $(ls /usr/lib/modules/$(uname -r)/kernel/net/netfilter/ipvs|grep -o "^[^.]*");do echo $i; /sbin/modinfo -F filename $i >/dev/null 2>&1 && /sbin/modprobe $i;done
# lsmod  |grep '^ip_vs'
```

node节点创建启动脚本

```
# vim /opt/kubernetes/server/bin/kube-proxy.sh
#!/bin/sh
./kube-proxy \
  --cluster-cidr 172.7.0.0/16 \
  --hostname-override node-122-84.host.com \
  --proxy-mode=ipvs \
  --ipvs-scheduler=nq \
  --kubeconfig ../../conf/kube-proxy.kubeconfig
# chmod +x /opt/kubernetes/server/bin/kube-proxy.sh
# mkdir -p /data/logs/kubernetes/kube-proxy
# vim /etc/supervisord.d/kube-proxy.ini
[program:kube-proxy-122-84]
command=/opt/kubernetes/server/bin/kube-proxy.sh
numprocs=1
directory=/opt/kubernetes/server/bin
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kube-proxy/proxy.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=5
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
# supervisorctl update
# supervisorctl status
kube-kubelet-122-84              RUNNING   pid 13916, uptime 3 days, 0:09:03
kube-proxy-122-84                RUNNING   pid 14566, uptime 0:00:31
```

验证集群

```
# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.0.1:443 nq
  -> 192.168.122.81:6443          Masq    1      0          0         
  -> 192.168.122.82:6443          Masq    1      0          0         
  -> 192.168.122.83:6443          Masq    1      0          0      
```

验证服务

```
yaml]# cat nginx-ds.yaml 
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: nginx-ds
spec:
  template:
    metadata:
      labels:
        app: nginx-ds
    spec:
      containers:
      - name: my-nginx
        image: harbor.k8s.com:8080/public/nginx:v1.18.0
        ports:
        - containerPort: 80
# kubectl create -f nginx-ds.yaml 
daemonset.extensions/nginx-ds created
# kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
nginx-ds-kmrnx   1/1     Running   0          47s
nginx-ds-tlhqr   1/1     Running   0          47s
# kubectl get pods -o wide
NAME             READY   STATUS    RESTARTS   AGE   IP           NODE                   NOMINATED NODE   READINESS GATES
nginx-ds-kmrnx   1/1     Running   0          73s   172.7.85.2   node-122-85.host.com   <none>           <none>
nginx-ds-tlhqr   1/1     Running   0          73s   172.7.84.2   node-122-84.host.com   <none>           <none>
```



```
[root@node-122-85 ~]# curl  -I http://172.7.85.2
HTTP/1.1 200 OK
Server: nginx/1.18.0
Date: Mon, 07 Dec 2020 09:31:32 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 21 Apr 2020 12:43:12 GMT
Connection: keep-alive
ETag: "5e9eea60-264"
Accept-Ranges: bytes
[root@node-122-84 ~]# curl  -I http://172.7.84.2
HTTP/1.1 200 OK
Server: nginx/1.18.0
Date: Mon, 07 Dec 2020 09:31:38 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 21 Apr 2020 12:43:12 GMT
Connection: keep-alive
ETag: "5e9eea60-264"
Accept-Ranges: bytes
```

```
[root@node-122-85 ~]# curl  -I http://172.7.84.2
^C
[root@node-122-84 ~]# curl  -I http://172.7.85.2
^C
```

flannel

运算节点做如下操作

```
# mkdir /opt/flannel
# tar -xf /tmp/flannel-v0.11.0-linux-amd64.tar.gz -C /opt/
```

分发证书

```
certs]# ansible k8s_node -m copy -a "src=./ca.pem dest=/opt/flannel/certs/"
certs]# ansible k8s_node -m copy -a "src=./client.pem dest=/opt/flannel/certs/"
certs]# ansible k8s_node -m copy -a "src=./client-key.pem dest=/opt/flannel/certs/"
```

```
[root@ctrl-122-81 ~]# /opt/etcd/etcdctl set /coreos.com/network/config '{"Network": "172.7.0.0/16", "Backend": {"Type": "host-gw"}}'
{"Network": "172.7.0.0/16", "Backend": {"Type": "host-gw"}}
[root@ctrl-122-81 ~]# /opt/etcd/etcdctl get /coreos.com/network/config
{"Network": "172.7.0.0/16", "Backend": {"Type": "host-gw"}}
```

```
# vim /opt/flannel/flanneld.sh
#!/bin/sh
./flanneld \
    --public-ip=192.168.122.84 \
    --etcd-endpoints=https://192.168.122.81:2379,https://192.168.122.82:2379,https://192.168.122.83:2379 \
    --etcd-keyfile=./certs/client-key.pem \
    --etcd-certfile=./certs/client.pem \
    --etcd-cafile=./certs/ca.pem \
    --iface=eth0 \
    --subnet-file=./subnet.env \
    --healthz-port=2401
# chmod +x /opt/flannel/flanneld.sh 
```

```
# vim /etc/supervisord.d/flanneld.ini
[program:flanneld-122-84]
command=/opt/flannel/flanneld.sh                             ; the program (relative uses PATH, can take args)
numprocs=1                                                   ; number of processes copies to start (def 1)
directory=/opt/flannel                                       ; directory to cwd to before exec (def no cwd)
autostart=true                                               ; start at supervisord start (default: true)
autorestart=true                                             ; retstart at unexpected quit (default: true)
startsecs=30                                                 ; number of secs prog must stay running (def. 1)
startretries=3                                               ; max # of serial start failures (default 3)
exitcodes=0,2                                                ; 'expected' exit codes for process (default 0,2)
stopsignal=QUIT                                              ; signal used to kill process (default TERM)
stopwaitsecs=10                                              ; max num secs to wait b4 SIGKILL (default 10)
user=root                                                    ; setuid to this UNIX account to run the program
redirect_stderr=true                                         ; redirect proc stderr to stdout (default false)
stdout_logfile=/data/logs/flanneld/flanneld.stdout.log       ; stderr log path, NONE for none; default AUTO
stdout_logfile_maxbytes=64MB                                 ; max # logfile bytes b4 rotation (default 50MB)
stdout_logfile_backups=5                                     ; # of stdout logfile backups (default 10)
stdout_capture_maxbytes=1MB                                  ; number of bytes in 'capturemode' (default 0)
stdout_events_enabled=false                                  ; emit events on stdout writes (default false)
# mkdir /data/logs/flanneld/
# supervisorctl update
# supervisorctl status
flanneld-122-84                  RUNNING   pid 6435, uptime 20:41:38
kube-kubelet-122-84              RUNNING   pid 13916, uptime 6 days, 21:13:13
kube-proxy-122-84                RUNNING   pid 14566, uptime 3 days, 21:04:41
```

运算节点iptables优化，容器之间访问使用容器IP，不使用运算节点IP：

最初容器日志如下

```
# kubectl logs -f  nginx-ds-kmrnx
...
192.168.122.84 - - [11/Dec/2020:07:16:50 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.64.0" "-"
# kubectl logs -f nginx-ds-tlhqr  
...
192.168.122.85 - - [11/Dec/2020:07:23:46 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.64.0" "-"
```

修改iptables规则

```
# yum install -y iptables-services
# iptables-save > /etc/sysconfig/iptables  
# vim +53 /etc/sysconfig/iptables
-A POSTROUTING -s 172.7.84.0/24 ! -d 172.7.0.0/16 ! -o docker0 -j MASQUERADE
# systemctl restart iptables
# systemctl enable iptables
```

修改后日志如下

```
root@nginx-ds-kmrnx:/# curl 172.7.84.2
root@nginx-ds-tlhqr:/# curl 172.7.85.2
[root@ctrl-122-81 ~]# kubectl get pods -o wide
NAME             READY   STATUS    RESTARTS   AGE     IP           NODE                   NOMINATED NODE   READINESS GATES
nginx-ds-kmrnx   1/1     Running   0          3d21h   172.7.85.2   node-122-85.host.com   <none>           <none>
nginx-ds-tlhqr   1/1     Running   0          3d21h   172.7.84.2   node-122-84.host.com   <none>           <none>
# kubectl logs -f nginx-ds-kmrnx
172.7.84.2 - - [11/Dec/2020:07:27:12 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.64.0" "-"
# kubectl logs -f nginx-ds-tlhqr
172.7.85.2 - - [11/Dec/2020:07:27:06 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.64.0" "-"
```

```
# Generated by iptables-save v1.4.21 on Fri Dec 11 15:21:13 2020
*filter
:INPUT ACCEPT [9:2628]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [7:630]
:DOCKER - [0:0]
:DOCKER-ISOLATION-STAGE-1 - [0:0]
:DOCKER-ISOLATION-STAGE-2 - [0:0]
:DOCKER-USER - [0:0]
:KUBE-FIREWALL - [0:0]
:KUBE-FORWARD - [0:0]
-A INPUT -j KUBE-FIREWALL
-A FORWARD -m comment --comment "kubernetes forwarding rules" -j KUBE-FORWARD
-A FORWARD -j DOCKER-USER
-A FORWARD -j DOCKER-ISOLATION-STAGE-1
-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -o docker0 -j DOCKER
-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT
-A FORWARD -s 172.7.0.0/16 -j ACCEPT
-A FORWARD -d 172.7.0.0/16 -j ACCEPT
-A OUTPUT -j KUBE-FIREWALL
-A DOCKER-ISOLATION-STAGE-1 -i docker0 ! -o docker0 -j DOCKER-ISOLATION-STAGE-2
-A DOCKER-ISOLATION-STAGE-1 -j RETURN
-A DOCKER-ISOLATION-STAGE-2 -o docker0 -j DROP
-A DOCKER-ISOLATION-STAGE-2 -j RETURN
-A DOCKER-USER -j RETURN
-A KUBE-FIREWALL -m comment --comment "kubernetes firewall for dropping marked packets" -m mark --mark 0x8000/0x8000 -j DROP
-A KUBE-FORWARD -m comment --comment "kubernetes forwarding rules" -m mark --mark 0x4000/0x4000 -j ACCEPT
-A KUBE-FORWARD -s 172.7.0.0/16 -m comment --comment "kubernetes forwarding conntrack pod source rule" -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A KUBE-FORWARD -d 172.7.0.0/16 -m comment --comment "kubernetes forwarding conntrack pod destination rule" -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
COMMIT
# Completed on Fri Dec 11 15:21:13 2020
# Generated by iptables-save v1.4.21 on Fri Dec 11 15:21:13 2020
*nat
:PREROUTING ACCEPT [3:120]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:DOCKER - [0:0]
:KUBE-FIREWALL - [0:0]
:KUBE-LOAD-BALANCER - [0:0]
:KUBE-MARK-DROP - [0:0]
:KUBE-MARK-MASQ - [0:0]
:KUBE-NODE-PORT - [0:0]
:KUBE-POSTROUTING - [0:0]
:KUBE-SERVICES - [0:0]
-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
-A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
-A POSTROUTING -m comment --comment "kubernetes postrouting rules" -j KUBE-POSTROUTING
-A POSTROUTING -s 172.7.85.0/24 ! -d 172.7.0.0/16 ! -o docker0 -j MASQUERADE
-A DOCKER -i docker0 -j RETURN
-A KUBE-FIREWALL -j KUBE-MARK-DROP
-A KUBE-LOAD-BALANCER -j KUBE-MARK-MASQ
-A KUBE-MARK-DROP -j MARK --set-xmark 0x8000/0x8000
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
-A KUBE-POSTROUTING -m comment --comment "Kubernetes endpoints dst ip:port, source ip for solving hairpin purpose" -m set --match-set KUBE-LOOP-BACK dst,dst,src -j MASQUERADE
-A KUBE-SERVICES ! -s 172.7.0.0/16 -m comment --comment "Kubernetes service cluster ip + port for masquerade purpose" -m set --match-set KUBE-CLUSTER-IP dst,dst -j KUBE-MARK-MASQ
-A KUBE-SERVICES -m addrtype --dst-type LOCAL -j KUBE-NODE-PORT
-A KUBE-SERVICES -m set --match-set KUBE-CLUSTER-IP dst,dst -j ACCEPT
COMMIT
# Completed on Fri Dec 11 15:21:13 2020
```

如果有拒绝 icmp-host-prohibited 方面的配置，需要删除。

CoreDNS



```
# kubectl apply -f http://yaml.k8s.com/coredns/rbac.yaml
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
# kubectl apply -f http://yaml.k8s.com/coredns/configmap.yaml
configmap/coredns created
# kubectl apply -f http://yaml.k8s.com/coredns/deployment.yaml
deployment.apps/coredns created
# kubectl apply -f http://yaml.k8s.com/coredns/service.yaml
service/coredns created
```

```
# cat rbac.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
  labels:
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: Reconcile
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: EnsureExists
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
```

```
# cat configmap.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        log
        health
        ready
        kubernetes cluster.local 192.168.0.0/16
        forward . 192.168.23.192
        cache 30
        loop
        reload
        loadbalance
    }
```

```
# cat deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/name: "CoreDNS"
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: coredns
  template:
    metadata:
      labels:
        k8s-app: coredns
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: coredns
      containers:
      - name: coredns
        image: harbor.k8s.com:8080/public/coredns:v1.6.1
        args:
        - -conf
        - /etc/coredns/Corefile
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
```

```
# cat service.yaml 
apiVersion: v1
kind: Service
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: coredns
  clusterIP: 192.168.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
  - name: metrics
    port: 9153
    protocol: TCP
```



```
# kubectl get all -n kube-system
NAME                           READY   STATUS    RESTARTS   AGE
pod/coredns-64df75d6f5-b4kzb   1/1     Running   0          38s


NAME              TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
service/coredns   ClusterIP   192.168.0.2   <none>        53/UDP,53/TCP,9153/TCP   21s


NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns   1/1     1            1           38s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/coredns-64df75d6f5   1         1         1       38s
```

```
[root@node-122-84 ~]# dig -t A www.baidu.com @192.168.0.2 +short
www.a.shifen.com.
180.101.49.12
180.101.49.11
[root@node-122-84 ~]# dig -t A ctrl-122-81.host.com @192.168.0.2 +short 
192.168.122.81
```

```
[root@ctrl-122-81 ~]# kubectl get svc -n kube-public
NAME       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
nginx-dp   ClusterIP   192.168.10.246   <none>        80/TCP    3d20h
[root@ctrl-122-81 ~]# kubectl get deploy -n kube-public
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
nginx-dp   2/2     2            2           3d20h
[root@ctrl-122-81 ~]# kubectl get pod -n kube-public
NAME                        READY   STATUS    RESTARTS   AGE
nginx-dp-6fdfdfc5bb-9xj48   1/1     Running   0          6d20h
nginx-dp-6fdfdfc5bb-hzwbw   1/1     Running   0          6d20h
[root@ctrl-122-81 ~]# kubectl -n kube-public exec -it nginx-dp-6fdfdfc5bb-9xj48 /bin/bash
root@nginx-dp-6fdfdfc5bb-9xj48:/# curl -I nginx-dp
HTTP/1.1 200 OK
Server: nginx/1.18.0
Date: Thu, 17 Dec 2020 03:24:10 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 21 Apr 2020 12:43:12 GMT
Connection: keep-alive
ETag: "5e9eea60-264"
Accept-Ranges: bytes
root@nginx-dp-6fdfdfc5bb-9xj48:/# cat /etc/resolv.conf 
nameserver 192.168.0.2
search kube-public.svc.cluster.local svc.cluster.local cluster.local host.com
options ndots:5
[root@node-122-84 ~]# dig -t A nginx-dp.kube-public.svc.cluster.local @192.168.0.2 +short       
192.168.10.246
[root@node-122-84 ~]# cat /etc/resolv.conf
search host.com
nameserver 192.168.23.192
```



````
# docker pull traefik:v1.7.2-alpine
# docker tag traefik:v1.7.2-alpine harbor.k8s.com:8080/public/traefik:v1.7.2
# docker push harbor.k8s.com:8080/public/traefik:v1.7.2
````



```
[root@ctrl-122-81 ~]# kubectl apply -f http://yaml.k8s.com/traefik/rbac.yaml
serviceaccount/traefik-ingress-controller created
clusterrole.rbac.authorization.k8s.io/traefik-ingress-controller created
clusterrolebinding.rbac.authorization.k8s.io/traefik-ingress-controller created
[root@ctrl-122-81 ~]# kubectl apply -f http://yaml.k8s.com/traefik/daemonset.yaml
daemonset.extensions/traefik-ingress created
[root@ctrl-122-81 ~]# kubectl apply -f http://yaml.k8s.com/traefik/service.yaml
service/traefik-ingress-service created
[root@ctrl-122-81 ~]# kubectl apply -f http://yaml.k8s.com/traefik/ingress.yaml
ingress.extensions/traefik-web-ui created
```

```
# cat rbac.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
```

```
# cat daemonset.yaml 
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: traefik-ingress
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress
        name: traefik-ingress
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: harbor.k8s.com:8080/public/traefik:v1.7.2
        name: traefik-ingress
        ports:
        - name: controller
          containerPort: 80
          hostPort: 81
        - name: admin-web
          containerPort: 8080
        securityContext:
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        args:
        - --api
        - --kubernetes
        - --logLevel=INFO
        - --insecureskipverify=true
        - --kubernetes.endpoint=https://192.168.122.10:7443
        - --accesslog
        - --accesslog.filepath=/var/log/traefik_access.log
        - --traefiklog
        - --traefiklog.filepath=/var/log/traefik.log
        - --metrics.prometheus
```

```
# cat service.yaml 
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress
  ports:
    - protocol: TCP
      port: 80
      name: controller
    - protocol: TCP
      port: 8080
      name: admin-web
```

```
# cat ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: traefik.k8s.com
    http:
      paths:
      - path: /
        backend:
          serviceName: traefik-ingress-service
          servicePort: 8080
```



```
[root@ctrl-122-81 ~]# kubectl get pod -n kube-system -o wide
NAME                       READY   STATUS    RESTARTS   AGE     IP           NODE                   NOMINATED NODE   READINESS GATES
coredns-64df75d6f5-b4kzb   1/1     Running   0          5d22h   172.7.84.4   node-122-84.host.com   <none>           <none>
traefik-ingress-8cdwm      1/1     Running   0          57s     172.7.85.4   node-122-85.host.com   <none>           <none>
traefik-ingress-hqnh8      1/1     Running   0          57s     172.7.84.5   node-122-84.host.com   <none>           <none>
[root@ctrl-122-81 ~]# kubectl get ds -n kube-system
NAME              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
traefik-ingress   2         2         2       2            2           <none>          88s
```



dashboard

```
[root@ctrl-122-81 ~]# docker pull k8scn/kubernetes-dashboard-amd64:v1.8.3
[root@ctrl-122-81 ~]# docker tag k8scn/kubernetes-dashboard-amd64:v1.8.3 harbor.k8s.com:8080/public/kubernetes-dashboard-amd64:v1.8.3
[root@ctrl-122-81 ~]# docker push harbor.k8s.com:8080/public/kubernetes-dashboard-amd64:v1.8.3
```

```
# kubectl apply -f http://yaml.k8s.com/dashboard/rbac.yaml
# kubectl apply -f http://yaml.k8s.com/dashboard/deployment.yaml
# kubectl apply -f http://yaml.k8s.com/dashboard/service.yaml
# kubectl apply -f http://yaml.k8s.com/dashboard/ingress.yaml
```

```
# cat rbac.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
    addonmanager.kubernetes.io/mode: Reconcile
  name: kubernetes-dashboard-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard-admin
  namespace: kube-system
```

```
# cat deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      priorityClassName: system-cluster-critical
      containers:
      - name: kubernetes-dashboard
        image: harbor.k8s.com:8080/public/kubernetes-dashboard-amd64:v1.10.1
        resources:
          limits:
            cpu: 100m
            memory: 300Mi
          requests:
            cpu: 50m
            memory: 100Mi
        ports:
        - containerPort: 8443
          protocol: TCP
        args:
          # PLATFORM-SPECIFIC ARGS HERE
          - --auto-generate-certificates
        volumeMounts:
        - name: tmp-volume
          mountPath: /tmp
        livenessProbe:
          httpGet:
            scheme: HTTPS
            path: /
            port: 8443
          initialDelaySeconds: 30
          timeoutSeconds: 30
      volumes:
      - name: tmp-volume
        emptyDir: {}
      serviceAccountName: kubernetes-dashboard-admin
      tolerations:
      - key: "CriticalAddonsOnly"
        operator: "Exists"
```

```
# cat service.yaml 
apiVersion: v1
kind: Service
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    k8s-app: kubernetes-dashboard
  ports:
  - port: 443
    targetPort: 8443
```

```
# cat ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: dashboard.k8s.com
    http:
      paths:
      - backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
```



```
# kubectl get pods -n kube-system
NAME                                    READY   STATUS    RESTARTS   AGE
coredns-64df75d6f5-b4kzb                1/1     Running   0          6d23h
kubernetes-dashboard-674c458676-ljmzq   1/1     Running   0          57s
traefik-ingress-8cdwm                   1/1     Running   0          25h
traefik-ingress-hqnh8                   1/1     Running   0          25h
# kubectl get svc -n kube-system
NAME                      TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)                  AGE
coredns                   ClusterIP   192.168.0.2       <none>        53/UDP,53/TCP,9153/TCP   6d23h
kubernetes-dashboard      ClusterIP   192.168.230.152   <none>        443/TCP                  72s
traefik-ingress-service   ClusterIP   192.168.53.58     <none>        80/TCP,8080/TCP          25h
# kubectl get ingress -n kube-system
NAME                   HOSTS               ADDRESS   PORTS   AGE
kubernetes-dashboard   dashboard.k8s.com             80      87s
traefik-web-ui         traefik.k8s.com               80      25h
```

```
[root@node-122-84 ~]# dig -t A dashboard.k8s.com @192.168.0.2 +short
192.168.122.10
```

升级

```
docker image pull registry.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.1
docker image tag f9aed6605b81 harbor.k8s.com:8080/public/kubernetes-dashboard-amd64:v1.10.1
docker image push harbor.k8s.com:8080/public/kubernetes-dashboard-amd64:v1.10.1
```



```
[root@ctrl-122-81 ~]# kubectl apply -f http://yaml.k8s.com/dashboard/deployment.yaml
deployment.apps/kubernetes-dashboard configured
```



```
certs]# (umask 077;openssl genrsa -out dashboard.k8s.com.key 2048)
certs]# openssl req -new -key dashboard.k8s.com.key -out dashboard.k8s.com.csr -subj "/CN=dashboard.k8s.com/C=CN/ST=SH/L=Shanghai/O=k8s/OU=ops"
certs]# openssl x509 -req -in dashboard.k8s.com.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out dashboard.k8s.com.crt -days 3650 
Signature ok
subject=/CN=dashboard.k8s.com/C=CN/ST=SH/L=Shanghai/O=k8s/OU=ops
Getting CA Private Key
certs]# ll dashboard.k8s.com.*
-rw-r--r-- 1 root root 1196 Dec 24 15:05 dashboard.k8s.com.crt
-rw-r--r-- 1 root root 1001 Dec 24 15:02 dashboard.k8s.com.csr
-rw------- 1 root root 1679 Dec 24 14:59 dashboard.k8s.com.key
certs]# scp dashboard.k8s.com.crt dashboard.k8s.com.key ingress-122-71:/etc/nginx/certs/
certs]# scp dashboard.k8s.com.crt dashboard.k8s.com.key ingress-122-72:/etc/nginx/certs/
```



```
[root@ingress-122-71 certs]# vim /etc/nginx/conf.d/dashboard.conf 
server {
    listen       80;
    server_name  dashboard.od.com;
    rewrite ^(.*)$ https://${server_name}$1 permanent;
}

server {
    listen       443 ssl;
    server_name  dashboard.k8s.com;

    ssl_certificate "certs/dashboard.k8s.com.crt";
    ssl_certificate_key "certs/dashboard.k8s.com.key";
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout  10m;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://default_backend_traefik;
        proxy_set_header Host       $http_host;
        proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
    }
}
# ngint -t 
# systemctl nginx reload
```

```
[root@ctrl-122-81 ~]# kubectl get secret -n kube-system |grep kubernetes-dashboard-admin
kubernetes-dashboard-admin-token-b2hvs   kubernetes.io/service-account-token   3      5d21h
# kubectl describe secret kubernetes-dashboard-admin-token-b2hvs -n kube-system
...
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi10b2tlbi1iMmh2cyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjJkOWE1MjYxLTIxMDQtNGQ2OS1hZDc1LTE3NGRjNzQ2ZjY2YiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTprdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiJ9.Rmg6UJdjakYdx3fcIMR-jcgDP7YWHQxRWMpxwr4-bwJs87KUuAkP2x4Q3aAVxsRtgz-lXNg3pVefO6nBXJNtzOzzEF6fwS4WtRUU0SiPilhMM85NVLZ2-0RmsVPttph8N_4HesnrIMnYG9r2LasUtcb8-OmUUi0pFkJ2PdDxXqF4rtT0nSnd0SfA1GhcoqHLjliJG6xPBPU3XW8pvjr959CX_BFRR97tF5m1ZyblGqGBjT5EtaXrGuvw8qxV5o-G_ws0HVueqHwwvjA0CdUYefuGTE5y8gyc6oVhzXjIXSy_4zPvSGkTea8pHiG11OxY5mZJN9-2YKgE1iOxogcjCw
```

