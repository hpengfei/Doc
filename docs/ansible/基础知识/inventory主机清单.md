## inventory 行为参数

### 主机连接

| 参数               | 说明                                                         |
| :----------------- | :----------------------------------------------------------- |
| ansible_connection | 与主机的连接类型.比如:local, ssh 或者 paramiko. Ansible 1.2 以前默认使用 paramiko.1.2 以后默认使用 ‘smart’,’smart’ 方式会根据是否支持 ControlPersist, 来判断’ssh’ 方式是否可行. |

### ssh 连接参数

| 参数                         | 说明                                                         |
| :--------------------------- | :----------------------------------------------------------- |
| ansible_ssh_host             | 将要连接的远程主机名.与你想要设定的主机的别名不同的话,可通过此变量设置. |
| ansible_ssh_port             | ssh端口号.如果不是默认的端口号,通过此变量设置.               |
| ansible_ssh_user             | 默认的 ssh 用户名                                            |
| ansible_ssh_pass             | ssh 密码(这种方式并不安全,我们强烈建议使用 –ask-pass 或 SSH 密钥) |
| ansible_ssh_private_key_file | ssh 使用的私钥文件.适用于有多个密钥,而你不想使用 SSH 代理的情况. |
| ansible_ssh_common_args      | 此设置附加到sftp，scp和ssh的缺省命令行                       |
| ansible_sftp_extra_args      | 此设置附加到默认sftp命令行。                                 |
| ansible_scp_extra_args       | 此设置附加到默认scp命令行。                                  |
| ansible_ssh_extra_args       | 此设置附加到默认ssh命令行。                                  |
| ansible_ssh_pipelining       | 确定是否使用SSH管道。 这可以覆盖ansible.cfg中得设置。        |

### 远程主机环境参数

| 参数                       | 说明                                                         |
| :------------------------- | :----------------------------------------------------------- |
| ansible_shell_type         | 目标系统的shell类型.默认情况下,命令的执行使用 ‘sh’ 语法,可设置为 ‘csh’ 或 ‘fish’. |
| ansible_python_interpreter | 目标主机的 python2 路径.适用于的情况: 系统中有多个 Python 版本, 或者命令路径不是”/usr/bin/python”,比如 *BSD, 或者 /usr/bin/python |
| ansible\_*\_interpreter    | 这里的”*“可以是ruby 或perl 或其他语言的解释器，作用和ansible_python_interpreter 类似 |
| ansible_shell_executable   | 这将设置ansible控制器将在目标机器上使用的shell，覆盖ansible.cfg中的配置，默认为/bin/sh。 |

## inventory 清单示例

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

