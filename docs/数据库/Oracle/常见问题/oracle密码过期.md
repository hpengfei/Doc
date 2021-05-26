### 问题

**使用sqlplus登陆oracle数据库时提示“ORA-28002: 7 天之后口令将过期” 或提示 密码过期。**

### 问题原因

Oracle数据库默认情况下用户口令有效期为180天， 如果超过180天用户密码未做修改则该用户无法登录。

### 解决步骤

1. 查看用户的proifle是哪个，一般是default：

```sql
SQL> SELECT username,PROFILE FROM dba_users;
```

2. 查看指定概要文件(如default)的密码有效期设置：

```sql
SQL> SELECT * FROM dba_profiles s WHERE s.profile='DEFAULT' AND resource_name='PASSWORD_LIFE_TIME';

PROFILE 		       RESOURCE_NAME			RESOURCE
------------------------------ -------------------------------- --------
LIMIT
----------------------------------------
DEFAULT 		       PASSWORD_LIFE_TIME		PASSWORD
180
```

3. 密码有效期由默认的180天修改成“无限制”

```sql
SQL> ALTER PROFILE DEFAULT LIMIT PASSWORD_LIFE_TIME UNLIMITED;  

Profile altered.
```

修改之后不需要重启动数据库，会立即生效。

4. 修改登录时间限制后继续显示ORA-28002警告，则需要将帐户必须再改一次密码

```sql
SQL> alter user username identified by password account unlock; 

User altered.
```

上面的密码可以继续使用原先的密码，而不用换新密码。

