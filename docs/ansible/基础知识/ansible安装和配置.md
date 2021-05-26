## ansible 初步使用

### 安装

后面学习使用 ansible 是基于 CentOS 6 系统，如果通过 yum 安装 ansible 需要首先配置 epel 源。可以根据自己的系统版本进行如下操作配置好 epel 源。

epel(RHEL 6)

```shell
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-6.repo
```

epel(RHEL 7)

```shell
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
```

配置好 epel 源之后直接通过 yum 命令进行安装：

```shell
# yum install -y ansible
```

### 配置参数

> inventory      = /etc/ansible/hosts

该参数表示资源清单 inventory  文件的位置，其指定的文件或目录就是 ansible 需要连接管理的主机列表。

> library        = /usr/share/my_modules/

该参数是指定第三方编写的 ansible 模块存放目录。

> forks          = 5

设置默认情况下 ansible 最多能有多少个进程同时工作，默认设置为 5 个进程并行处理。

> sudo_user      = root

设置默认执行命令的用户，也可以在 playbook 中重新设置该参数。

> remote_port    = 22

指定连接被管理节点的默认 ssh 端口，可以根据实际情况进行设置。

> host_key_checking = False

设置是否检查 ssh 主机的密钥。如果在 ansible 中有一台已经认证过的机器重装系统，就会导致ansible 管理该机器提示一个密钥不匹配的错误，为避免出现这种情况，可以关闭该功能。同时在新增被管理机器时默认是会提示对key信息的确认，为避免确认导致增加的手动操作，可以关闭该参数。

> log_path = /var/log/ansible.log

默认 ansible 是不会记录日志的，如果需要把 ansible 系统输出记录到日志中，可以开启该参数。

### ssh 免密登录配置

在确认该系统未有密钥的情况下生成密钥对：

```shell
# ssh-keygen 
```

执行命令之后就是全部回车确认并生成公私钥。下面通过 `ssh-copy-id` 命令将密钥拷贝到被管理的主机上：

```shell
# ssh-copy-id 192.168.120.62
# ssh-copy-id 192.168.120.63
```

### 主机连通性测试

通过 yum 安装的 ansible 默认会将主机配置文件放在 `/etc/ansible/hosts` 中，修改改文件如下：

```shell 
# vim /etc/ansible/hosts
[webservers]
192.168.120.62
192.168.120.63
```

通过如下操作就能验证主机的互通：

```shell
# ansible webservers -m ping -o
192.168.120.62 | SUCCESS => {"changed": false, "ping": "pong"}
192.168.120.63 | SUCCESS => {"changed": false, "ping": "pong"}
```

### 指定配置文件进行执行命令

前面是使用默认主机配置文件，下面也可以自己指定配置文件：

```shell
# cat test.cfg 
[test]
192.168.120.62
192.168.120.63
```

通过上面指定主机文件来执行相关命令：

```shell
# ansible test -m shell -a '/bin/echo hello ansible!' -i test.cfg 
192.168.120.63 | SUCCESS | rc=0 >>
hello ansible!

192.168.120.62 | SUCCESS | rc=0 >>
hello ansible!
```

## ansible 组件介绍

### Ansible Inventory

在管理大型集群时我们需要管理不同业务的不同机器，这些机器信息都存放在 ansible 的 inventory 组件中。日常使用中配置部署的主机清单都是存放在 inventory 中，这样才很方便的管理集群。其中默认 inventory 是一个静态的 INI 格式的文件 `/etc/ansible/hosts`，在实际使用中也可以通过 -i 参数进行临时配置。

#### 主机和主机组定义

通过下面一个配置文件来进行相关配置讲解：

```shell
192.168.120.62 ansible_ssh_pass='redhat'
192.168.120.63 ansible_ssh_pass='redhat'
[webservers]
192.168.120.6[1:3]
[webservers:vars]
ansible_ssh_pass='redhat'
[ansible:children]
webservers
```

第一、二行定义了两个主机，并且定义了 ssh 登录密码。

