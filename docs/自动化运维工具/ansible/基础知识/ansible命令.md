## ansible

### 选项参数

| 参数                                     | 说明                                                         |
| :--------------------------------------- | :----------------------------------------------------------- |
| -a MODULE_ARGS, –args=MODULE_ARGS        | 模块的参数。                                                 |
| -B SECONDS, –background=SECONDS          | 异步运行时，多长时间超时。                                   |
| -C, –check                               | 只是测试一下会改变什么内容，不会真正去执行;相反,试图预测一些可能发生的变化。 |
| -D, –diff                                | 当更改文件和模板时，显示这些文件得差异，比–check效果好。     |
| -e EXTRA_VARS, –extra-vars=EXTRA_VARS    | 添加附加变量，比如key=value，yaml，json格式。                |
| -f FORKS, –forks=FORKS                   | 指定定要使用的并行进程数，默认为5个。                        |
| -i INVENTORY, –inventory-file=INVENTORY  | 指定主机清单文件或逗号分隔的主机，默认为/etc/ansible/hosts。 |
| -l SUBSET, –limit=SUBSET                 | 进一步限制所选主机/组模式，只执行-l 后的主机和组。           |
| –list-hosts                              | 输出匹配主机的列表。                                         |
| -m MODULE_NAME, –module-name=MODULE_NAME | 要执行的模块，默认为command。                                |
| -M MODULE_PATH, –module-path=MODULE_PATH | 要执行的模块的路径。                                         |
| -o, –one-line                            | 压缩输出，摘要输出.尝试一切都在一行上输出。                  |
| –output=OUTPUT_FILE                      | 加密或解密输出文件名 用于标准输出。                          |
| -P POLL_INTERVAL, –poll=POLL_INTERVAL    | 如果使用-B，则设置轮询间隔。                                 |
| –syntax-check                            | 对playbook进行语法检查，且不执行playbook。                   |
| -t TREE, –tree=TREE                      | 将日志内容保存在该目录中,文件名以执行主机名命名。            |

### 特权升级选项

| 参数                               | 说明                                                         |
| :--------------------------------- | :----------------------------------------------------------- |
| -s, –sudo                          | 使用sudo (nopasswd)运行操作 ， 不推荐使用                    |
| -U SUDO_USER, –sudo-user=SUDO_USER | sudo 用户，默认为root， 不推荐使用                           |
| -S, –su                            | 使用su运行操作， 不推荐使用                                  |
| -R SU_USER, –su-user=SU_USER       | su 用户，默认为root，不推荐使用                              |
| -b, –become                        | 运行操作                                                     |
| –become-method=BECOME_METHOD       | 权限升级方法使用 ，默认为sudo，有效选择：sudo,su,pbrun,pfexec,runas,doas,dzdo |
| –become-user=BECOME_USER           | 使用哪个用户运行，默认为root                                 |
| –ask-sudo-pass                     | sudo密码，不推荐使用                                         |
| –ask-su-pass                       | su密码，不推荐使用                                           |
| -K, –ask-become-pass               | 权限提升密码                                                 |

### 连接选项

| 参数                                                      | 说明                                                         |
| :-------------------------------------------------------- | :----------------------------------------------------------- |
| -k, –ask-pass                                             | 要求用户输入请求连接密码                                     |
| –private-key=PRIVATE_KEY_FILE, –key-file=PRIVATE_KEY_FILE | 私钥路径，使用这个文件来验证连接                             |
| -u REMOTE_USER, –user=REMOTE_USER                         | 连接用户                                                     |
| -c CONNECTION, –connection=CONNECTION                     | 连接类型，默认smart                                          |
| -T TIMEOUT, –timeout=TIMEOUT                              | 指定默认超时时间，默认是10S                                  |
| –ssh-common-args=SSH_COMMON_ARGS                          | 指定要传递给sftp / scp / ssh的常见参数 （例如 ProxyCommand） |
| –sftp-extra-args=SFTP_EXTRA_ARGS                          | 指定要传递给sftp，例如-f -l                                  |
| –scp-extra-args=SCP_EXTRA_ARGS                            | 指定要传递给scp，例如 -l                                     |
| –ssh-extra-args=SSH_EXTRA_ARGS                            | 指定要传递给ssh，例如 -R                                     |

## ansible-doc

### 选项参数

| 参数                                     | 说明                       |
| :--------------------------------------- | :------------------------- |
| -l, –list                                | 列出可用的模块             |
| -M MODULE_PATH, –module-path=MODULE_PATH | 指定到模块库的路径         |
| -s, –snippet                             | 显示playbook制定模块的用法 |

## ansible-playbook

### 选项参数

 相对于ansible，增加了下列选项： 

| 参数                         | 说明                             |
| :--------------------------- | :------------------------------- |
| –flush-cache                 | 清除fact缓存                     |
| –force-handlers              | 如果任务失败，也要运行handlers   |
| –list-tags                   | 列出所有可用的标签               |
| –list-tasks                  | 列出将要执行的所有任务           |
| –skip-tags=SKIP_TAGS         | 跳过运行标记此标签的任务         |
| –start-at-task=START_AT_TASK | 在此任务处开始运行               |
| –step                        | 一步一步：在运行之前确认每个任务 |
| -t TAGS, –tags=TAGS          | 只运行标记此标签的任务           |

[引用](https://lework.github.io/2016/11/19/Ansible-xiao-shou-ce-xi-lie-san-(-ming-ling-jie-shao-)/)

