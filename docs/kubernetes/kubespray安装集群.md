## kubespray

### 安装环境介绍

| 主机名 | IP             | 系统            | 作用                  |
| ------ | -------------- | --------------- | --------------------- |
| node1  | 192.168.120.71 | CentOS 7.4.1708 | master、node、ansible |
| node2  | 192.168.120.72 | CentOS 7.4.1708 | master、node          |
| node3  | 192.168.120.73 | CentOS 7.4.1708 | master、node          |

### 安装环境准备

#### 关闭 `selinux`

```shell
# setenforce 0
# sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux
```

#### 时间同步

```shell
# /usr/sbin/ntpdate 1.centos.pool.ntp.org &>/dev/null
```

#### 关闭防火墙

```shell
# systemctl stop firewalld
# systemctl disable firewalld
```

#### 关闭 `swap`

```shell
swapoff -a
sed -i 's/.*swap.*/#&/' /etc/fstab
```

#### ssh免密登录

这是在 192.168.120.71 主机上生成公私钥，然后分发到集群中其他主机中去。

```shell
# ssh-keygen
# ssh-copy-id 192.168.120.71
# ssh-copy-id 192.168.120.72
# ssh-copy-id 192.168.120.73
```

#### 配置转发参数

```shell
cat  >/etc/sysctl.d/k8s.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl -p
```

### 安装配置

#### ansible 安装

```shell
# yum install -y python36 python36-devel python36-pip ansible gcc openssl-devel git
```

#### pip 安装源修改

```shell
mkdir /root/.pip
cat > /root/.pip/pip.conf < EOF
[global]
trusted-host =  mirrors.aliyun.com
index-url = https://mirrors.aliyun.com/pypi/simple
EOF
```

#### kubespray 下载并修改

```shell
# wget https://github.com/kubernetes-sigs/kubespray/archive/v2.10.4.tar.gz
# tar -xf v2.10.4.tar.gz -C /opt/
# cd /opt/kubespray-2.10.4
# pip3 install -r requirements.txt
# cp -rfp inventory/sample inventory/mycluster
# declare -a IPS=(192.168.120.71 192.168.120.72 192.168.120.73)
# CONFIG_FILE=inventory/mycluster/hosts.yml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
# sed -i  s/k8s.gcr.io/gcr.azk8s.cn/g  roles/download/defaults/main.yml
# sed -i  s/gcr.io/gcr.azk8s.cn/g  roles/download/defaults/main.yml
# sed -i  s/gcr.io/gcr.azk8s.cn/g  inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml
# vim inventory/mycluster/group_vars/all/docker.yml
......
docker_registry_mirrors:
  - https://registry.docker-cn.com
  - https://mirror.aliyuncs.com
......
# vim roles/download/defaults/main.yml
......
nodelocaldns_image_repo: "gcr.azk8s.cn/google-containers/k8s-dns-node-cache"
......
dnsautoscaler_image_repo: "gcr.azk8s.cn/google-containers/cluster-proportional-autoscaler-{{ image_arch }}"
......
# ansible-playbook -i inventory/mycluster/hosts.yml --become --become-user=root cluster.yml
```

pip 安装失败，多尝试几次，坑爹的GFW

#### 验证集群

```shell
查看集群状态
# kubectl get nodes -o wide
NAME    STATUS   ROLES    AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION          CONTAINER-RUNTIME
node1   Ready    master   39m   v1.14.3   192.168.120.71   <none>        CentOS Linux 7 (Core)   3.10.0-693.el7.x86_64   docker://18.9.5
node2   Ready    master   38m   v1.14.3   192.168.120.72   <none>        CentOS Linux 7 (Core)   3.10.0-693.el7.x86_64   docker://18.9.5
node3   Ready    <none>   37m   v1.14.3   192.168.120.73   <none>        CentOS Linux 7 (Core)   3.10.0-693.el7.x86_64   docker://18.9.5
查看机器组件
# kubectl get all --all-namespaces
NAMESPACE     NAME                                          READY   STATUS    RESTARTS   AGE
default       pod/nginx-app-7bdbd99ff-l9pdj                 1/1     Running   0          17m
kube-system   pod/calico-kube-controllers-fcb874cb5-whfpr   1/1     Running   0          36m
kube-system   pod/calico-node-lffcr                         1/1     Running   0          36m
kube-system   pod/calico-node-mn8gx                         1/1     Running   0          36m
......
创建一个deployment
vim /root/nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment 
metadata: 
  name: nginx-dm
spec: 
  replicas: 3
  selector:
    matchLabels:
      name: nginx
  template: 
    metadata: 
      labels: 
        name: nginx 
    spec: 
      containers: 
        - name: nginx 
          image: nginx:alpine 
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
              name: http
---

apiVersion: v1 
kind: Service
metadata: 
  name: nginx-svc 
spec: 
  ports: 
    - port: 80
      name: http
      targetPort: 80
      protocol: TCP 
  selector: 
    name: nginx
# kubectl apply -f nginx-deployment.yaml 
# kubectl get pods -o wide
NAME                        READY   STATUS    RESTARTS   AGE    IP            NODE    NOMINATED NODE   READINESS GATES
nginx-app-7bdbd99ff-l9pdj   1/1     Running   0          21m    10.233.92.2   node3   <none>           <none>
nginx-dm-76cf455886-clsmt   1/1     Running   0          100s   10.233.90.2   node1   <none>           <none>
nginx-dm-76cf455886-gt75w   1/1     Running   0          100s   10.233.92.3   node3   <none>           <none>
nginx-dm-76cf455886-svv8k   1/1     Running   0          100s   10.233.96.3   node2   <none>           <none>
# kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE    SELECTOR
kubernetes   ClusterIP   10.233.0.1      <none>        443/TCP   44m    <none>
nginx-svc    ClusterIP   10.233.17.159   <none>        80/TCP    2m6s   name=nginx
# ipvsadm -Ln  |grep -A 3 '10.233.17.159'
TCP  10.233.17.159:80 rr
  -> 10.233.90.2:80               Masq    1      0          0         
  -> 10.233.92.3:80               Masq    1      0          0         
  -> 10.233.96.3:80               Masq    1      0          0  
# curl -I  http://10.233.17.159
HTTP/1.1 200 OK
Server: nginx/1.17.2
Date: Thu, 01 Aug 2019 09:26:01 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 23 Jul 2019 13:01:30 GMT
Connection: keep-alive
ETag: "5d37052a-264"
Accept-Ranges: bytes
```

#### ansible 操作

删除节点

```shell
ansible-playbook -i inventory/mycluster/hosts.yml --become --become-user=root remove-node.yml node3
```

添加节点

```shell
ansible-playbook -i inventory/mycluster/hosts.yml --become --become-user=root scale.yml
```

升级节点

```shell
ansible-playbook -i inventory/mycluster/hosts.yml --become --become-user=root upgrade-cluster.yml
```

卸载

```shell
ansible-playbook -i inventory/mycluster/hosts.ini reset.yml
```

#### 说明

* 主机角色的自定义可以在 `inventory/mycluster/inventory.ini` 文件中进行编辑。
* ansible 运行中如果非root，切换sudo用户: -u username -b