第三、四行定义了一个组，组名为 webservers ，改组有三台主机：192.168.120.61、192.168.120.62、192.168.120.63 。

第五、六行针对 webservers 组使用了 inventory 内置变量定义了 ssh 登录密码。

第七、八行定义一个组叫 ansible ，这个组包含 webservers 组。

```shell
# ansible webservers -m ping -o
192.168.120.62 | SUCCESS => {"changed": false, "ping": "pong"}
192.168.120.63 | SUCCESS => {"changed": false, "ping": "pong"}
192.168.120.61 | SUCCESS => {"changed": false, "ping": "pong"}
# ansible ansible -m ping -o          
192.168.120.63 | SUCCESS => {"changed": false, "ping": "pong"}
192.168.120.61 | SUCCESS => {"changed": false, "ping": "pong"}
192.168.120.62 | SUCCESS => {"changed": false, "ping": "pong"}
```

#### 多个 inventory 列表

使用多个 inventory 文件首先在 `/etc/ansible/hosts` 修改inventory 的值，如下：

```shell
vim /etc/ansible/ansible.cfg
...
inventory       = /root/inventory/
...
```

通过上面指定的目录定义如下的 inventory 文件：

```shell
# tree /root/inventory/
/root/inventory/
├── hosts
└── webservers
# cat /root/inventory/hosts 
192.168.120.62 ansible_ssh_pass='redhat'
192.168.120.63 ansible_ssh_pass='redhat'
# cat /root/inventory/webservers 
[webservers]
192.168.120.6[1:3]
[webservers:vars]
ansible_ssh_pass='redhat'
[ansible:children]
webservers
```

通过 `--list-hosts` 参数验证上面配置的 inventory 是否有问题：

```shell
# ansible 192.168.120.62:192.168.120.63 --list-hosts
  hosts (2):
    192.168.120.62
    192.168.120.63
# ansible webservers --list-hosts
  hosts (3):
    192.168.120.61
    192.168.120.62
    192.168.120.63
# ansible ansible --list-hosts
  hosts (3):
    192.168.120.61
    192.168.120.62
    192.168.120.63    
```

#### inventory 的内置参数

| 参数                         | 解释                           |
| ---------------------------- | ------------------------------ |
| ansible_ssh_host             | 定义 host ssh 地址             |
| ansible_ssh_port             | 定义 host ssh 端口             |
| ansible_ssh_user             | 定义 host ssh 认证用户         |
| ansible_ssh_pass             | 定义 host ssh 认证密码         |
| ansible_sudo                 | 定义 host sudo 用户            |
| ansible_sudo_pass            | 定义 host sudo 密码            |
| ansible_sudo_exe             | 定义 host sudo 路径            |
| ansible_connection           | 定义 host 连接方式             |
| ansible_ssh_private_key_file | 定义 host  私钥                |
| ansible_shell_type           | 定义 host shell 类型           |
| ansible_python_interpreter   | 定义 host 任务执行 python 路径 |
| ansible\_*\_interpreter      | 定义 host 其他语言解析器路径   |

### Ansible Ad-Hoc 

日常使用ansible时，我们一般会通过ansible模块使用相关功能，而每个模块的使用帮助可以通过 `ansible-doc`命令 + 模块名进行获取。而通过 `ansible-doc -l` 可以看到目前支持的所有模块帮助。下面是介绍几个 Ad-Hoc 命令。

#### 执行命令

```shell
# ansible webservers -m shell -a 'hostname' -f 5 -o
192.168.120.63 | CHANGED | rc=0 | (stdout) C6-node3
192.168.120.62 | CHANGED | rc=0 | (stdout) C6-node2
192.168.120.61 | CHANGED | rc=0 | (stdout) C6-node1
```

下面使用异步执行功能，为了异步启动一个任务，指定其async最大超时时间以及轮询其状态的频率。如果你没有为 poll 指定值,那么默认的轮询频率是`15`秒钟。pool设置为0时，任务会立即返回，而不等待命令执行的结果，继续执行下面的任务。

参数说明
`-B 3600` ：启用异步，超时时间3600
`-P 0` ：轮询时间为0

