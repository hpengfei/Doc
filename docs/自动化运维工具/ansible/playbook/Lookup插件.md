### file：获取文件内容

```
---
- hosts: all
  vars:
     contents: "{{ lookup('file', '/etc/foo.txt') }}"
tasks:
- debug: msg="the value of foo.txt is {{ contents }}"
```

### password：生成密码字符串

```
---
- hosts: all
tasks:
# 使用只有ascii字母且长度为8的随机密码创建一个mysql用户:
    - mysql_user: name={{ client }}
                  password="{{ lookup('password', '/tmp/passwordfile chars=ascii_letters length=8') }}"
                  priv={{ client }}_{{ tier }}_{{ role }}.*:ALL
# 使用只有数字的随机密码创建一个mysql用户:
    - mysql_user: name={{ client }}
                  password="{{ lookup('password', '/tmp/passwordfile chars=digits') }}"
                  priv={{ client }}_{{ tier }}_{{ role }}.*:ALL
# 使用许多不同的字符集使用随机密码创建一个mysql用户：
    - mysql_user: name={{ client }}
                  password="{{ lookup('password', '/tmp/passwordfile chars=ascii_letters,digits,hexdigits,punctuation') }}"
                  priv={{ client }}_{{ tier }}_{{ role }}.*:ALL
```

### 读取csv文件

```
f.csv 
Symbol,Atomic Number,Atomic Mass
H,1,1.008
He,2,4.0026
Li,3,6.94
Be,4,9.012
B,5,10.81
```

```
- debug: msg="The atomic number of Lithium is {{ lookup('csvfile', 'Li file=elements.csv delimiter=,') }}"
- debug: msg="The atomic mass of Lithium is {{ lookup('csvfile', 'Li file=elements.csv delimiter=, col=2') }}"
```



