### 创建逻辑卷

```
pvcreate /dev/vdb
vgcreate vg00 /dev/vdb
lvcreate -l 100%free vg00 -n d
```

### 格式化和挂载

```
mkfs.xfs /dev/mapper/vg00-d
mkdir /data
mount /dev/mapper/vg00-d /data
echo "/dev/mapper/vg00-d     /data      xfs     defaults,noatime,nobarrier        1       2">>/etc/fstab
```

### 扩展逻辑卷

```
pvcreate /dev/xxx
vgextend vg00 /dev/xxx
lvextend -L +扩展量 lv完整名 
resize2fs  lv完整名
```

### 删除逻辑卷

```
1.卸载：umount 
2.删lv：lvremove lv完整路径 
3.删vg：vgremove vg名 
4.删PV：pvremove 设备完整路径 去硬盘
```