```shell
# ansible webservers -B 120 -P 0 -m shell -a 'sleep 10;hostname' -f 5 -o
192.168.120.63 | CHANGED | rc=-1 | (stdout) 
192.168.120.62 | CHANGED | rc=-1 | (stdout) 
192.168.120.61 | CHANGED | rc=-1 | (stdout) 
# cat .ansible_async/191389062778.6516
{"changed": true, "end": "2019-10-31 14:34:02.827643", "stdout": "C6-node1", "cmd": "sleep 10;hostname", "start": "2019-10-31 14:33:52.821549", "delta": "0:00:10.006094", "stderr": "", "rc": 0, "invocation": {"module_args": {"creates": null, "executable": null, "_uses_shell": true, "_raw_params": "sleep 10;hostname", "removes": null, "argv": null, "warn": true, "chdir": null, "stdin": null}}}
# ansible  192.168.120.61 -m async_status -a 'jid=191389062778.6516'
192.168.120.61 | SUCCESS => {
    "ansible_job_id": "191389062778.6516", 
    "changed": true, 
    "cmd": "sleep 10;hostname", 
    "delta": "0:00:10.006094", 
    "end": "2019-10-31 14:34:02.827643", 
    "finished": 1, 
    "rc": 0, 
    "start": "2019-10-31 14:33:52.821549", 
    "stderr": "", 
    "stderr_lines": [], 
    "stdout": "C6-node1", 
    "stdout_lines": [
        "C6-node1"
    ]
}
```

上面是获取的 192.168.120.61 主机执行后的结果，如果想获取 192.168.120.62 和 192.168.120.63 这两台机器的结果需要在该机器上找到 jid 进行查询。

#### 执行shell

以shell解释器执行脚本

```
ansible webservers -m shell -a "ifconfig eth0|grep addr"
```

以raw模块执行脚本

```shell
ansible webservers -m raw -a "ifconfig eth0|grep addr"
```

将本地脚本传送到远程节点上运行

```shell
ansible webservers -m script -a ip.sh
```

#### 复制文件

拷贝本地的/etc/hosts 文件到web组所有主机的/tmp/hosts（空目录除外）

```shell
ansible webservers -m copy -a "src=/etc/hosts dest=/tmp/hosts"
```

拷贝本地的ntp文件到目的地址，设置其用户及权限，如果目标地址存在相同的文件，则备份源文件。

```shell
ansible webservers -m copy -a "src=/etc/ntp.conf dest=/tmp/ntp.conf owner=root group=root mode=644 backup=yes force=yes"
```

file 模块允许更改文件的用户及权限

```shell
ansible webservers -m file -a "dest=/tmp/a.txt mode=600 owner=user group=user"
```

使用file 模块创建目录，类似mkdir -p

```shell
ansible webservers -m file -a "dest=/tmp/test mode=755 owner=user group=user state=directory"
```

使用file 模块删除文件或者目录

```shell
ansible webservers -m file -a "dest=/tmp/ntp.conf state=absent"
```

创建软连接，并设置所属用户和用户组

```shell
ansible webservers -m file -a "src=/file/to/link/to dest=/path/to/symlink owner=user group=user state=link"
```

touch 一个文件并添加用户读写权限，用户组去除写执行权限，其他组减去读写执行权限

```shell
ansible webservers -m file -a "path=/etc/foo.conf state=touch mode='u+rw,g-wx,o-rwx'"
```

#### 管理软件包

更新仓库缓存，并安装"foo"

```shell
ansible webservers -m apt -a "name=foo update_cache=yes"
```

删除 "foo"

```shell
ansible webservers -m apt -a "name=foo state=absent"
```

安装 "foo"

```shell
ansible webservers -m apt -a "name=foo state=present"
```

安装 1.0版本的 "foo"

```shell
ansible webservers -m apt -a "name=foo=1.00 state=present"
```

安装最新得"foo"

```shell
ansible webservers -m apt -a "name=foo state=latest"
```

从testing 仓库中安装最后一个版本的apache

