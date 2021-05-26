## `kubectl`
### `get`
获取 `node` 节点信息
```shell
# kubectl get  nodes
```
获取 `pod` 的信息
```shell
# kubectl get pod --all-namespaces
```
获取 `pod` 指定 `namespace` 信息
```shell
# kubectl get pod -n kube-system
```
查看 `RC` 和 `service` 列表
```shell
# kubectl get rc,svc
```
### `describe`
显示 `Node` 的详细信息
```shell
# kubectl describe node node1
```
显示 `Pod` 的详细信息
```shell
# kubectl describe pod coredns-fb8b8dccf-pf5lm -n kube-system
# kubectl describe pod nginx-65f88748fd-7f79p
```
### `delete `
基于 `pod.yaml` 定义的名称删除 `pod`
```shell
# kubectl delete -f pod.yaml
```
删除所有包含某个 `label` 的 `pod` 和 `service`
```shell
# kubectl delete pod,svc -l name=<label-name>
```
删除所有 `Pod`
```shell
#  kubectl delete pod --all
```
### `exec`
执行 `pod` 的 `date` 命令
```shell
# kubectl exec <pod-name> -- date
```
通过bash 获得 `pod` 中某个容器的 `TTY`，相当于登陆容器
```shell
# kubectl exec -it <pod-name> -c <container-name> -- bash
```
### `log`
查看容器的日志
```shell
# kubectl logs <pod-name> 
```



技巧

查看证书过期时间

```
# cfssl-certinfo -cert apiserver.pem |grep not_after
```

查看公网证书信息

```
# cfssl-certinfo -domain www.baidu.com
```

查询名称空间

```
# kubectl get ns
# kubectl get namespace
```

查询指定名称空间所有资源

```
# kubectl get all -n default
```

创建名称空间

```
# kubectl create namespace app
```

删除名称空间

```
# kubectl delete namespace app
```

创建deployment

```
# kubectl create deployment nginx-dp --image=harbor.k8s.com:8080/public/nginx:v1.18.0 -n kube-public
# kubectl get deploy -n kube-public
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
nginx-dp   1/1     1            1           15s
# kubectl get pod -n kube-public
NAME                        READY   STATUS    RESTARTS   AGE
nginx-dp-6fdfdfc5bb-ffphw   1/1     Running   0          25s
 kubectl get pods -n kube-public -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP           NODE                   NOMINATED NODE   READINESS GATES
nginx-dp-6fdfdfc5bb-ffphw   1/1     Running   0          9m51s   172.7.84.3   node-122-84.host.com   <none>           <none>
```

查看详细信息

```
# kubectl describe deployment nginx-dp -n kube-public
```

删除pod，删除以后还会拉取一个新pod。

```
# kubectl delete pod nginx-dp-6fdfdfc5bb-ffphw -n kube-public
# kubectl get pod -n kube-public -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP           NODE                   NOMINATED NODE   READINESS GATES
nginx-dp-6fdfdfc5bb-dc2tc   1/1     Running   0          30s   172.7.85.3   node-122-85.host.com   <none>           <none>
```

删除deployment

```
# kubectl delete deployment nginx-dp -n kube-public
```

service资源

```
# kubectl create deployment nginx-dp --image=harbor.k8s.com:8080/public/nginx:v1.18.0 -n kube-public
# kubectl expose deployment nginx-dp --port=80 -n kube-public 
# kubectl get service -o wide -n kube-public
NAME       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE   SELECTOR
nginx-dp   ClusterIP   192.168.10.246   <none>        80/TCP    32s   app=nginx-dp
# kubectl get pod -o wide -n kube-public
NAME                        READY   STATUS    RESTARTS   AGE   IP           NODE                   NOMINATED NODE   READINESS GATES
nginx-dp-6fdfdfc5bb-9xj48   1/1     Running   0          42s   172.7.84.3   node-122-84.host.com   <none>           <none>
# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.0.1:443 nq
  -> 192.168.122.81:6443          Masq    1      0          0         
  -> 192.168.122.82:6443          Masq    1      0          0         
  -> 192.168.122.83:6443          Masq    1      0          0         
TCP  192.168.10.246:80 nq
  -> 172.7.84.3:80                Masq    1      0          1  
# kubectl scale deployment nginx-dp --replicas=2 -n kube-public
# kubectl get pod -o wide -n kube-public                                                
NAME                        READY   STATUS    RESTARTS   AGE     IP           NODE                   NOMINATED NODE   READINESS GATES
nginx-dp-6fdfdfc5bb-9xj48   1/1     Running   0          5m30s   172.7.84.3   node-122-84.host.com   <none>           <none>
nginx-dp-6fdfdfc5bb-hzwbw   1/1     Running   0          17s     172.7.85.3   node-122-85.host.com   <none>           <none>
[root@node-122-84 ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.0.1:443 nq
  -> 192.168.122.81:6443          Masq    1      0          0         
  -> 192.168.122.82:6443          Masq    1      0          0         
  -> 192.168.122.83:6443          Masq    1      0          0         
TCP  192.168.10.246:80 nq
  -> 172.7.84.3:80                Masq    1      0          0         
  -> 172.7.85.3:80                Masq    1      0          0
# kubectl describe service nginx-dp -n kube-public
Name:              nginx-dp
Namespace:         kube-public
Labels:            app=nginx-dp
Annotations:       <none>
Selector:          app=nginx-dp
Type:              ClusterIP
IP:                192.168.10.246
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         172.7.84.3:80,172.7.85.3:80
Session Affinity:  None
Events:            <none>
```

查看资源的yaml，主要四大段，apiVersion、kind、metadate、 spec

````
# kubectl get svc -n kube-public -o yaml
````

查看资源的参数：

```
# kubectl explain svc.metadata
```

删除label：

```
]# kubectl get node --show-labels
NAME                   STATUS   ROLES   AGE     VERSION    LABELS
node-122-84.host.com   Ready    node    48d     v1.15.12   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node-122-84.host.com,kubernetes.io/os=linux,node-role.kubernetes.io/node=
node-122-85.host.com   Ready    node    48d     v1.15.12   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node-122-85.host.com,kubernetes.io/os=linux,node-role.kubernetes.io/node=,ssd=true
node-122-86.host.com   Ready    node    3h41m   v1.15.12   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node-122-86.host.com,kubernetes.io/os=linux,node-role.kubernetes.io/node=
]# kubectl label node node-122-85.host.com ssd-
```

新增label值：

```
]# kubectl label node node-122-86.host.com disk=ssd
```

修改label值：

```
]# kubectl label node node-122-86.host.com disk=hdd --overwrite
```