```shell
ansible webservers -m yum -a "name=httpd enablerepo=testing state=present"
```

更新所有的包

```shell
ansible webservers -m yum -a "name=* state=latest"
```

安装远程的rpm包

```shell
ansible webservers -m yum -a "name=http://nginx.org/packages/centos/6/noarch/RPMS/nginx-release-centos-6-0.el6.ngx.noarch.rpm state=present"
```

安装 'Development tools' 包组

```shell
ansible webservers -m yum -a "name='@Development tools' state=present"
```

#### 用户和用户组

添加用户 'user'并设置其 uid 和主要的组'admin'

 ```shell
ansible webservers -m user -a "name=user comment='I am user ' uid=1040 group=admin"
 ```

添加用户 'user'并设置其登陆shell，并将其假如admins和developers组

```shell
ansible webservers -m user -a "name=user shell=/bin/bash groups=admins,developers append=yes"
```

删除用户 'user '

```shell
ansible webservers -m user -a "name=user state=absent remove=yes"
```

创建 user用户得   2048-bit SSH key，并存放在 ~user/.ssh/id_rsa

```shell
ansible webservers -m user -a "name=user generate_ssh_key=yes ssh_key_bits=2048 ssh_key_file=.ssh/id_rsa"
```

设置用户过期日期

```shell
ansible webservers -m user -a "name=user shell=/bin/zsh groups=nobdy expires=1422403387"
```

创建test组，并设置git为1000

```shell
ansible webservers -m group -a "name=test gid=1000 state=present"
```

删除test组

```shell
ansible webservers -m group -a "name=test state=absent"
```

创建monitor用户并指定密码

```shell
echo redhat|openssl passwd -1 -stdin
$1$WXMdXtJL$jQ1MyMhbAzCYkwynNyh4c/
ansible webservers -m user -a 'name=monitor password="$1$WXMdXtJL$jQ1MyMhbAzCYkwynNyh4c/"' -o
```

#### 服务管理

确保web组所有主机的httpd 是启动的

```shell
ansible webservers -m service -a "name=httpd state=started"
```

重启web组所有主机的httpd 服务

```shell
ansible webservers -m service -a "name=httpd state=restarted"
```

确保web组所有主机的httpd 是关闭的

```shell
ansible webservers -m service -a "name=httpd state=stopped"
```

#### 后台运行

长时间运行的操作可以放到后台执行，ansible 会检查任务的状态；在主机上执行的同一个任务会分配同一个job ID
后台执行命令3600s，-B 表示后台执行的时间

```shel
ansible all -B 3600 -a "/usr/bin/long_running_operation --do-stuff"
```

检查任务状态

```shell
ansible all -m async_status -a "jid=123456789"
```

后台执行命令最大时间是1800s 即30 分钟，-P 每60s 检查下状态默认15s

```shell
ansible all -B 1800 -P 60 -a "/usr/bin/long_running_operation --do-stuff"
```

#### 定时任务

每天5点，2点得时候执行 ls -alh > /dev/null

```shell
ansible webservers -m cron -a "name='check dirs' minute='0' hour='5,2' job='ls -alh > /dev/null'"
```

#### 搜集系统信息

搜集主机的所有系统信息

```shell
ansible all -m setup
```

搜集系统信息并以主机名为文件名分别保存在/tmp/facts 目录

```shell
ansible all -m setup --tree /tmp/facts
```

搜集和内存相关的信息

```shell
ansible all -m setup -a 'filter=ansible_*_mb'
```

搜集网卡信息

```shell
ansible all -m setup -a 'filter=ansible_eth[0-2]'
```

### Ansible playbook

上面使用命令行的方式能解决一些简单的问题，如果处理一些复杂的环境的配置管理工作，那么就需要使用playbook 组件了。实际使用中最好是使用playbook进行操作，这样维护起来会很方面，具体使用在后面专门做介绍。

### Ansible facts

facts 组件是 Ansible 用于采集被管理机器设备信息的一个功能，可以通过setup 模块查询机器的所有facts 信息，然后使用filter 来查询指定的信息。输出的facts 信息被包装在一个 JSON 格式的数据结构中。

#### 常用变量

* ansible_distribution

* ansible_distribution_release

* ansible_distribution_version

* ansible_fqdn

* ansible_hostname

* ansible_os_family

* ansible_pkg_mgr

* ansible_default_ipv4.address

* ansible_default_ipv6.address

#### 自定义目标系统facts

在远程主机/etc/ansible/facts.d/目录下创建.fact 结尾的文件，也可以是json、ini 或者返回json 格式数据的可执行文件，这些将被作为远程主机本地的facts 执行

```shell
# cat /etc/ansible/facts.d/hello.fact 
[test]
h=hello
w=world
```

可以通过`{{ ansible_local.preferences.test.h }}`方式来使用该变量

```shell
# ansible 192.168.120.62 -m setup -a "filter=ansible_local"
192.168.120.62 | SUCCESS => {
    "ansible_facts": {
        "ansible_local": {
            "hello": {
                "test": {
                    "h": "hello", 
                    "w": "world"
                }
            }
        }
    }, 
    "changed": false
}
```

#### Facts 使用文件作为缓存

修改ansible配置文件

```shell
# vim /etc/ansible/ansible.cfg
fact_caching = jsonfile
fact_caching_connection = /tmp/facts_cache
# mkdir /tmp/facts_cache
# ansible 192.168.120.62 -m setup -a "filter=ansible_local"
# cat /tmp/facts_cache/192.168.120.62
{
    "ansible_local": {
        "hello": {
            "test": {
                "h": "hello", 
                "w": "world"
            }
        }
    }
}
```

#### Facts 使用redis作为缓存

安装redis

```shell
# pip install redis
```

修改ansible配置文件

```shell
# vim  /etc/ansible/ansible.cfg
gathering = smart
fact_caching = redis
fact_caching_timeout = 86400
fact_caching_connection = 192.168.120.61:6379:0
```

使用 CentOS 6.7 系统 Python 2.6.6 和 ansible 2.6.17。使用上面安装配置失败，可以尝试升级版本进行测试。

### Ansible role

当一个配置管理的任务十分复杂时，playbook文件会十分庞大，这这种情况下将不利于扩展和复用。 这个时候可以使用ansible role将这个复杂的playbook模块化。Ansible role实际上是对playbook进行了逻辑上的划分，分成不同目录。

#### 项目结构

```shell
site.yml
webservers.yml
fooservers.yml
roles/
   common/
     files/
     templates/
     tasks/
     handlers/
     vars/
     defaults/
     meta/
   webservers/
     files/
     templates/
     tasks/
     handlers/
     vars/
     defaults/
     meta/
```

一个 playbook 如下：

```shell
---
- hosts: webservers
  roles:
     - common
     - webservers
```

这个 playbook 为一个角色 ‘x’ 指定了如下的行为：

- 如果 roles/x/tasks/main.yml 存在, 其中列出的 tasks 将被添加到 play 中
- 如果 roles/x/handlers/main.yml 存在, 其中列出的 handlers 将被添加到 play 中
- 如果 roles/x/vars/main.yml 存在, 其中列出的 variables 将被添加到 play 中
- 如果 roles/x/meta/main.yml 存在, 其中列出的 “角色依赖” 将被添加到 roles 列表中 (1.3 and later)
- 所有 copy tasks 可以引用 roles/x/files/ 中的文件，不需要指明文件的路径。
- 所有 script tasks 可以引用 roles/x/files/ 中的脚本，不需要指明文件的路径。
- 所有 template tasks 可以引用 roles/x/templates/ 中的文件，不需要指明文件的路径。
- 所有 include tasks 可以引用 roles/x/tasks/ 中的文件，不需要指明文件的路径。

### Ansible Galaxy

[Ansible Galaxy](http://galaxy.ansible.com/) 是一个自由分享 roles 的网站，网站提供所有类型的由社区开发的 roles，这对于实现你的自动化项目是一个很好的参考。网站提供这些 roles 的排名、查找以及下载。













